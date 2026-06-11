
> 本方案基于 **用户 → 角色 → 权限（菜单+操作+行级规则）** 的极简模型，无数据域、无中间表、无派生表。  
> 权限直接定义在 `t_permission` 表中，包含行级规则模板（JSON）。  
> AI Agent 对话时，系统根据用户已有权限与用户意图，动态筛选所需权限，低风险自动批准，高风险触发人工确认。  

---
## 一、设计目标与核心原则

### 1.1 目标
- **极简模型**：仅保留用户、角色、权限三张核心表（均带 `t_` 前缀），权限直接绑定菜单、操作、行级规则。
- **动态授权**：AI Agent 根据用户对话意图，自动匹配所需权限，低风险放行，高风险需用户确认。
- **数据安全**：行级规则通过 SQL 片段附加在查询上，确保用户只能访问授权范围内的数据行。
- **全链路审计**：记录每一次权限判断与用户确认过程。
- **A2A 协同支持**：Agent 间调用可通过 JWT 传递权限声明，无需依赖用户角色。

### 1.2 核心原则
- **默认拒绝**：任何未显式授权的操作或数据行禁止访问。
- **最小动态授权**：每次对话仅激活用户权限中的必要子集，避免权限滥用。
- **人工确认兜底**：高风险操作必须由用户实时确认，AI 不得自行决定。
- **可审计性**：所有权限决策和用户确认动作均写入审计日志。

---
## 二、核心概念定义

| 概念 | 定义 | 示例 |
|------|------|------|
| **权限** | 一个包含菜单、操作、行级规则模板的单元 | `order_read`：菜单=`order_menu`，操作=`READ`，行级规则=`{"orders":"salesperson_id = #{user.id}"}` |
| **角色** | 一组权限的集合 | `SALES_REP` 拥有 `order_read`、`order_export` |
| **用户** | 系统使用者，可关联多个角色 | `zhang.san` 关联 `SALES_REP`、`REGION_MANAGER` |
| **对话上下文** | 自然语言输入解析出的操作意图 | “查询我上个月的订单” → 对 `order_menu` 执行 `READ` |
| **风险等级** | 权限操作的风险程度，由管理员预定义 | `READ` 低风险，`DELETE` 高风险 |

---
## 三、数据模型设计（极简 DDL，带前缀 t_）

```sql
-- 用户表
CREATE TABLE t_user (
    user_id     VARCHAR(64) PRIMARY KEY,
    username    VARCHAR(64) NOT NULL UNIQUE,
    department  VARCHAR(128),
    created_at  TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6)
);

-- 角色表
CREATE TABLE t_role (
    role_id     VARCHAR(32) PRIMARY KEY,
    role_name   VARCHAR(64) NOT NULL
);

-- 用户-角色关联
CREATE TABLE t_user_role (
    user_id     VARCHAR(64) NOT NULL,
    role_id     VARCHAR(32) NOT NULL,
    PRIMARY KEY (user_id, role_id),
    FOREIGN KEY (user_id) REFERENCES t_user(user_id),
    FOREIGN KEY (role_id) REFERENCES t_role(role_id)
);

-- 权限表（直接包含菜单、操作、行级规则模板）
CREATE TABLE t_permission (
    permission_id     VARCHAR(32) PRIMARY KEY,
    permission_name   VARCHAR(128) NOT NULL,
    menu_id           VARCHAR(32) NOT NULL COMMENT '菜单标识',
    action_code       VARCHAR(32) NOT NULL COMMENT '操作枚举: READ, CREATE, UPDATE, DELETE, EXPORT',
    risk_level        VARCHAR(16) NOT NULL DEFAULT 'LOW' COMMENT 'LOW, MEDIUM, HIGH',
    row_rule_template JSON COMMENT '行级规则模板，如 {"orders":"salesperson_id = #{user.id}"}',
    description       TEXT,
    created_at        TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6)
);

-- 角色-权限关联
CREATE TABLE t_role_permission (
    role_id       VARCHAR(32) NOT NULL,
    permission_id VARCHAR(32) NOT NULL,
    PRIMARY KEY (role_id, permission_id),
    FOREIGN KEY (role_id) REFERENCES t_role(role_id),
    FOREIGN KEY (permission_id) REFERENCES t_permission(permission_id)
);
```

### 操作枚举与风险等级（Java 代码）

```java
public enum Action {
    READ("READ", 1 << 0, "SELECT", "LOW"),
    CREATE("CREATE", 1 << 1, "INSERT", "MEDIUM"),
    UPDATE("UPDATE", 1 << 2, "UPDATE", "MEDIUM"),
    DELETE("DELETE", 1 << 3, "DELETE", "HIGH"),
    EXPORT("EXPORT", 1 << 0, "SELECT", "HIGH");

    private final String code;
    private final int bitMask;
    private final String dbOperation;
    private final String defaultRisk;
    // constructor, getters...
}
```

### 行级规则模板格式

支持两种格式：

**格式一（多表）**：
```json
{
  "orders": "salesperson_id = #{user.id}",
  "order_items": "order_id IN (SELECT id FROM orders WHERE salesperson_id = #{user.id})"
}
```

**格式二（单表简化）**：
```json
{
  "table": "orders",
  "condition": "region_id = #{user.regionId}"
}
```

> 模板中的 `#{user.xxx}` 会在运行时从当前用户上下文中取值并替换为参数占位符。所有模板内容必须经过安全校验（禁止危险 SQL 关键字）。

---
## 四、菜单与表的映射（配置化，不建表）

配置文件 `application.yml`：
```yaml
a2a:
  menu-table-mapping:
    order_menu:
      - orders
      - order_items
    customer_menu:
      - customers
```

也可使用 Java 枚举/常量。此映射关系用于将菜单操作转换为具体表的权限校验。

---
## 五、运行时权限获取与筛选（核心逻辑）

### 5.1 获取用户完整权限集合

```java
public Set<Permission> getUserPermissions(String userId) {
    // 查询 t_user_role + t_role_permission + t_permission
    // 结果可本地缓存（Caffeine, TTL 5分钟），权限变更时清除对应角色缓存
    return permissionMapper.selectByUserId(userId);
}
```

### 5.2 对话意图解析

AI Agent 调用 NLU 模块或 LLM 提取意图，得到：
```json
{
  "intent": {
    "menuId": "order_menu",
    "action": "READ",
    "parameters": {"timeRange": "last_month"}
  }
}
```

### 5.3 筛选本次对话需要的权限

从用户完整权限中匹配 `menu_id` 和 `action_code`。若匹配到多个权限（例如两个不同权限都包含 `order_menu:READ`），则合并其行级规则模板：
- 对同一张表的多条规则，使用 `OR` 连接。
- 对不同表的规则，分别处理。

```java
List<Permission> required = userPermissions.stream()
    .filter(p -> p.getMenuId().equals(intent.getMenuId()) && p.getActionCode().equals(intent.getAction()))
    .collect(Collectors.toList());
```

### 5.4 风险判断与授权决策

```java
if (permission.getRiskLevel().equals("HIGH")) {
    // 必须用户确认
    return requestUserConfirmation(permission, intent);
} else if (permission.getRiskLevel().equals("MEDIUM") && requireMediumConfirm) {
    return requestUserConfirmation(permission, intent);
} else {
    return autoApprove(permission, intent);
}
```

### 5.5 人工确认流程（Human-in-the-Loop）

1. Agent 向用户发送确认请求（通过 WebSocket 或消息队列），展示：
   - 将要执行的操作描述（如“查询上个月的订单”）
   - 涉及的数据范围（如“仅限您负责的华东区域”）
   - 风险提示（如“此操作将导出数据，请注意安全”）
2. 用户选择“同意”或“拒绝”。
3. 若同意，Agent 继续执行；若拒绝，终止本次操作并记录。
4. 确认结果可设置会话级别缓存（如 5 分钟内相同权限自动通过）。
5. **缓存失效条件**：用户登出、角色变更、权限变更时，系统需广播事件清除该用户的确认缓存。

### 5.6 Agent 自身权限（A2A 调用场景）

当 Agent 调用另一个 Agent 时，上游 Agent 可在 JWT 中传递 `allowed_permission_ids`，下游 Agent 优先使用该列表作为授权依据，不再查询用户角色。此机制用于支持无人值守的自动化任务。

```json
{
  "allowed_permission_ids": ["order_read", "order_export"]
}
```

---
## 六、权限执行层（SQL 拦截）

### 6.1 表级权限校验

MyBatis 拦截器解析 SQL 中的表名，与用户权限关联的表集合（通过菜单-表映射获得）比对，确保用户有对应表的操作权限。  
**操作类型判定**：优先从 `SecurityContext` 中获取（由 `@PreAuthorize` 或意图解析传入）；若为空，则根据 `MappedStatement.getSqlCommandType()` 推断（SELECT→READ，INSERT→CREATE，UPDATE→UPDATE，DELETE→DELETE）。推断仅用于降级兼容，生产环境建议显式传递。

```java
SqlCommandType cmdType = ms.getSqlCommandType();
Action requiredAction = mapSqlCommandToAction(cmdType);
```

### 6.2 行级规则注入

根据权限的 `row_rule_template`，为 SQL 动态附加 WHERE 条件。  
- 若匹配到多个权限，合并规则：同一表的多条条件用 `OR` 连接，不同表之间用 `AND` 连接。
- 模板中的占位符 `#{user.xxx}` 替换为预编译参数（`?`），并绑定用户属性值。
- 必须校验模板内容不含危险关键字（如 `UNION`, `DROP`, `--` 等）。

示例：模板 `{"orders": "salesperson_id = #{user.id}"}`，原始 SQL `SELECT * FROM orders` 注入后为 `SELECT * FROM orders WHERE (salesperson_id = ?)`。

### 6.3 参数化处理

使用 MyBatis 的 `DynamicContext` 或自定义 `ParameterHandler` 将用户属性添加到参数映射中，确保使用预编译语句。

---
## 七、API 层权限校验（传统 REST API 可选）

```java
@PreAuthorize("@permEvaluator.hasPermission('order_menu', 'UPDATE')")
@PutMapping("/orders/{id}")
public Result updateOrder(...) { ... }
```

`hasPermission` 直接查询用户权限集合（与对话筛选逻辑共享缓存），不依赖数据域。

---
## 八、全链路记录与审计

审计日志 JSON 示例：
```json
{
  "timestamp": "2026-06-11T10:00:00Z",
  "trace_id": "abc123",
  "user_id": "zhang.san",
  "session_id": "sess_456",
  "intent": {"menuId": "order_menu", "action": "READ"},
  "matched_permissions": ["order_read"],
  "decision": "AUTO_APPROVE",
  "human_confirmed": false,
  "risk_level": "LOW",
  "sql_before": "SELECT * FROM orders",
  "sql_after": "SELECT * FROM orders WHERE salesperson_id = ?",
  "deny_reason": null
}
```

日志异步写入，不阻塞主流程。

---
## 九、配置与扩展

```yaml
a2a:
  security:
    risk:
      low-auto-approve: true
      medium-require-confirm: true
      confirm-ttl-seconds: 300
    cache:
      permission-ttl-seconds: 300
      max-size: 1000
```

支持自定义 `RowRuleRenderer` 接口，用于从非 `SecurityContext` 来源（如 JWT 属性、请求头）取值。

---
## 十、部署检查清单

- [ ] 执行 DDL 创建 `t_user`, `t_role`, `t_user_role`, `t_permission`, `t_role_permission`。
- [ ] 初始化权限数据（含 `row_rule_template`）。
- [ ] 配置菜单-表映射（YAML 或代码）。
- [ ] 实现意图解析与权限筛选逻辑。
- [ ] 实现人工确认 WebSocket 端点及缓存失效广播。
- [ ] 配置 MyBatis 拦截器进行表级和行级校验。
- [ ] 实现审计日志与 OpenTelemetry。
- [ ] 配置 A2A 场景下 JWT 传递 `allowed_permission_ids` 的验证逻辑。

---
## 十一、总结

本方案完全遵循要求：
- 所有表名前缀为 `t_`。
- 无数据域、无中间表、无派生表。
- 行级规则模板直接挂在 `t_permission` 表 JSON 字段。
- AI Agent 根据用户意图动态筛选权限，低风险自动批准，高风险需人工确认。
- 支持 A2A 协同（JWT 传递权限声明）。
- 模型极简，易于实现和审计，适合企业级 AI Agent 协同与自然语言查询场景。