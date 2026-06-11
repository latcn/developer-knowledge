> **适用框架**：Spring AI Alibaba 1.1.3.0 / Spring AI 1.1.3  
> **核心定位**：面向高可靠性业务场景的生产级 NL2SQL 解决方案  
> **设计哲学**：正确性优先于智能性，人工兜底 + 自动沉淀

---
## 前言：NL2SQL 的本质与正确性挑战

**核心问题**：数据存储在关系型数据库中，而人用自然语言思考和表达。这两者之间存在“语言鸿沟”。

**第一性原理**：NL2SQL = **Schema 召回（RAG） + SQL 生成（LLM）**。它并非万能翻译器，而是一个**需要与人工审核、知识沉淀深度结合的系统工程**。

**正确性第一原则**：在金融、医疗、政务等强监管领域，AI 生成错误的 SQL 可能导致灾难性后果。因此，生产级系统必须建立 **“动态生成 → 人工审批 → 固化工具”** 的闭环，宁可牺牲实时性，也要保证正确性。

---
## 第一篇：架构设计 —— 双轨制（预定义工具 + 动态生成）

### 1.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         NL2SQL 双轨制架构                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐      ┌─────────────────────────────────────┐              │
│  │   用户输入    │─────▶│          问题分类与路由              │              │
│  └──────────────┘      │  - 语义匹配预定义工具库              │              │
│                        │  - 相似度阈值判断                    │              │
│                        └──────────────┬──────────────────────┘              │
│                                       │                                      │
│              ┌────────────────────────┼────────────────────────┐             │
│              ▼                        ▼                        ▼             │
│     ┌────────────────┐      ┌────────────────┐      ┌────────────────┐     │
│     │ 相似度>0.85    │      │ 0.6~0.85       │      │ 相似度<0.6     │     │
│     │ 直接匹配工具   │      │ 候选列表返回   │      │ 动态生成       │     │
│     └───────┬────────┘      │ 用户选择工具   │      │ (NL2SQL Graph) │     │
│             │               └───────┬────────┘      └────────┬───────┘     │
│             │                       │                        │             │
│             ▼                       ▼                        ▼             │
│     ┌────────────────┐      ┌────────────────┐      ┌────────────────┐     │
│     │ 直接执行       │      │ 根据用户选择   │      │ 生成SQL草案     │     │
│     │ (只读数据库)    │      │ 执行          │      │ + 质量评估      │     │
│     └───────┬────────┘      └───────┬────────┘      └────────┬───────┘     │
│             │                       │                        │             │
│             │                       │                        ▼             │
│             │                       │               ┌────────────────┐     │
│             │                       │               │  人工审批流程   │     │
│             │                       │               │ - 查看SQL草案   │     │
│             │                       │               │ - 执行/修改/拒绝│     │
│             │                       │               └────────┬───────┘     │
│             │                       │                        │             │
│             └───────────────────────┼────────────────────────┼─────────────┘
│                                     ▼                        ▼
│                           ┌─────────────────┐        ┌─────────────┐
│                           │ 知识沉淀与固化   │        │ 通知用户结果 │
│                           │ - 频次统计      │        └─────────────┘
│                           │ - 达到阈值→工具 │
│                           └─────────────────┘
```

### 1.2 核心设计原则

| 原则 | 说明 |
|------|------|
| **正确性 > 实时性** | 动态生成的 SQL 必须经过人工审批才能执行，拒绝“黑盒自动执行”。 |
| **预定义工具优先** | 通过参数化 SQL 模板覆盖 90% 以上的常见查询，极速且零风险。 |
| **动态生成作为补充** | 仅用于真正长尾、新颖、无法模板化的问题。 |
| **人工审批兜底** | 数据分析师审核 SQL 草案，可执行、修改或拒绝。 |
| **知识沉淀闭环** | 审批通过的 SQL 逐步固化为新工具，系统越用越“聪明”。 |

---
## 第二篇：预定义工具库设计

### 2.1 工具定义模型

```java
@Data
@Builder
public class QueryTool {
    private String id;                // 工具唯一标识
    private String name;              // 工具名称（如“销售额趋势”）
    private String description;       // 自然语言描述（用于语义匹配）
    private String sqlTemplate;       // 参数化 SQL 模板（使用 #{param}）
    private List<String> parameters;  // 参数列表（名称、类型、可选值）
    private String category;          // 业务分类
    private int usageCount;           // 使用次数（用于热度排序）
}
```

### 2.2 参数化 SQL 模板示例

```sql
-- 销售额趋势工具
-- 模板：SELECT DATE(order_date) as date, SUM(actual_amount) as sales
--       FROM orders
--       WHERE order_date >= #{startDate} AND order_date <= #{endDate}
--       GROUP BY DATE(order_date)
--       ORDER BY date
```

### 2.3 工具匹配策略

- **向量检索**：将工具的 `name` 和 `description` 向量化，用户问题同样向量化，计算相似度。
- **关键词匹配**：对高频词建立倒排索引，快速过滤候选工具。
- **阈值设置**：
  - **相似度 > 0.85**：直接匹配唯一工具，执行查询。
  - **0.6 ≤ 相似度 ≤ 0.85**：返回候选工具列表（按相似度排序），由用户选择具体工具，然后引导用户填写参数。
  - **相似度 < 0.6**：进入动态生成流程。

---
## 第三篇：动态生成与人工审批流程

### 3.1 NL2SQL Graph 调整（去掉自动执行，增加审批节点）

```
START → QueryRewriteNode → KeywordExtractNode → SchemaRecallNode → TableRelationNode
                                                                     │
                                                                     ▼
                                                              SqlGenerateNode
                                                                     │
                                                                     ▼
                                                            (生成 SQL 草案)
                                                                     │
                                                                     ▼
                                                          语法校验 + 风险评估
                                                   (预估影响行数、是否全表扫描)
                                                                     │
                                                                     ▼
                                                              HumanFeedbackNode
                                                         (中断等待人工审批)
                                                                     │
                                              ┌──────────────────┴──────────────────┐
                                              ▼                                      ▼
                                     审批通过                               审批拒绝/修改
                                              │                                      │
                                              ▼                                      ▼
                                      SqlExecuteNode                           END (通知用户)
                                              │
                                              ▼
                                      SemanticConsistencyNode (可选)
                                              │
                                              ▼
                                            END
```

### 3.2 SQL 草案质量评估

在提交人工审批前，系统自动执行以下检查：
- **语法校验**：使用数据库驱动或 SQL 解析器检查 SQL 是否合法。
- **风险评估**：
  - 预估返回行数（`EXPLAIN` 或数据库统计信息），若超过 10000 行，标记为“大结果集”。
  - 检查是否包含全表扫描（`EXPLAIN` 中的 `type=ALL`），若存在，标记为“性能风险”。
- 将评估结果随 SQL 草案一同展示给审批人。

### 3.3 人工审批界面设计

审批人（数据分析师）看到的信息：
- 用户原始问题
- 生成的 SQL 草案（高亮显示）
- 预估影响行数 + 性能风险提示
- 操作按钮：**执行**、**修改后执行**、**拒绝**

审批通过后，系统调用 `compiledGraph.resume(threadId, feedback)` 继续执行。

### 3.4 审批超时与自动驳回

```java
// 设置审批超时（如 2 小时）
private final ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
// 注意：应用关闭时应调用 scheduler.shutdown() 清理资源

scheduler.schedule(() -> {
    if (status == PENDING) {
        HumanFeedback reject = HumanFeedback.builder()
            .action(REJECT)
            .comment("审批超时自动驳回")
            .build();
        resume(threadId, reject);
        notifyUser("您的查询请求已超时，请稍后重试或联系数据团队。");
    }
}, 2, TimeUnit.HOURS);
```

---
## 第四篇：知识沉淀与工具固化机制

### 4.1 审批通过记录表

```sql
CREATE TABLE approved_queries (
    id BIGINT PRIMARY KEY,
    user_question TEXT NOT NULL,
    approved_sql TEXT NOT NULL,
    executor VARCHAR(50),          -- 审批人
    execute_time TIMESTAMP,
    usage_count INT DEFAULT 1,
    is_toolified BOOLEAN DEFAULT FALSE
);
```

### 4.2 自动固化策略

- **频次阈值**：同一 `approved_sql`（或高度相似的 SQL）被审批通过并执行 **≥ 3 次** 后，系统自动标记为候选工具。
- **人工确认**：数据分析师收到通知，检查该 SQL 是否可以参数化，然后将其转化为正式工具。
- **参数化抽象**：将常量替换为变量（如具体日期 → `#{startDate}`，具体产品名 → `#{productName}`）。

### 4.3 定期退化清理

- 对于长期未使用（如 90 天）的工具，可降级为“非推荐”状态，或提醒维护人员确认是否仍有效。
- 若业务变更导致工具 SQL 失效，提供人工编辑或重新生成功能。

---
## 第五篇：生产级示例（电商数据平台）

### 5.1 预定义工具示例

| 工具名称 | 自然语言描述 | SQL 模板 |
|---------|-------------|----------|
| 近7日销售额 | 查询最近7天每天的总销售额 | `SELECT DATE(order_date) AS date, SUM(actual_amount) AS sales FROM orders WHERE order_date >= CURDATE() - INTERVAL 7 DAY GROUP BY DATE(order_date)` |
| 品类销售额排行 | 按品类统计销售额，取前N名 | `SELECT c.category_name, SUM(oi.quantity * oi.unit_price) AS sales FROM order_items oi JOIN products p ON oi.product_id = p.product_id JOIN categories c ON p.category_id = c.category_id GROUP BY c.category_name ORDER BY sales DESC LIMIT #{topN}` |
| 复购用户数 | 统计指定时间段内下单次数≥2的用户数 | `SELECT COUNT(DISTINCT user_id) FROM orders WHERE order_date BETWEEN #{startDate} AND #{endDate} GROUP BY user_id HAVING COUNT(*) >= 2` |

### 5.2 动态生成 + 人工审批完整流程代码

```java
@Service
public class HybridQueryService {

    private final ToolMatcher toolMatcher;
    private final Nl2SqlGraph nl2SqlGraph;
    private final ApprovalService approvalService;
    private final JdbcTemplate jdbcTemplate;

    public QueryResult handleQuery(String userQuestion) {
        // 1. 匹配预定义工具
        Optional<QueryTool> tool = toolMatcher.match(userQuestion);
        if (tool.isPresent()) {
            String sql = tool.get().getSqlTemplate();
            String finalSql = replaceParameters(sql, userQuestion);
            List<Map<String, Object>> result = jdbcTemplate.queryForList(finalSql);
            return QueryResult.success(result, finalSql, "TOOL");
        }

        // 2. 动态生成
        Nl2SqlRequest request = Nl2SqlRequest.builder()
                .query(userQuestion)
                .humanFeedback(true)
                .build();

        Nl2SqlResponse response = nl2SqlGraph.invoke(request);
        String draftSql = response.getSql();

        // 3. 质量评估
        SqlRiskAssessment risk = assessSqlRisk(draftSql);
        ApprovalTask task = approvalService.createTask(userQuestion, draftSql, risk);
        notifyApprover(task);

        return QueryResult.pending(task.getId(), 
            "您的查询已提交审批，预计响应时间 ≤ 2 小时，审批通过后结果将发送到您的工作台。");
    }

    private SqlRiskAssessment assessSqlRisk(String sql) {
        // 语法校验（使用 JSqlParser 或直接执行 EXPLAIN）
        // 预估行数、是否全表扫描等
        return new SqlRiskAssessment(rowCount, hasFullTableScan);
    }

    @PreDestroy
    public void cleanup() {
        scheduler.shutdown(); // 应用关闭时清理调度器
    }
}
```

### 5.3 工具固化后台任务

```java
@Scheduled(cron = "0 0 2 * * ?")
public void toolifyCandidates() {
    List<ApprovedQuery> candidates = approvalService.findApprovedQueriesNotToolified();
    Map<String, Long> groupBySql = candidates.stream()
        .collect(Collectors.groupingBy(ApprovedQuery::getApprovedSql, Collectors.counting()));
    
    for (Map.Entry<String, Long> entry : groupBySql.entrySet()) {
        if (entry.getValue() >= 3) {
            notifyAnalyst("高频审批通过的SQL，建议固化为工具", entry.getKey());
        }
    }
}
```

---
## 第六篇：安全与审计

### 6.1 权限控制

| 角色 | 权限 |
|------|------|
| **普通用户** | 只读查询（预定义工具），可提交动态生成需求。 |
| **数据分析师** | 审批动态生成的 SQL，固化工具，查看审计日志。 |
| **管理员** | 配置数据源、管理用户、查看全部日志。 |

### 6.2 审计日志表

```sql
CREATE TABLE audit_log (
    id BIGINT PRIMARY KEY,
    user_id VARCHAR(50),
    user_question TEXT,
    executed_sql TEXT,
    is_tool BOOLEAN,
    approval_status VARCHAR(20),
    approver VARCHAR(50),
    execution_time TIMESTAMP,
    row_count INT
);
```

### 6.3 安全三原则

1. **数据库只读账户**：所有执行 SQL 的数据库连接必须是只读账户（核心防线）。
2. **SQL 安全校验器**：辅助拦截危险关键词（仅作为辅助，不依赖）。
3. **人工审批**：所有动态生成的 SQL 必须经过审批，杜绝自动写操作。

---
## 第七篇：部署与监控

### 7.1 部署架构

- **Spring Boot 应用**：包含 Graph 编排、工具匹配、审批流程。
- **PostgreSQL + pgvector**：存储表结构向量、证据、审批记录、工具定义。
- **Redis**：缓存用户会话、审批任务临时状态。
- **DashScope API**：大模型服务。
- **内部审批系统**（钉钉/企微/飞书）：通知与回调。

### 7.2 关键监控指标

| 指标 | 阈值 | 用途 |
|------|------|------|
| 预定义工具命中率 | > 85% | 评估工具库覆盖度 |
| 动态生成请求数/天 | 不限 | 监控长尾需求变化 |
| 平均审批时长 | < 2 小时 | SLA 达标性 |
| 审批通过率 | > 70% | 动态生成质量 |
| 工具固化数/周 | 统计 | 知识沉淀速度 |

---
## 第八篇：常见问题与解决方案

| 问题 | 解决方案 |
|------|----------|
| 用户问题模糊，难以匹配工具 | 返回候选工具列表（相似度0.6~0.85），让用户选择；或引导用户补充结构化信息。 |
| 动态生成的 SQL 频繁被拒绝 | 检查 Schema 召回质量，优化 Evidence，或调整生成模型参数。 |
| 审批人长时间不响应 | 设置超时自动驳回 + 升级提醒（短信/电话）。 |
| 固化工具数量过多，难以维护 | 定期清理低频工具（如 90 天未使用），标记为“已废弃”。 |
| 业务变化导致工具 SQL 失效 | 提供“工具版本管理”，支持回滚和重新生成。 |

---
## 第九篇：学习路线图（双轨制实践）

| 阶段 | 目标 | 任务 |
|------|------|------|
| 第1周 | 构建预定义工具库 | 梳理业务高频查询，编写 20~30 个参数化 SQL 模板。 |
| 第2周 | 集成 NL2SQL Graph + 人工审批 | 搭建审批流程，实现动态生成 + 人工审核闭环。 |
| 第3周 | 部署生产环境 | 配置只读数据库、DashScope、审批通知。 |
| 第4周 | 灰度上线，监控指标 | 分析工具命中率、审批通过率，收集用户反馈。 |
| 第5~8周 | 知识沉淀与工具固化 | 定期将高频审批通过的 SQL 固化为新工具，迭代优化。 |
| 第9周及以后 | 持续运营 | 定期清理低效工具，调整相似度阈值，更新业务知识库。 |

---
## 附录：关键概念速查表

| 概念 | 解释 |
|------|------|
| **预定义工具** | 参数化 SQL 模板，由分析师预先编写，覆盖常见查询，直接执行零风险。 |
| **语义视图** | 固化 SQL 逻辑的一种方式（如将复杂关联定义为视图），预定义工具可基于语义视图构建。 |
| **动态生成** | 通过 NL2SQL Graph + 大模型生成 SQL 草案，必须经过人工审批。 |
| **人工审批** | 数据分析师审核 SQL 草案，可执行、修改或拒绝。 |
| **知识沉淀** | 将审批通过的 SQL 记录、统计、参数化后固化为新工具。 |
| **正确性优先** | 系统设计的第一原则，动态 SQL 绝不自动执行。 |
| **双轨制** | 预定义工具与动态生成并行，大部分查询走工具，极少部分走审批。 |

---
> **最终建议**：对于高可靠性业务场景，强烈推荐采用本技术文档描述的 **“预定义工具 + 动态生成 + 人工审批 + 知识沉淀”** 双轨制架构。它既保证了数据查询的正确性，又赋予了系统应对未知需求的能力，且能随着业务发展不断自我进化。
