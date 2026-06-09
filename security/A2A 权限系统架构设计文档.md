
> **技术栈**  
> - JDK 21（开启虚拟线程）  
> - Spring Boot 3.5.3（Spring MVC + Tomcat）  
> - Spring Cloud Alibaba 2025.0.0.0  
> - Spring Authorization Server 1.8.3  
> - Spring AI Alibaba 1.1.2.0（Spring AI BOM 1.1.2）  
> - Nacos 3.0.3、MySQL 8.0、Redis 7.x  
> - jCasbin 1.81.0 + casbin-spring-boot-starter 1.8.0  
> - Caffeine、Lettuce（同步Redis客户端）  
> - Logback + Logstash Encoder（JSON Lines审计日志）  
>
> **包名**：`io.github.latcn.a2a.security`  
> **模块**：  
> - `a2a-authorization-server`  
> - `a2a-common-permission`  
> - `a2a-security-starter`

---
## 目录

1. [背景与目标](#1-背景与目标)
2. [核心设计理念](#2-核心设计理念)
3. [总体架构与模块划分](#3-总体架构与模块划分)
4. [模块间依赖关系](#4-模块间依赖关系)
5. [核心接口与组件（同步版·责任链模式）](#5-核心接口与组件同步版责任链模式)
6. [数据模型设计（协同优化版）](#6-数据模型设计协同优化版)
7. [jCasbin策略引擎集成与策略同步](#7-jcasbin策略引擎集成与策略同步)
8. [授权服务器Token声明与职责划分](#8-授权服务器token声明与职责划分)
   - 8.1 Token声明
   - 8.2 Scope分类
   - 8.3 权限判断职责划分规则
   - 8.4 批量场景优化（两种模式）
   - 8.5 核心场景覆盖
   - 8.6 大模型生成权限的保障机制
9. [大模型原生安全防护与平台级管控](#9-大模型原生安全防护与平台级管控)
   - 9.1 MCP Tool 调用统一权限切面
   - 9.2 工具级 Scope 限制（tool_scope）
   - 9.3 Human-in-the-Loop确认机制
   - 9.4 凭据保险箱设计
   - 9.5 防篡改与防重放综合设计
   - 9.6 限流熔断与策略同步
10. [对话驱动的动态授权与资源约束](#10-对话驱动的动态授权与资源约束)
    - 10.1 约束传递模式
    - 10.2 用户确认与统计摘要
    - 10.3 元数据服务设计
    - 10.4 完整授权流程
11. [配置与部署](#11-配置与部署)
12. [审计日志设计](#12-审计日志设计)
13. [安全加固矩阵](#13-安全加固矩阵)
14. [扩展点设计](#14-扩展点设计)
15. [总结与修正记录](#15-总结与修正记录)

---
## 1. 背景与目标

在政企、军工或大型金融机构的封闭内网中，存在一个致命误区：“系统部署在物理隔离的内网，外面黑客进不来，里面都是‘自己人’，AI Agent 之间互相调用不需要复杂的权限系统。”这一理念在 AI Agent 时代已经彻底破产。

**五大必要性**：
1. **应对 AI 原生风险**：间接提示词注入可使 Agent 执行恶意指令；大模型幻觉可能调用核心 API。权限系统是最后的物理防线。
2. **控制爆炸半径**：内网隔离防不住内部威胁。最小特权原则强制实施零信任，将安全事件的横向移动限制在最小范围。
3. **数据密级隔离与行级管控**：内网数据有严格密级划分，Agent 必须以用户权限为边界，防止数据聚合泄露。
4. **合规审计与防抵赖**：审计要求追溯完整调用链（用户→Agent→工具），权限系统记录不可篡改的决策日志。
5. **高可用性防雪崩**：权限系统通过限流和配额管理，防止失控 Agent 耗尽内网核心资源。

**核心目标**：
- **身份认证**：Agent间认证、用户身份透传、防冒充。
- **最小权限**：Scope细粒度、资源实例级控制、行级过滤。
- **性能优化**：虚拟线程、批量令牌Redis化、三级缓存。
- **可审计性**：全链路追溯、零丢失审计日志。
- **高可用性**：本地验签、Redis Streams可靠广播。
- **大模型安全**：Tool Calling拦截、动态限流、HITL。

---
## 2. 核心设计理念

- **同步编码，异步性能**：全面拥抱 JDK 21 虚拟线程，摒弃复杂的响应式编程，以极低的心智负担支撑大模型 SSE 长连接与 A2A 高并发。
- **分类保底，标签微调**：构建“资源分类（刚性骨架）+ 资源标签（柔性血肉）”的双层 ABAC 模型，兼顾批量授权性能与细粒度动态拦截。
- **大模型原生安全**：权限校验从 HTTP 接口层下沉至 Spring AI 的 Tool Calling 层，防御大模型幻觉与提示词注入。
- **责任链可扩展**：权限校验采用责任链模式，每个校验器独立，支持热插拔。
- **最小耦合**：授权服务器不直接访问业务数据库，通过约束传递和元数据服务实现解耦。

---
## 3. 总体架构与模块划分

**四个独立部署单元**：
1. **授权服务器**（Authorization Server）— 基于 Spring Authorization Server，负责颁发代理令牌、管理 ACL 和用户同意。
2. **Agent**（既是 OAuth2 客户端，又是资源服务器）— 基于 Spring AI Alibaba A2A，内嵌安全过滤器与权限评估器。
3. **Nacos** — A2A 注册中心与 AgentCard 存储。
4. **Redis** — 一次性令牌、批量任务ID、Streams广播、限流计数器。

**Maven父模块**：`io.github.latcn:a2a-security:1.0.0`

```
a2a-security/
├── a2a-authorization-server/         # 授权服务器
│   └── src/main/java/io/github/latcn/a2a/security/authz/
│       ├── token/                    # Token Exchange端点 + 定制器
│       ├── acl/                      # Agent间ACL管理
│       ├── consent/                  # 用户同意管理
│       ├── audit/                    # 审计日志记录
│       └── config/                   # 安全配置
├── a2a-common-permission/            # 独立权限JAR
│   └── src/main/java/io/github/latcn/a2a/security/permission/
│       ├── api/                      # PermissionChecker, ResourceTagService等接口
│       ├── jcasbin/                  # Enforcer配置、策略管理、Watcher
│       ├── tag/                      # 资源标签服务（三级缓存）
│       ├── chain/                    # 责任链处理器（Authz侧和Agent侧）
│       └── config/                   # 自动配置
└── a2a-security-starter/             # Agent安全起步依赖
    └── src/main/java/io/github/latcn/a2a/security/agent/
        ├── client/                   # TokenExchangeService, A2aClientFilter
        ├── server/                   # A2aJwtAuthenticationFilter, OneTimeTokenValidator
        ├── tool/                     # SecurityToolCallbackWrapper, 限流器
        ├── chain/                    # Agent侧责任链实现
        └── config/                   # 自动配置（虚拟线程、过滤器链）
```

---
## 4. 模块间依赖关系

**编译时依赖**（箭头方向表示“依赖者 → 被依赖者”）：

```
┌─────────────────────┐
│ a2a-agent-xxx       │  业务Agent
└──────────┬──────────┘
           │ depends on
           ▼
┌─────────────────────┐      ┌────────────────────────┐
│ a2a-security-starter│─────▶│ a2a-common-permission  │
└──────────┬──────────┘      └────────────────────────┘
           │
           │ depends on
           ▼
┌─────────────────────────────────────────────────────────────────┐
│ 外部依赖：Spring Boot 3.5.3, Spring AI Alibaba 1.1.2.0,        │
│ jCasbin, Redis (Lettuce), MySQL, Caffeine, Logstash Encoder    │
└─────────────────────────────────────────────────────────────────┘
```

**运行期服务调用关系**：授权服务器负责 Token Exchange、ACL、Consent 和批量预授权（但不直接查询业务库）；Agent 负责本地验签、一次性令牌校验、责任链权限评估、MCP工具拦截。

---
## 5. 核心接口与组件（同步版·责任链模式）

### 5.1 PermissionCheckHandler（权限校验器接口）

```java
public interface PermissionCheckHandler {
    CheckResult check(PermissionCheckContext context);
    enum CheckResult { ALLOW, DENY, ABSTAIN }
    default int order() { return 0; }
}
```

### 5.2 PermissionCheckContext（校验上下文）

```java
public class PermissionCheckContext {
    private final String userId;
    private final String resourceId;
    private final String operation;
    private final String resourceType;
    private final Map<String, Object> userAttributes;
    private final Map<String, List<String>> resourceTags;
    private final Jwt jwtToken;
    // constructor, getters...
}
```

### 5.3 内置处理器

| 处理器 | 顺序 | 职责 |
|--------|------|------|
| `DenyOverrideHandler` | `Integer.MIN_VALUE` | 特例拒绝优先 |
| `AllowOverrideHandler` | `-1000` | 特例允许 |
| `CategoryQuickHandler` | `100` | 分类快速通道 |
| `AbacJCasbinHandler` | `200` | 标签+ABAC动态评估 |

### 5.4 双重责任链实现

- **`AuthzPermissionChain`**（授权服务器侧）：仅包含 `OverrideHandler` 和 `CategoryHandler`，用于静态权限预判。方法签名：`Set<String> preAuthorize(String userId, List<String> resourceIds, String operation)`。
- **`AgentPermissionChain`**（Agent侧）：包含所有处理器，用于最终运行时决策。

```java
@Component
public class AgentPermissionChain {
    private final List<PermissionCheckHandler> handlers;
    public boolean check(PermissionCheckContext ctx) {
        for (PermissionCheckHandler h : handlers) {
            CheckResult r = h.check(ctx);
            if (r == ALLOW) return true;
            if (r == DENY) return false;
        }
        return false;
    }
}
```

### 5.5 Spring Security PermissionEvaluator 集成

```java
@Component
public class A2aPermissionEvaluator implements PermissionEvaluator {
    private final AgentPermissionChain chain;
    @Override
    public boolean hasPermission(Authentication auth, Object targetId, String targetType, Object permission) {
        // 构建上下文并调用 chain.check()
    }
}
```

---
## 6. 数据模型设计（协同优化版）

### 6.1 核心表概览

| 表名 | 作用 |
|------|------|
| `resource_category` | 资源分类（含`path`物化路径） |
| `resource` | 资源实例，关联分类 |
| `user_category_permission` | 用户对分类的批量授权 |
| `user_resource_permission` | 用户对资源实例的特例授权（`revoked_at` BIGINT） |
| `resource_tag_definition` | 标签定义（支持分组） |
| `resource_tag_mapping` | 资源-标签关联 |
| `casbin_rule` | jCasbin策略表 |
| `a2a_acl` | Agent间调用ACL |
| `user_consent` | 用户委派授权 |
| `oauth2_client_settings` | 客户端安全配置 |
| `credential_vault` | 凭据保险箱（A2E） |

### 6.2 关键DDL示例

```sql
CREATE TABLE resource_category (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(64) NOT NULL,
    resource_type VARCHAR(64) NOT NULL,
    parent_id BIGINT DEFAULT 0,
    path VARCHAR(512) NOT NULL,
    created_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    created_by VARCHAR(64),
    UNIQUE KEY uk_category_type (name, resource_type),
    INDEX idx_path (path)
);

CREATE TABLE user_resource_permission (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id VARCHAR(64) NOT NULL,
    resource_type VARCHAR(64) NOT NULL,
    resource_id VARCHAR(128) NOT NULL,
    operation VARCHAR(32) NOT NULL,
    effect VARCHAR(8) CHECK (effect IN ('ALLOW','DENY')),
    granted_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    revoked_at BIGINT NOT NULL DEFAULT 0,
    expires_at TIMESTAMP(6) NULL,
    UNIQUE KEY uk_active (user_id, resource_type, resource_id, operation, revoked_at)
);

CREATE TABLE credential_vault (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    credential_ref VARCHAR(64) UNIQUE NOT NULL,
    client_id VARCHAR(64) NOT NULL,
    external_system VARCHAR(64) NOT NULL,
    credential_type VARCHAR(16) NOT NULL,
    encrypted_secret TEXT NOT NULL,
    scope VARCHAR(256),
    expires_at TIMESTAMP(6),
    created_at TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6)
);
```

### 6.3 数据一致性原则

- 不使用外键约束，应用层维护一致性。
- 删除资源时同步删除 `resource_tag_mapping` 和 `user_resource_permission` 中关联记录。
- 删除分类时，需先将引用该分类的资源的 `category_id` 置为 NULL 或拒绝删除。
- 权限决策优先级：DENY特例 > ALLOW特例 > 分类授权 > ABAC标签 > 默认拒绝。

---
## 7. jCasbin策略引擎集成与策略同步

### 7.1 模型配置 `model.conf`

```ini
[request_definition]
r = sub_attr, obj, act

[policy_definition]
p = sub_rule, obj, act

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = eval(p.sub_rule) && r.obj == p.obj && r.act == p.act
```

### 7.2 Spring Boot配置

```yaml
casbin:
  enableCasbin: true
  useSyncedEnforcer: true
  storeType: jdbc
  tableName: casbin_rule
  model: classpath:casbin/model.conf
```

### 7.3 Enforcer配置与策略同步

- 使用 `SyncedEnforcer` 保证并发安全。
- 策略变更时通过 **Redis Streams** 广播 `policy:stream` 消息，所有 Agent 实例消费后调用 `enforcer.loadPolicy()`。

```java
// 策略变更发布
redisTemplate.opsForStream().add("policy:stream", Map.of("policyId", id, "timestamp", now));

// Agent端消费者
@StreamListener(target = "policy:stream")
public void onPolicyChange(MapRecord<String, String, String> msg) {
    enforcer.loadPolicy();
}
```

---
## 8. 授权服务器Token声明与职责划分

### 8.1 Token包含的声明

| 声明 | 说明 |
|------|------|
| `iss`, `sub`, `aud`, `exp`, `iat`, `jti` | 标准JWT字段 |
| `scope` | 授权范围（空格分隔） |
| `tool_scope` | 工具级授权范围（字符串数组） |
| `act` | 最终用户信息（`sub`, `department`, `clearance`等） |
| `delegation_chain`, `delegation_remaining` | 多跳委托信息 |
| `single_use` | 一次性令牌 |
| `resource_instance` | 单资源ID |
| `bulk_mode`, `batch_task_id`, `bulk_partial` | 批量模式 |
| `constraint_hash` | 资源约束条件的哈希签名（约束传递模式） |
| `data_signature` | Redis中存储数据的签名（RSA私钥签名） |
| `credential_ref` | 凭据引用 |
| `body_hash` | 请求体的哈希值（防篡改） |

### 8.2 Scope分类

| 层级 | 类型 | 示例 | 说明 |
|------|------|------|------|
| L1 | 平台级 | `read`, `write`, `admin` | 粗粒度，面向用户Consent界面 |
| L2 | 服务级 | `doc:read`, `finance:report:export` | 按业务领域划分，用于ACL/Consent |
| L3 | 资源实例级 | `doc:123:read` | 针对具体资源，用于特例授权 |
| L4 | 工具级 | `tool:search_documents` | 控制大模型可调用的Function/Tool |

### 8.3 权限判断职责划分规则

| 规则 | 判断依据 | 执行者 |
|------|----------|--------|
| 1. 分类授权 | `resource.category_id` + `user_category_permission` | 授权服务器 |
| 2. 特例授权 | `user_resource_permission` | 授权服务器 |
| 3. 资源标签 | `resource_tag_mapping` | Agent |
| 4. ABAC组合 | 用户属性 + 资源标签 | Agent |
| 5. 请求上下文 | 时间、IP等 | Agent |
| 6. Scope粗粒度 | `user_consent` / 客户端白名单 | 授权服务器 |
| 7. 批量优化 | 仅适用于规则1/2 | 混合 |

**核心原则**：授权服务器负责静态可预判的权限，Agent负责动态（标签、上下文）权限。

### 8.4 批量场景优化（两种模式）

#### 模式一：显式资源列表模式（适用于资源ID已知）

- **开启条件**：调用方已经知道资源ID列表（如用户明确指定或前序查询获得）。
- **流程**：
  1. 调用方将 `resource_ids` 列表发送给授权服务器。
  2. 授权服务器调用 `AuthzPermissionChain.preAuthorize` 判断静态权限。
  3. 允许的ID存入 Redis Set：`batch:{batchId}`，并使用授权服务器私钥对列表计算签名，签名值放入 JWT 的 `data_signature`。
  4. JWT 携带 `bulk_mode=true, batch_task_id=batchId`。
  5. Agent 从 Redis 读取后验签，使用 `SISMEMBER` 判断。

#### 模式二：约束传递模式（推荐，适用于意图驱动）

- **开启条件**：调用方只知道约束条件（如分类、时间范围），不知道具体资源ID。
- **流程**：
  1. 调用方将 `resource_constraint`（如 `{categories:["financial"], timeRange:"last_week"}`）发送给授权服务器。
  2. 授权服务器验证约束合法性（分类是否存在、时间格式等），不查询资源列表。
  3. 授权服务器对约束条件计算哈希签名（`constraint_hash`），放入 JWT。
  4. JWT 携带 `constraint_mode=true, constraint_hash`。
  5. Agent 收到请求后，自行根据约束条件查询资源列表，然后使用 `AgentPermissionChain` 进行最终权限过滤（取交集：约束范围 ∩ 用户权限 ∩ Agent能力）。

**签名机制**：
- 授权服务器使用自己的 RSA 私钥对资源列表或约束条件进行签名。
- Agent 使用授权服务器的公钥（从 JWKS 获取）验签，确保数据未被篡改。

### 8.5 核心场景覆盖

本架构聚焦以下6类场景：

1. **A2A调用合法性**：Token Exchange + ACL + 多跳委托。
2. **MCP工具使用合法性**：ToolCallback包装器 + `tool_scope` 校验 + 限流 + HITL。
3. **资源读写合法性**：分类/特例 + 标签ABAC。
4. **知识库检索权限（RAG）**：使用 `filterAllowed` 注入检索条件。
5. **临时权限与即时审批**：增量scope申请 + 审批队列（扩展设计）。
6. **审计与行为基线**：JSON Lines审计日志 + 限流告警。

### 8.6 大模型生成权限的保障机制

- **四层防护**：scope白名单、合法性校验、用户同意、动态裁剪。
- **结构化输出**：使用JSON Schema约束大模型输出。
- **核心原则**：大模型输出仅作为提示（hint），授权服务器独立校验。

---
## 9. 大模型原生安全防护与平台级管控

### 9.1 MCP Tool 调用统一权限切面

```java
@Component
public class SecurityToolCallbackWrapper implements ToolCallback {
    private final ToolCallback delegate;
    private final AgentPermissionChain permissionChain;
    private final ResourceMetadataService metadataService;
    private final ToolResourceExtractor extractor;
    private final JwtHolder jwtHolder;

    @Override
    public String call(String toolInput) {
        Jwt jwt = jwtHolder.getJwt();
        // 1. 工具级 scope 校验
        List<String> toolScope = jwt.getClaimAsStringList("tool_scope");
        String toolName = delegate.getToolDefinition().name();
        if (toolScope != null && !toolScope.contains(toolName)) {
            throw new AccessDeniedException("当前会话不允许调用工具: " + toolName);
        }
        // 2. 提取资源ID列表
        List<ResourceRef> resources = extractor.extract(toolInput, delegate.getToolDefinition());
        // 3. 批量获取元数据并校验权限
        for (ResourceRef ref : resources) {
            PermissionCheckContext ctx = buildContext(ref);
            if (!permissionChain.check(ctx)) {
                throw new AccessDeniedException("无权通过工具访问资源: " + ref.getResourceId());
            }
        }
        // 4. 执行原工具
        return delegate.call(toolInput);
    }
}
```

### 9.2 工具级 Scope 限制（tool_scope）

- Route Agent 在申请令牌时，可申请一个临时、细粒度的工具范围（如 `["search_documents", "read_document"]`），不包括高风险工具（如 `delete_document`）。
- 授权服务器将 `tool_scope` 放入 JWT。
- `SecurityToolCallbackWrapper` 在执行前校验工具名是否在列表中。

### 9.3 Human-in-the-Loop确认机制

- 使用 `@RequireHumanConfirm` 注解标记高风险工具。
- 执行前通过WebSocket推送确认请求，阻塞等待用户响应（虚拟线程不消耗OS线程）。

### 9.4 凭据保险箱设计

- 表 `credential_vault` 存储加密后的API Key / OAuth Token。
- Agent持有 `credential_ref`，平台在调用外部系统时动态解析注入。

### 9.5 防篡改与防重放综合设计

| 攻击类型 | 防护措施 | 实现 |
|----------|----------|------|
| Token伪造 | JWT签名验证 | 授权服务器私钥签名，Agent用公钥验签 |
| Token重放 | 一次性令牌 + Redis SET NX | `single_use=true` + `jti` |
| 请求体篡改 | 请求体哈希 | Route Agent计算请求体SHA-256，放入JWT `body_hash`，Agent重新计算比对 |
| Redis数据篡改 | 数据签名 | 授权服务器对存入Redis的数据用私钥签名，Agent验签 |
| 约束条件篡改 | 约束哈希 | 约束条件计算哈希放入 `constraint_hash`，Agent重新计算比对 |

### 9.6 限流熔断与策略同步

- 基于Redis滑动窗口，按 Agent ID、User ID、Tool Name 三维度独立限流。
- 策略变更通过Redis Streams广播。

---
## 10. 对话驱动的动态授权与资源约束

### 10.1 约束传递模式

**核心思想**：调用方只传递约束条件（不传递具体资源ID），授权服务器验证约束合法性并签发JWT，目标Agent自己根据约束查询资源并执行权限过滤。

**优势**：
- 资源ID不暴露给中间组件（路由Agent、授权服务器），减少泄露风险。
- 授权服务器无需耦合业务数据库。
- 减少一次额外的A2A调用。

### 10.2 用户确认与统计摘要

**用户确认前的信息展示原则**：只展示统计摘要，不展示具体资源标识或内容。

**实现方式**：
- 授权服务器调用独立的**元数据服务**（见下节），根据约束条件获取资源数量、分类分布、密级分布等统计信息。
- 将统计信息返回给路由Agent，用于用户确认界面。
- 用户确认后，授权服务器才签发JWT。

**确认界面示例**：
```
您即将授权“财务助手Agent”代表您执行以下操作：
- 读取财务分类下的文档
- 时间范围：最近一周
- 涉及文档数量：23个（其中机密级3个，内部级20个）

是否同意？
[同意] [拒绝] [查看详细分类]
```

### 10.3 元数据服务设计

**目的**：为授权服务器提供资源统计信息，而不暴露具体资源内容或ID。

**接口定义**：
```
POST /api/metadata/statistics
Request:
{
  "constraint": {
    "categories": ["financial"],
    "timeRange": "last_week",
    "tags": [{"key": "classification", "value": "confidential"}]
  }
}
Response:
{
  "total": 23,
  "byCategory": {"financial": 23},
  "byTag": {"classification.confidential": 3, "classification.internal": 20},
  "byTimeRange": {"last_week": 23}
}
```

**部署方式**：
- 每个业务Agent可单独实现一个元数据服务端点。
- 或企业级统一使用Elasticsearch等索引服务。
- 授权服务器通过服务发现（Nacos）调用目标Agent的元数据接口。

**安全约束**：
- 元数据服务必须验证调用方（授权服务器）的身份（mTLS或固定IP白名单）。
- 元数据服务不返回资源ID或内容。

### 10.4 完整授权流程

```mermaid
sequenceDiagram
    participant User
    participant RouteAgent
    participant Authz
    participant MetaService
    participant AgentB

    User->>RouteAgent: "总结我最近一周的财务文档"
    RouteAgent->>Authz: 1. 预请求 + 约束条件
    Authz->>MetaService: 2. 查询统计信息
    MetaService-->>Authz: {total:23, byClassification:{financial:23}}
    Authz-->>RouteAgent: 3. 需用户确认 + 统计信息
    RouteAgent->>User: 4. 显示确认界面
    User-->>RouteAgent: 5. 同意
    RouteAgent->>Authz: 6. 确认授权 + 约束条件
    Authz->>Authz: 7. 生成JWT（constraint_hash, scope, tool_scope）
    Authz-->>RouteAgent: 8. JWT
    RouteAgent->>AgentB: 9. 转发请求 + JWT + 约束条件
    AgentB->>AgentB: 10. 验签、约束哈希校验
    AgentB->>AgentB: 11. 根据约束查询资源列表
    AgentB->>AgentB: 12. 权限过滤（分类+标签ABAC）
    AgentB-->>RouteAgent: 13. 结果
```

---
## 11. 配置与部署

```yaml
spring:
  threads:
    virtual:
      enabled: true
  datasource:
    url: jdbc:mysql://...
    username: ${DB_USER}
    password: ${DB_PASS}
  redis:
    host: redis.internal
    port: 6379
    timeout: 2s
    lettuce:
      pool:
        max-active: 10

casbin:
  enableCasbin: true
  useSyncedEnforcer: true
  storeType: jdbc
  tableName: casbin_rule
  model: classpath:casbin/model.conf

a2a:
  security:
    bulk:
      enabled: true
      max-batch-size: 300
    hitl:
      enabled: true
      default-timeout: 60s
    credential-vault:
      enabled: true
    audit:
      never-block: false
    metadata-service:
      timeout: 5s
      retry: 3
```

**Logback配置（审计）**：

```xml
<configuration>
    <appender name="ASYNC_AUDIT" class="ch.qos.logback.classic.AsyncAppender">
        <queueSize>10240</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <neverBlock>false</neverBlock>
        <appender-ref ref="AUDIT_FILE"/>
    </appender>

    <appender name="AUDIT_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>/var/log/a2a/audit.log</file>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>/var/log/a2a/audit-%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
    </appender>

    <logger name="A2A_AUDIT" level="INFO" additivity="false">
        <appender-ref ref="ASYNC_AUDIT"/>
    </logger>
</configuration>
```

---
## 12. 审计日志设计

- **输出**：JSON Lines 文件，由ELK/Loki采集。
- **可靠性**：`AsyncAppender` + `neverBlock=false`，队列满时阻塞虚拟线程，确保日志不丢失。
- **内容**：每条审计日志包含以下字段：

```json
{
  "@timestamp": "2026-06-10T10:00:00.123Z",
  "service_type": "AGENT",
  "trace_id": "abc-123-def",
  "jti": "jwt-id-xxx",
  "decision": "ALLOW",
  "bulk_mode": false,
  "caller_agent_id": "agent-a",
  "target_agent_id": "agent-b",
  "user_id": "user-123",
  "scope": "doc:read",
  "deny_reason": null,
  "http_status": 200,
  "details": {
    "resource_id": "doc-001",
    "eval_duration_ms": 12,
    "rule_hit": "category_quick"
  }
}
```

- **轮转**：每天轮转，保留30天，自动压缩。

---
## 13. 安全加固矩阵

| 安全机制 | 实现位置 | 必须 |
|---------|----------|------|
| OAuth2 Token Exchange | 授权服务器+Agent | ✅ |
| 一次性令牌防重放 | Agent+Redis | ✅ |
| ACL前缀匹配 | 授权服务器 | ✅ |
| 分类快速通道 | 授权服务器+Agent | ✅ |
| 标签ABAC + jCasbin | Agent | ✅ |
| 多跳委托限制 | 授权服务器 | ✅ |
| 批量预授权（签名+验签） | 授权服务器+Agent+Redis | ✅ |
| 约束传递模式 | 授权服务器+Agent | ✅ |
| 工具级Scope限制 | 授权服务器+Agent | ✅ |
| Tool Calling统一切面 | Agent | ✅ |
| 请求体哈希防篡改 | RouteAgent+Agent | ✅ |
| 元数据服务统计确认 | 授权服务器+元数据服务 | ✅ |
| HITL人工确认 | Agent | 可选 |
| 凭据保险箱 | 授权服务器+Agent | ✅ |
| 审计日志阻塞写入 | Logback | ✅ |
| 策略同步(Redis Streams) | Agent+Redis | ✅ |

---
## 14. 扩展点设计

- **自定义 `PermissionCheckHandler`**：实现接口并注册为Bean，自动加入责任链。
- **自定义 `ToolResourceExtractor`**：支持复杂工具参数解析。
- **自定义限流策略**：实现 `RateLimitStrategy`。
- **自定义凭据解析器**：实现 `CredentialResolver`。
- **自定义元数据服务适配器**：实现 `MetadataServiceProvider`。