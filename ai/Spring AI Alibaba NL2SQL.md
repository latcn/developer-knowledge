> **适用框架**：Spring AI Alibaba 1.1.3.0 / Spring AI 1.1.3  
> **核心聚焦**：NL2SQL 架构原理、生产级实践、pgvector 向量数据库集成  
> **文档性质**：生产可落地完整指南，含完整示例代码与排查表

---
## 前言：用第一性原理理解 NL2SQL

**核心问题**：数据存储在关系型数据库中，而人用自然语言思考和表达。这两者之间存在“语言鸿沟”。  
**本质**：NL2SQL = **Schema 召回（RAG） + SQL 生成（LLM）**

Spring AI Alibaba NL2SQL 是一个基于 **Graph 编排** 的 RAG 智能体框架。不同于简单的 LLM 调用（一次翻译即结束），Graph 编排允许在 SQL 生成失败或验证不通过时自动回溯、重试，形成闭环反馈系统，提升整体准确率。它通过向量检索召回相关表结构，再交给大模型生成 SQL。

---
## 第一篇：架构设计与核心算法

### 1.1 整体架构

Spring AI Alibaba NL2SQL 采用 **StateGraph 编排** 的分布式智能体工作流，支持状态管理和错误重试。

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              nl2sqlGraph 完整执行流程                             │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  START → QueryRewriteNode → KeywordExtractNode → SchemaRecallNode               │
│                │                    │                  │                         │
│                │ (问题分类+重写)       │ (关键词+证据召回)  │ (向量检索表/列)         │
│                ▼                    ▼                  ▼                         │
│            condition          EVIDENCES        TABLE_DOCUMENTS_FOR_SCHEMA_OUTPUT │
│                                                                                  │
│  ←──────────────────────────────────────────────────────────────────────────→    │
│                              (跨节点 State 共享)                                  │
│                                                                                  │
│  TableRelationNode → PlannerNode → PlanExecutorNode → SqlExecuteNode            │
│         │                │               │                │                      │
│         │ (外键扩展+      │ (大模型生成     │ (步骤检查+       │ (执行SQL)           │
│         │  表筛选)         │   计划)         │  路由)          │                     │
│         ▼                ▼               ▼                ▼                      │
│      外键扩展          SQL计划        分步调度            │                       │
│                                                        │                         │
│                                        ┌───────────────┘                         │
│                                        ▼                                         │
│                               SemanticConsistencyNode                            │
│                              (验证SQL是否能满足用户需求)                            │
│                                        │                                         │
│                    ┌───────────────────┼───────────────────┐                     │
│                    ▼                   ▼                   ▼                     │
│               SqlGenerateNode      PlanExecutorNode       END                    │
│            (重试最多3轮，每轮注入失败信息) (继续或结束)                              │
│                    │                                                              │
│                    └──────────────────→ SqlExecuteNode (再次执行)                 │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**分支触发条件与差异化重试机制**：

| 失败场景 | 路由目标 | 重试上限 | 重试策略 |
|----------|----------|----------|----------|
| `SqlExecuteNode` 执行失败 | 回退至 `SqlGenerateNode` 重新生成 SQL | 最多 **3 轮** | 每轮将失败原因（错误日志）注入提示词 |
| `SemanticConsistencyNode` 语义校验不通过 | 回退至 `SqlGenerateNode` 重新生成 | 最多 **3 轮** | 每轮将语义冲突原因注入提示词 |
| 超出最大重试次数 | 直接跳转到 END 节点 | - | - |

**中断恢复机制**：`CompiledGraph` 使用 `interruptBefore(NODE_NAME)` 在特定节点前挂起执行，中断时 Graph 会保存当前完整状态快照，返回中断信息及 `threadId`。收到外部反馈后，使用 `compiledGraph.resume(threadId, feedback)` 方法恢复 Graph 继续执行，其中 `threadId` 通过 `java.util.UUID.randomUUID().toString()` 生成，需与中断时保存的 ID 保持一致。

> **超时兜底策略**：对于人工审核节点，建议在应用层设置超时时间（如 30 分钟），超时后自动调用 `resume(threadId, buildRejectFeedback())` 驳回请求，避免流程永久阻塞。

### 1.2 核心节点职责

| 节点 | 核心职责 | 关键技术 | 输出 Key |
|------|----------|----------|----------|
| **QueryRewriteNode** | 问题分类 + 重写（替换代词、补充上下文）；仅数据分析类问题可进入后续节点 | 大模型 + 业务知识召回 | `QUERY_REWRITE_NODE_OUTPUT` |
| **KeywordExtractNode** | 提取关键词 + 扩写 + **Evidence 自动召回**（通过 LLM 提取关键词后，将关键词向量化并检索向量库中的 Evidence 文档） | NER + 业务知识向量检索 | `KEYWORD_EXTRACT_NODE_OUTPUT`、`EVIDENCES` |
| **SchemaRecallNode** | 从向量库检索表数据 + 列数据 | 向量嵌入 + 相似度检索 | `TABLE_DOCUMENTS_FOR_SCHEMA_OUTPUT` |
| **TableRelationNode** | 外键扩展 + 大模型筛选最终使用的表 | 外键元数据 + 图推理 | `TABLE_RELATION_OUTPUT` |
| **PlannerNode** | 使用大模型生成执行计划（plan.steps） | 大模型推理 | `PLANNER_NODE_OUTPUT` |
| **PlanExecutorNode** | 校验计划有效性；分步调度执行 | 状态机 + 步骤检查 | `PLAN_EXECUTOR_NODE_OUTPUT` |
| **SqlExecuteNode** | 执行 SQL，失败则触发重试流程 | SQL 执行器 | `SQL_EXECUTE_NODE_OUTPUT` |
| **SemanticConsistencyNode** | 校验 SQL 是否能满足原始问题语义；失败则重试 | 大模型一致性检查 | `SEMANTIC_CONSISTENCY_NODE_OUTPUT` |
| **SqlGenerateNode** | **重试专用**：将失败信息（错误日志/语义冲突）加入提示词，使用大模型重新生成 3 轮 SQL，每轮评分（0~1），取最高分 | 大模型 + 评分器 | `SQL_GENERATE_NODE_OUTPUT` |

> **Evidence 召回位置**：`KeywordExtractNode` 内部通过 LLM 提取关键词后，将关键词向量化并检索向量库中的 Evidence 文档，非 `SchemaRecallNode`。本文档所有描述均已统一。

### 1.3 第一性原则设计

- **为什么需要 Graph？** 因为 NL2SQL 不是一次翻译，而是多步骤决策。Graph 支持回溯和重试，形成闭环反馈——`SqlExecuteNode` 执行失败时回退到 `SqlGenerateNode`，`SemanticConsistencyNode` 校验不通过时也会触发重试。此外，`interruptBefore` 机制支持在人工审核节点前中断，等待外部反馈后恢复执行。
- **为什么需要 Schema 召回？** 全量表结构 Token 巨大，且会干扰模型推理，必须用 RAG 动态获取最相关部分。
- **为什么需要 Evidence？** 业务术语、规则、JOIN 模式无法从表结构自动推导，必须显式注入知识库，由 `KeywordExtractNode` 自动召回。

---
## 第二篇：主要功能与能力矩阵

### 2.1 基础能力

- 自然语言 → SQL（支持 MySQL、PostgreSQL、H2 等）
- SQL 自动执行并返回结构化结果
- 多表关联与复杂查询（窗口函数、子查询、聚合）

### 2.2 进阶能力

| 能力 | 说明 | 技术实现 |
|------|------|----------|
| **Evidence 参考证据** | 配置业务术语、规则、JOIN 模式，`KeywordExtractNode` **自动从向量库召回** | 向量存储 + 检索召回 |
| **Semantic View 语义视图** | 将业务语义与物理模型解耦，固化标准查询 | 预定义视图 + Prompt 注入 |
| **多轮差异化重试机制** | SQL 生成失败或语义校验不通过时自动重试，最多 3 轮，每轮注入不同的失败上下文，取评分最高结果 | `SqlGenerateNode` 内置重试循环 + 评分器 |
| **Python 代码生成与执行** | 对 SQL 结果集进行二次数据分析，**支持资源隔离（独立进程/容器）** | Python 引擎集成 + JVM 隔离 |
| **报告生成** | Markdown/HTML 格式输出分析结果 | 报告模板引擎 + LLM 改写 |
| **Human-in-the-Loop** | 人工审核高风险 SQL，支持同意/修改/驳回，带超时自动驳回 | `interruptBefore` + `resume` + 超时定时器 |
| **Multi-Agent** | 多智能体协作，不同 Agent 拥有独立数据源和业务知识 | DataAgent Graph & Agent Framework |

### 2.3 四种使用方式

| 方式 | 适用场景 | 能力范围 |
|------|----------|----------|
| `BaseNl2SqlService.nl2sql()` | 简单查询，不涉及 Graph | 仅 SQL 生成，无分类/重写/重试 |
| `Nl2SqlService.nl2sql(String query)` | 标准场景，使用预构建 nl2sqlGraph | 含问题分类、重写、Schema 召回、SQL 生成、执行、语义校验、重试 |
| `Nl2SqlService.nl2sql(String query, String agentId)` | 多智能体场景 | 不同 Agent 有独立数据源、Evidence 和 Prompt |
| `CompiledGraph`（`@Qualifier("nl2sqlGraph")`） | 复杂场景，需自定义节点或监听器 | 解锁完整 Graph 编排能力，**可按需禁用特定节点**（需自定义图） |

> **节点开关**：框架默认 Graph 包含所有节点。如需禁用某些节点（如不需要查询重写），可自行构建 Graph 并排除对应节点，或通过条件边动态跳过。社区已有相关实践方案。

### 2.4 运行时配置能力

- **动态 Prompt**：通过 Nacos 2.X（使用 `configService`）或 3.X（使用 OpenAPI，需 `spring-cloud-starter-alibaba-nacos-config` 版本 ≥2023.0.1.0）实现 Prompt 模板的动态管理，支持实时更新、统一管理、版本控制，无需重启应用即可调整提示词。
- **多数据源**：支持为不同 Agent 配置独立的数据库数据源，需显式指定 `@Qualifier` 区分。
- **Human-in-the-Loop**：运行时请求参数可开启人工审核，暂停执行等待用户确认，支持超时自动驳回。

---
## 第三篇：生产级完整示例（电商数据分析）

### 3.1 场景与架构

**业务**：电商运营人员用自然语言查询销售额、订单量、用户行为等。

**技术栈**：
- Spring Boot 3.5.8
- Spring AI Alibaba 1.1.3.0
- Spring AI 1.1.3（由 BOM 统一管理）
- pgvector 0.8.2（PostgreSQL 17）
- DashScope（通义千问 Qwen-Max）

### 3.2 数据库 Schema（带业务注释）

```sql
-- 用户表
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY COMMENT '用户唯一ID',
    user_name VARCHAR(50) NOT NULL COMMENT '用户姓名',
    register_date DATE NOT NULL COMMENT '注册日期',
    user_level VARCHAR(20) DEFAULT 'normal' COMMENT '等级：normal/silver/gold'
) COMMENT '用户信息表';

-- 订单表
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY COMMENT '订单ID',
    user_id BIGINT NOT NULL COMMENT '用户ID',
    total_amount DECIMAL(12,2) NOT NULL COMMENT '订单总金额（元）',
    actual_amount DECIMAL(12,2) NOT NULL COMMENT '实付金额',
    order_status VARCHAR(20) NOT NULL COMMENT '状态：pending/completed/cancelled',
    order_date DATE NOT NULL COMMENT '下单日期'
) COMMENT '订单主表';
```

### 3.3 Maven 依赖

```xml
<properties>
    <java.version>21</java.version>
    <spring-ai-alibaba.version>1.1.3.0</spring-ai-alibaba.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.alibaba.cloud.ai</groupId>
            <artifactId>spring-ai-alibaba-bom</artifactId>
            <version>${spring-ai-alibaba.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>com.alibaba.cloud.ai</groupId>
        <artifactId>spring-ai-alibaba-starter-nl2sql</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
</dependencies>
```

> **版本说明**：Spring AI Alibaba 1.1.3.0 是当前最新稳定版本，其 BOM 内部引入的 Spring AI 版本为 **1.1.3**。无需单独声明 `spring-ai.version`，完全由 BOM 管理。请通过 [GitHub Releases](https://github.com/alibaba/spring-ai-alibaba/releases) 关注最新版本发布。

### 3.4 配置文件（application.yml）

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/vectordb
    username: postgres
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 10
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}      # 推荐环境变量名，也支持 AI_DASHSCOPE_API_KEY
      chat:
        options:
          model: qwen-max
          temperature: 0.2
    vectorstore:
      pgvector:
        index-type: HNSW                 # 索引类型：NONE（默认）/ HNSW / IVFFlat
        distance-type: COSINE_DISTANCE
        dimensions: 1536
        initialize-schema: true

management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
```

> **配置格式说明**：Spring AI 1.1.x 配置属性**同时支持连字符格式（`index-type`）和驼峰格式（`indexType`）**，两种格式均可正常工作，无需刻意选择。
>
> **索引类型说明**：`index-type` 默认值为 `NONE`（精确最近邻搜索，适用于十万级以下小数据集），可选值 `HNSW`（高性能近似搜索，查询快但内存占用高）和 `IVFFlat`（适用于百万级静态数据集）。请根据数据规模和性能需求选择合适的索引类型。
>
> **维度限制说明**：`dimensions` 参数用于定义 embedding 向量的维度数（如 OpenAI `text-embedding-ada-002` 为 1536 维），可设为任意整数值。但需要注意：若使用 **HNSW 索引类型**（`index-type: HNSW`），`dimensions` **必须 ≤ 2000**（pgvector 0.7.0+ 官方限制），否则索引创建会失败。若需存储超过 2000 维的向量，请改用 `IVFFlat` 或 `NONE` 索引类型。
>
> **扩展依赖说明**：`initialize-schema: true` 会在启动时自动安装 PostgreSQL 所需的**三个扩展**：`vector`、`hstore`、`uuid-ossp`，并创建向量存储表 `vector_store`。

### 3.5 核心服务代码

**DTO 定义**：

```java
@Data
@Builder
public class QueryResult {
    private boolean success;
    private String message;
    private List<Map<String, Object>> data;
    private String generatedSql;

    public static QueryResult success(List<Map<String, Object>> data, String sql) {
        return QueryResult.builder().success(true).data(data).generatedSql(sql).build();
    }
    public static QueryResult error(String msg) {
        return QueryResult.builder().success(false).message(msg).build();
    }
}
```

**服务类**：

```java
@Service
@Slf4j
public class DataAnalysisService {
    private final Nl2SqlService nl2SqlService;
    private final JdbcTemplate jdbcTemplate;
    private final SqlSecurityValidator sqlValidator;

    public QueryResult executeQuery(String naturalQuery) {
        Nl2SqlRequest request = Nl2SqlRequest.builder().query(naturalQuery).build();
        Nl2SqlResponse response = nl2SqlService.nl2sql(request);
        String sql = response.getSql();
        if (!sqlValidator.isSafe(sql)) {
            return QueryResult.error("SQL 被安全策略拦截: " + sql);
        }
        try {
            jdbcTemplate.setQueryTimeout(30);
            List<Map<String, Object>> result = jdbcTemplate.queryForList(sql);
            return QueryResult.success(result, sql);
        } catch (Exception e) {
            return QueryResult.error("执行失败: " + e.getMessage());
        }
    }
}
```

**SQL 安全校验器**：

```java
@Component
public class SqlSecurityValidator {
    // 匹配 SELECT 开头，处理前导空白和单行注释
    private static final Pattern SELECT_PATTERN =
        Pattern.compile("^\\s*(?:--.*?\\n)?\\s*SELECT\\s+.*$", Pattern.CASE_INSENSITIVE | Pattern.DOTALL);
    private final Set<String> dangerousKeywords = new HashSet<>(Arrays.asList(
        "DROP", "TRUNCATE", "ALTER", "UPDATE", "DELETE", "INSERT", "CREATE", "REPLACE", "MERGE"
    ));

    public boolean isSafe(String sql) {
        if (sql == null) return false;
        if (!SELECT_PATTERN.matcher(sql.trim()).matches()) return false;
        String upper = sql.toUpperCase();
        return dangerousKeywords.stream().noneMatch(upper::contains);
    }
}
```

> **安全说明**：SQL 注入绕过方式多样（大小写混写、URL 编码、注释干扰），**正则无法从根本上防御 SQL 注入**。此校验器仅为辅助拦截手段，**核心安全防线是数据库只读账户**。生产环境建议使用完整 SQL 解析器（如 Druid SQL Parser、JSqlParser）进行严格校验。

### 3.6 Evidence 配置（业务知识注入）

Evidence 是 `KeywordExtractNode` 会自动从向量库召回的**业务知识库**，用于帮助大模型理解业务术语和规则。

**Evidence 召回机制说明**：`KeywordExtractNode` 在节点执行过程中，先通过 LLM 提取用户问题中的关键词，然后将关键词向量化，与向量库中的 Evidence 文档进行相似度检索，将检索到的业务规则、术语映射、JOIN 模式等信息一同传入后续节点，显著提升 SQL 生成的业务匹配准确率。

示例文件 `evidence-config.yml`：

```yaml
evidence:
  - type: TERM_MAPPING
    content: |
      "销售额" → total_amount
      "实付金额" → actual_amount
      "复购用户" → 下单次数 >= 2
  - type: BUSINESS_RULE
    content: |
      查询"最近30天" → order_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
      销量只统计订单状态为 'completed' 的记录
```

初始化代码（将 Evidence 存入向量库，供 `KeywordExtractNode` 自动召回）：

```java
@Component
public class EvidenceInitializer implements CommandLineRunner {
    @Autowired
    private VectorStore vectorStore;

    @Override
    public void run(String... args) {
        if ("true".equals(System.getProperty("init.evidence"))) {
            List<Document> docs = loadEvidenceDocuments();
            vectorStore.add(docs);
            log.info("已添加 {} 条 Evidence", docs.size());
        }
    }
    private List<Document> loadEvidenceDocuments() {
        Document doc = new Document(content);
        doc.getMetadata().put("type", "TERM_MAPPING");
        return Collections.singletonList(doc);
    }
}
```

> **Schema 变更后处理**：当数据库表结构发生变化（新增表、修改字段）时，需要重新调用 `vectorStore.schema()` 方法。该方法会**清空旧的 Schema 向量并全量重建**，有一定性能开销，建议在低峰期执行，并做好通知。方法是幂等的，重复调用安全但非增量。

### 3.7 向量库切换实践（开发 → 生产）

| 阶段 | 推荐方案 | 配置要点 | 说明 |
|------|----------|----------|------|
| **开发测试** | `SimpleVector`（内存向量库） | 无需额外配置 | 快速验证，重启后数据丢失 |
| **生产环境** | `pgvector` 或 `AnalyticDB` | 配置数据源和索引参数 | 持久化存储，支持高并发 |

**pgvector Docker Compose 完整配置**：

```yaml
version: '3.8'
services:
  postgres-pgvector:
    image: pgvector/pgvector:pg17
    container_name: pgvector-demo
    environment:
      POSTGRES_DB: vectordb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - pgvector_data:/var/lib/postgresql/data
volumes:
  pgvector_data:
```

> `initialize-schema: true` 会自动安装所需的三个 PostgreSQL 扩展：`vector`、`hstore`、`uuid-ossp`，无需手动干预。

**手动创建 HNSW 索引（可选）** ：

```sql
CREATE INDEX CONCURRENTLY vector_hnsw_idx
ON vector_store USING hnsw (embedding vector_cosine_ops)
WITH (m = 24, ef_construction = 200);
```

> **索引参数说明**：`m` 控制每个节点最大连接数（默认 16），`ef_construction` 控制构建时动态列表大小（默认 64）。增大两者可提升查询精度，但会增加构建时间和内存占用。对于百万级数据，建议 `m=32`，`ef_construction=200`；对于千万级数据，`m=48`，`ef_construction=400` 可获更好精度，但内存消耗会显著上升。

### 3.8 人工审核集成（Human-in-the-Loop）

生产环境中，对于包含 `UPDATE`、`DELETE` 等写操作或预估影响行数超过阈值的 SQL，应引入人工审批机制。

**三层安全防线**：
1. **数据库层**：只读账户，从源头禁止写操作
2. **代码层**：`SqlSecurityValidator` 辅助拦截危险关键词
3. **人工审核层**：高风险操作必须经过 Human-in-the-Loop 审批

**中断恢复实现**：

```java
// 1. 在 Graph 编译时配置中断点
CompileConfig.builder()
    .saverConfig(saverConfig)
    .interruptBefore(Set.of("SQL_EXECUTE_NODE"))  // 执行 SQL 前中断等待审批
    .build();

// 2. 执行 Graph，遇到中断点会返回中断信息
compiledGraph.invoke(state);  // 返回 InterruptException，包含 threadId

// 3. 审批通过后恢复执行（带超时自动驳回）
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
scheduler.schedule(() -> {
    // 超时未审批，自动驳回
    OverAllState.HumanFeedback rejectFeedback = OverAllState.HumanFeedback.builder()
        .action(HumanFeedbackAction.REJECT)
        .comment("审批超时自动驳回")
        .build();
    compiledGraph.resume(threadId, rejectFeedback);
}, 30, TimeUnit.MINUTES);

OverAllState.HumanFeedback feedback = OverAllState.HumanFeedback.builder()
    .action(HumanFeedbackAction.APPROVE)
    .comment("审批通过")
    .build();
compiledGraph.resume(threadId, feedback);
```

> **threadId 说明**：`threadId` 应使用 `java.util.UUID.randomUUID().toString()` 生成，并在整个流程中保持不变，用于关联中断状态和恢复请求。

### 3.9 监控与可观测性

#### 3.9.1 依赖配置

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

#### 3.9.2 监控指标记录

```java
@RestController
public class DataQueryController {
    private final MeterRegistry meterRegistry;

    // 通过构造函数注入 MeterRegistry（由 Spring Boot Actuator 自动配置）
    public DataQueryController(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @PostMapping("/nl2sql")
    public QueryResult query(@RequestBody QueryRequest request) {
        Timer.Sample sample = Timer.start(meterRegistry);
        try {
            QueryResult result = dataAnalysisService.executeQuery(request.getQuestion());
            sample.stop(Timer.builder("nl2sql.query.duration")
                .tag("status", result.isSuccess() ? "success" : "failure")
                .register(meterRegistry));
            meterRegistry.counter("nl2sql.query.total",
                "status", result.isSuccess() ? "success" : "failure").increment();
            return result;
        } catch (Exception e) {
            sample.stop(Timer.builder("nl2sql.query.duration")
                .tag("status", "error")
                .register(meterRegistry));
            throw e;
        }
    }
}
```

> **多数据源场景**：如使用多个 `JdbcTemplate`（例如一个用于向量库，一个用于业务库），必须在注入时使用 `@Qualifier` 明确指定，避免歧义。

#### 3.9.3 关键监控指标

| 指标 | 阈值 | 采集方式 |
|------|------|----------|
| SQL 生成成功率 | < 80% | 拦截 `Nl2SqlResponse` 中的错误 |
| SQL 执行成功率 | < 85% | 记录 `JdbcTemplate` 异常 |
| P99 响应时间 | > 5s | Micrometer `Timer` |
| 大模型调用延迟 | > 3s | 记录 `ChatClient` 调用耗时 |

### 3.10 数据库连接池与慢查询治理

高并发场景下，连接池耗尽和慢查询堆积是常见问题。建议采取以下措施：

- **连接池配置**：`maximum-pool-size` 根据并发量设置（一般 10~50），`connection-timeout` 设为 30s，`idle-timeout` 10min。
- **慢查询处理**：启用 `jdbcTemplate.setQueryTimeout(30)`，并监控数据库慢查询日志，对超过阈值的 SQL 进行人工 Review 和索引优化。
- **熔断降级**：当连接池活跃连接数超过 80% 时，可拒绝新请求并返回友好提示。

---
## 第四篇：向量数据库选型与部署

### 4.1 方案对比

| 特性 | AnalyticDB (云) | pgvector (开源) | SimpleVector (内存) |
|------|----------------|-----------------|---------------------|
| 部署方式 | 阿里云全托管 | Docker/RDS/物理机 | 无需部署 |
| 运维成本 | 低 | 中高 | 零 |
| 适用场景 | 生产、海量数据 | 开发/测试、中小规模 | 本地开发、单元测试 |
| 持久化 | 是 | 是 | 否 |
| HNSW 索引维度限制 | 无限制 | `dimensions` 可设任意值，但 HNSW ≤ **2000 维** | N/A |

**推荐路径**：开发阶段用 `SimpleVector` → 测试/中小规模生产用 `pgvector` → 大型生产/海量并发用 `AnalyticDB`。

### 4.2 pgvector 索引类型选择

| 索引类型 | 适用场景 | 维度限制 | 内存占用 | 默认值 |
|----------|----------|----------|----------|--------|
| **NONE** | 十万级以下小数据集，需要精确结果 | 无限制 | 无额外索引开销 | ✅ 默认值 |
| **HNSW** | 高查询性能，动态数据 | `dimensions` 任意，但 HNSW 索引要求 ≤2000 维 | 极高（3~5 倍原始数据） | - |
| **IVFFlat** | 百万级静态数据集 | 无限制（理论可达 16000 维） | 适中 | - |

> **关键注意**：`dimensions` 参数定义的是嵌入向量本身的维度数，可以设为任意值（如 1536、3072 等）。但若使用 HNSW 索引类型，该维度数**必须 ≤ 2000**（pgvector 0.7.0+ 官方限制），否则索引创建会失败并报错。若需使用超过 2000 维的嵌入向量（如 OpenAI `text-embedding-3-large` 3072 维），请选择 `IVFFlat` 或 `NONE` 索引类型。

### 4.3 其他向量数据库适配

若用户已有其他向量数据库（如 Milvus、Qdrant、Weaviate），或使用其他关系数据库（Oracle、SQL Server），当前 Spring AI Alibaba NL2SQL 未内置支持，但可参考以下适配路径：
- 实现 `VectorStore` 接口，对接目标向量库。
- 修改 `SchemaRecallNode` 中的检索逻辑，替换为自定义实现。
- 若目标数据库不是 PostgreSQL/MySQL，需要自行实现 `SQLDialect` 适配。

---
## 第五篇：生产问题分类与解决办法

### 5.1 Schema 召回失败

| 问题 | 解决办法 |
|------|----------|
| 召回不到表 | 执行 `vectorStore.schema()` 初始化表结构到向量库（幂等操作，会全量重建索引，应在低峰期执行） |
| 字段名不匹配 | 配置 Evidence 映射，建立业务术语到字段名的映射关系 |
| 相似度阈值不当 | 调整 `similarityThreshold`（如 0.7） |
| HNSW 索引创建失败（维度超 2000） | 改用 IVFFlat 或 NONE 索引类型，或换用 ≤2000 维的 embedding 模型 |

### 5.2 SQL 生成质量

| 问题 | 解决办法 |
|------|----------|
| JOIN 错误 | 在 Evidence 中明确 JOIN 条件，或检查数据库外键约束 |
| 聚合函数用错 | 优化 Prompt 或补充 Few-shot 示例 |
| 时间条件硬编码 | 启用查询重写节点的动态计算能力 |
| 生成不稳定 | 调低 `temperature` 至 0.1~0.3，启用 `SqlGenerateNode` 内置 3 轮差异化重试机制 |

### 5.3 执行安全（三层防线）

1. **数据库层**：使用只读账户，禁用写权限（**核心防线！**）
2. **代码层**：`SqlSecurityValidator` 辅助拦截危险关键词 + 强制 SELECT 语句
3. **人工审核层**：高风险操作通过 `interruptBefore` + `resume` 机制等待人工审批（带超时自动驳回）

### 5.4 常见异常速查表

| 异常现象 | 最可能原因 | 1分钟解决 |
|----------|------------|------------|
| `No tables found in vector store` | 表未向量化 | 调用 `vectorStore.schema()` |
| `Embedding token limit exceeded` | token 超限 | 降低 `max-token-count`，预留 15% 缓冲 |
| `DashScope rate limit exceeded` | 并发过高 | 增加重试或切换 `qwen-plus` |
| `LLM output is not a valid SELECT` | 模型生成了非 SELECT | 检查 Evidence 是否强调只读 |
| `PgVector connection refused` | Docker 未启动 | `docker ps` 检查容器 |
| `Relation "vector_store" does not exist` | 表未自动创建 | 设置 `initializeSchema: true` |
| `API key not found` | 环境变量未配置 | 确认 `DASHSCOPE_API_KEY` 已设置 |
| `HNSW index creation failed: dimension exceeds 2000` | 向量维度超 2000 | 改用 IVFFlat 或 NONE 索引类型 |
| `Missing extensions: hstore/uuid-ossp` | 扩展未安装 | 设置 `initializeSchema: true` 自动安装 |
| `Graph execution stuck` | 中断等待人工审批 | 调用 `resume(threadId, feedback)` 恢复，或配置超时自动驳回 |
| `SQL contains DROP` | 危险 SQL 生成 | 加固安全校验器或增加人工审核节点 |
| `Connection pool exhausted` | 连接池耗尽 | 增大 `maximum-pool-size`，检查慢查询 |

---
## 第六篇：调优指南

### 6.1 Schema 召回调优

- 为表和字段添加中文注释（最高 ROI）
- 配置 Evidence 业务规则（`KeywordExtractNode` 会自动召回）
- 向量化批处理参数：

```yaml
spring.ai.alibaba.nl2sql.embedding-batch:
  max-token-count: 4000        # 每批次最大令牌数（2000-8000）
  reserve-percentage: 0.15     # 预留 15% 缓冲，避免超限
  max-text-count: 10           # DashScope 限制为 10
```

> 该配置在 Spring AI Alibaba 1.1.3.0 版本中已固化，为稳定支持的内部 API。实际使用时，建议结合官方文档或源码确认最新格式。

### 6.2 大模型选择

| 模型 | 推荐场景 |
|------|----------|
| qwen-turbo | 快速原型，成本最低 |
| qwen-plus | 通用生产，平衡性能与成本 |
| qwen-max | 复杂 SQL、多表关联、高质量生成 |

### 6.3 性能优化——缓存策略

- **本地缓存**：常用表结构（Caffeine）
- **Redis**：问题 → SQL 映射，TTL 1 小时
- **向量库**：元数据本身已持久化

### 6.4 Python 代码执行安全与资源隔离

- **独立进程隔离**：通过 JVM 调用 Python 脚本时，建议使用独立进程（如 `ProcessBuilder`）而非内嵌引擎，防止 Python 内存泄漏影响主 JVM。
- **资源限制**：为 Python 进程设置 CPU/内存配额（如 Docker 容器限制），超时自动终止。
- **环境沙箱**：限制 Python 可访问的模块和文件系统，禁止执行系统命令。

---
## 第七篇：生产部署架构与安全最佳实践

### 7.1 部署架构

```
┌─────────────┐
│  负载均衡    │
│   (Nginx)   │
└──────┬──────┘
       │
   ┌───┴─────────────────────────────────┐
   ▼                                      ▼
┌─────────────┐                      ┌─────────────┐
│ NL2SQL Pod1 │                      │ NL2SQL Pod2 │
└──────┬──────┘                      └──────┬──────┘
       │                                      │
       └──────────────┬───────────────────────┘
                      ▼
            ┌─────────────────┐
            │     Redis       │
            │  (会话共享/缓存)  │
            └─────────────────┘
                      │
       ┌──────────────┼───────────────────────┐
       ▼              ▼                       ▼
┌──────────┐   ┌──────────┐            ┌──────────┐
│ pgvector │   │ 目标数据  │            │ DashScope│
│向量数据库 │   │ 库(只读)  │            │   API    │
└──────────┘   └──────────┘            └──────────┘
```

### 7.2 三层安全防线体系

| 防线 | 实现方式 | 防护效果 |
|------|----------|----------|
| **数据库层** | 只读账户，禁用 DDL/DML 权限 | 从源头禁止任何写操作（**最核心的防线**） |
| **代码层** | `SqlSecurityValidator` 拦截危险关键词 + 强制 SELECT | 辅助拦截 AI 误生成的危险 SQL |
| **人工审核层** | `interruptBefore` + `resume`，高风险操作等待审批，支持超时自动驳回 | 无法自动判定的操作由人类介入把关 |

### 7.3 生产环境初始化检查清单

- [ ] 数据库表结构已完成注释，具备良好的自解释性
- [ ] 表结构已成功向量化到 pgvector（或 AnalyticDB）
- [ ] DashScope API Key 已配置为环境变量 `DASHSCOPE_API_KEY`（或 `AI_DASHSCOPE_API_KEY`）
- [ ] **数据库连接配置为只读账户**（核心安全防线！）
- [ ] `initializeSchema: true` 已配置
- [ ] Evidence 参考证据已配置核心业务规则
- [ ] HNSW 索引下 embedding 维度 ≤ 2000（若超限改用 IVFFlat）
- [ ] 监控和告警已配置（Actuator + Prometheus + Grafana）
- [ ] Nacos 服务已部署并配置完成（如需动态 Prompt 配置），支持 2.X 和 3.X 版本，3.X 需使用 `spring-cloud-starter-alibaba-nacos-config` ≥2023.0.1.0
- [ ] 多租户场景需配置 `tenant_id` 元数据过滤，**推荐使用 `FilterExpressionBuilder` 编程式构造过滤条件**，避免字符串拼接注入风险。
- [ ] Python 代码执行已配置独立进程和资源隔离
- [ ] 连接池参数已根据并发量调优，慢查询治理方案已就绪

---
## 第八篇：常见异常速查表

| 异常现象 | 最可能原因 | 1分钟解决 |
|----------|------------|------------|
| `No tables found in vector store` | 表未向量化 | 调用 `vectorStore.schema()` |
| `Embedding token limit exceeded` | token 超限 | 降低 `max-token-count` |
| `DashScope rate limit exceeded` | 并发过高 | 增加重试或切换 `qwen-plus` |
| `LLM output is not a valid SELECT` | 模型生成了非 SELECT | 检查 Evidence 是否强调只读 |
| `PgVector connection refused` | Docker 未启动 | `docker ps` 检查容器 |
| `Relation "vector_store" does not exist` | 表未自动创建 | 设置 `initializeSchema: true` |
| `API key not found` | 环境变量未配置 | 确认 `DASHSCOPE_API_KEY` 已设置 |
| `HNSW index creation failed` | 向量维度超 2000 | 改用 IVFFlat 或 NONE 索引类型 |
| `Missing extensions: hstore` | 扩展未安装 | 设置 `initializeSchema: true` |
| `Graph execution stuck` | 中断等待人工审批 | 调用 `resume(threadId, feedback)` 恢复，或配置超时自动驳回 |
| `SQL contains DROP` | 危险 SQL 生成 | 加固安全校验器或增加人工审核节点 |
| `Connection pool exhausted` | 连接池耗尽 | 增大 `maximum-pool-size`，检查慢查询 |

---
## 第九篇：学习路线图与专家进阶

| 阶段 | 任务 | 时间 |
|------|------|------|
| 1 | 理解架构与原理（第一篇至第二篇） | 2天 |
| 2 | 运行电商示例（使用 SimpleVector） | 1天 |
| 3 | 配置 Evidence 和 pgvector | 1天 |
| 4 | 实现三层安全防线（只读账户 + 校验器 + 人工审核） | 1天 |
| 5 | 配置 `interruptBefore` 中断恢复机制及超时兜底 | 1天 |
| 6 | 调优索引、批处理、监控，治理慢查询 | 2天 |
| 7 | 阅读源码：`BaseNl2SqlService`、`nl2sqlGraph` 节点实现 | 1周 |
| 8 | 自定义 Graph 节点，扩展功能（如节点开关、多向量库适配） | 1~2周 |

---
## 附录 A：关键概念速查表

| 概念 | 一句话解释 |
|------|------------|
| NL2SQL | 自然语言 → SQL |
| RAG | 检索增强生成 |
| Schema 召回 | 从向量库检索相关表结构 |
| Evidence | 业务知识库，`KeywordExtractNode` 自动从向量库召回 |
| Graph 编排 | 多节点有状态执行流程，基于 StateGraph 框架 |
| pgvector | PostgreSQL 向量扩展，需 `vector`、`hstore`、`uuid-ossp` 三个扩展 |
| SimpleVector | 内存向量库，用于开发测试 |
| `interruptBefore` | Graph 编译时配置的中断点，用于 Human-in-the-Loop |
| `resume(threadId, feedback)` | 中断后恢复 Graph 执行的方法 |
| `SemanticConsistencyNode` | SQL 语义一致性验证节点，验证失败触发重试 |
| `SqlGenerateNode` | 重试专用节点，最多 3 轮，每轮注入不同失败信息，取最高分 |
| DataAgent | 基于 Spring AI Alibaba Graph 的企业级 AI 数据分析师 |

---
> **文档维护**：由于框架仍在快速迭代，部分配置属性名可能变化。请始终以 [Spring AI Alibaba 官方文档](https://github.com/alibaba/spring-ai-alibaba) 和 [Spring AI 文档](https://docs.spring.io/spring-ai/reference/) 为最终依据。
>
> **反馈与贡献**：欢迎提交 Issue 或 PR 改进本文档。