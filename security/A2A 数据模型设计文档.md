
> **适用范围**：A2A 权限系统的数据库表结构、字段含义、索引、约束及设计决策  
> **设计原则**：最小权限、高性能、可审计、可扩展  
> **约束**：不使用外键约束（应用层维护一致性），使用 `TIMESTAMPTZ` 统一时区  

---
## 1. 概述

本文档定义 A2A 权限系统所需的所有持久化数据模型，包括资源分类、用户权限、Agent 间 ACL、用户委派同意、客户端安全配置和审计日志。数据模型支持细粒度资源实例级授权、批量预授权、多跳委托、全链路审计等核心功能。

---
## 2. 核心表作用概览

| 表名                          | 作用                                             |
| --------------------------- | ---------------------------------------------- |
| `resource_category`         | 定义资源分类（如财务文档、技术文档），为批量授权提供分组                   |
| `resource_category_mapping` | 将资源实例关联到资源分类（多对多）                              |
| `user_category_permission`  | 用户对资源分类的批量授权（支持操作粒度和通配符）                       |
| `user_resource_permission`  | 用户对特定资源实例的特例授权（覆盖分类权限），支持 ALLOW/DENY           |
| `a2a_acl`                   | Agent 间调用访问控制，定义调用方 Agent 对目标 Agent 的 Scope 权限 |
| `user_consent`              | 用户委派授权，记录用户同意 Agent 代表自己执行某类操作                 |
| `oauth2_client_settings`    | Agent（OAuth2 客户端）的安全配置（信任级别、令牌类型白名单）           |
| `a2a_audit_log`             | 所有权限决策的审计日志，包含全链路追踪信息                          |

---
## 3. 表结构详细定义

### 3.1 `resource_category` — 资源分类定义

| 字段 | 类型 | 约束 | 描述 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY | 分类唯一标识 |
| `category_name` | VARCHAR(128) | NOT NULL | 分类名称，如 `financial_report` |
| `resource_type` | VARCHAR(64) | NOT NULL | 资源类型，如 `document`、`table` |
| `description` | TEXT | | 分类描述 |
| `created_at` | TIMESTAMPTZ | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| `created_by` | VARCHAR(64) | | 创建人 ID |
| UNIQUE | (category_name, resource_type) | | 分类名称在资源类型内唯一 |

**索引**：主键索引。

---
### 3.2 `resource_category_mapping` — 资源实例与分类关联

| 字段 | 类型 | 约束 | 描述 |
|------|------|------|------|
| `resource_id` | VARCHAR(128) | NOT NULL | 资源实例 ID |
| `resource_type` | VARCHAR(64) | NOT NULL | 资源类型，与 `resource_category.resource_type` 对应 |
| `category_id` | UUID | NOT NULL | 关联的分类 ID |
| `created_at` | TIMESTAMPTZ | DEFAULT CURRENT_TIMESTAMP | 映射创建时间 |
| `created_by` | VARCHAR(64) | | 创建人/系统标识 |
| PRIMARY KEY | (resource_type, resource_id, category_id) | | 复合主键 |
| INDEX | `idx_category` ON (category_id) | | 加速按分类查询资源 |

**一致性约束**（应用层）：插入时需确保 `resource_type` 与 `category_id` 对应行的 `resource_type` 一致。

---
### 3.3 `user_category_permission` — 用户对分类的批量授权

| 字段 | 类型 | 约束 | 描述 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY | 记录唯一标识 |
| `user_id` | VARCHAR(64) | NOT NULL | 用户 ID |
| `category_id` | UUID | NOT NULL | 资源分类 ID |
| `operation` | VARCHAR(32) | NOT NULL | 操作类型，`'*'` 表示该分类下所有操作 |
| `granted_at` | TIMESTAMPTZ | DEFAULT CURRENT_TIMESTAMP | 授权时间 |
| `expires_at` | TIMESTAMPTZ | NULL | 过期时间，NULL 表示永不过期 |
| UNIQUE | (user_id, category_id, operation) | | 防止重复授权 |
| INDEX | `idx_user_category` ON (user_id, category_id) | | 按用户+分类查询 |
| INDEX | `idx_category_user` ON (category_id, user_id) | | 按分类+用户查询（管理用） |

---
### 3.4 `user_resource_permission` — 用户对特定资源实例的特例授权

> 该表用于覆盖分类权限，实现资源实例级的允许或拒绝。优先级最高。

| 字段 | 类型 | 约束 | 描述 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY | |
| `user_id` | VARCHAR(64) | NOT NULL | 用户 ID |
| `resource_type` | VARCHAR(64) | NOT NULL | 资源类型，如 `document`、`table` |
| `resource_id` | VARCHAR(128) | NOT NULL | 资源实例 ID |
| `operation` | VARCHAR(32) | NOT NULL | 操作类型，`'*'` 表示所有操作 |
| `effect` | VARCHAR(8) | NOT NULL CHECK (effect IN ('ALLOW', 'DENY')) | 允许或拒绝 |
| `granted_at` | TIMESTAMPTZ | DEFAULT CURRENT_TIMESTAMP | 授权时间 |
| `revoked_at` | TIMESTAMPTZ | NULL | 撤销时间，NULL 表示有效 |
| `expires_at` | TIMESTAMPTZ | NULL | 过期时间，NULL 表示永不过期 |
| UNIQUE | (user_id, resource_type, resource_id, operation, revoked_at) | | 支持多版本历史 |
| INDEX | `idx_permission_resource` ON (resource_type, resource_id) | | 按资源查询 |
| PARTIAL UNIQUE INDEX | `idx_permission_active` ON (user_id, resource_type, resource_id, operation) WHERE revoked_at IS NULL | | 保证最多一条有效特例 |

---
### 3.5 `a2a_acl` — Agent 间调用访问控制

| 字段 | 类型 | 约束 | 描述 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY | |
| `caller_client_id` | VARCHAR(64) | NOT NULL | 调用方 Agent ID |
| `target_client_id` | VARCHAR(64) | NOT NULL | 目标 Agent ID |
| `allowed_scope_patterns` | TEXT | NOT NULL | 允许的 Scope 模式，支持 `*` 通配符，逗号分隔。单行建议不超过 4096 字符 |
| `denied_scope_patterns` | TEXT | | 拒绝的 Scope 模式，优先级高于 allowed。单行建议不超过 4096 字符 |
| `created_at` | TIMESTAMPTZ | DEFAULT CURRENT_TIMESTAMP | |
| `updated_at` | TIMESTAMPTZ | DEFAULT CURRENT_TIMESTAMP | 应用层在更新时需手动设置此字段 |
| UNIQUE | (caller_client_id, target_client_id) | | 每对调用关系唯一 |
| INDEX | `idx_caller_target` ON (caller_client_id, target_client_id) | | |

**Scope 模式匹配规则**：
- 支持前缀匹配（如 `doc:` 匹配 `doc:read`、`doc:summarize`）
- 支持通配符 `*`（如 `doc:read:*` 匹配 `doc:read:report-123`）
- 算法：将模式按 `:` 分割，每段支持 `*`；先检查 `denied` 列表，如有匹配则拒绝；否则检查 `allowed` 列表，如有匹配则允许；否则拒绝。

---
### 3.6 `user_consent` — 用户委派授权

| 字段 | 类型 | 约束 | 描述 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY | |
| `user_id` | VARCHAR(64) | NOT NULL | 用户 ID |
| `client_id` | VARCHAR(64) | NOT NULL | 被授权的 Agent ID |
| `scope_prefix` | VARCHAR(256) | NOT NULL | 授权的 Scope 前缀（如 `doc:read`） |
| `granted_at` | TIMESTAMPTZ | DEFAULT CURRENT_TIMESTAMP | 授权时间 |
| `revoked_at` | TIMESTAMPTZ | NULL | 撤销时间，NULL 表示有效 |
| `expires_at` | TIMESTAMPTZ | NULL | 过期时间，NULL 表示永不过期 |
| `consent_context` | JSONB | | 环境上下文：`{"ip":"...", "user_agent":"...", "location":"..."}` |
| UNIQUE | (user_id, client_id, scope_prefix, revoked_at) | | 支持多版本历史 |
| PARTIAL UNIQUE INDEX | `idx_consent_active` ON (user_id, client_id, scope_prefix) WHERE revoked_at IS NULL | | 保证最多一条有效授权 |
| INDEX | `idx_consent_user` ON (user_id, revoked_at) | | 查询用户的有效授权 |
| INDEX | `idx_consent_client` ON (client_id, revoked_at) | | 查询某 Agent 的授权 |
| INDEX | `idx_consent_expires` ON (expires_at) | | 用于清理过期记录 |

**注意**：`consent_context` 仅存储环境信息，不存储资源分类或资源 ID；资源实例权限由 `user_resource_permission` 和 `user_category_permission` 管理。

---
### 3.7 `oauth2_client_settings` — 客户端安全配置

| 字段 | 类型 | 约束 | 描述 |
|------|------|------|------|
| `client_id` | VARCHAR(64) | PRIMARY KEY | Agent 的 client_id |
| `allowed_subject_token_types` | TEXT | | JSON 数组，如 `["access_token","jwt"]` |
| `trust_level` | VARCHAR(16) | DEFAULT 'INTERNAL' CHECK (trust_level IN ('INTERNAL', 'PARTNER', 'PUBLIC')) | 信任级别 |
| `created_at` | TIMESTAMPTZ | DEFAULT CURRENT_TIMESTAMP | |
| `updated_at` | TIMESTAMPTZ | DEFAULT CURRENT_TIMESTAMP | 应用层手动更新 |

**信任级别语义**：
- `INTERNAL`：完全信任，可跳过一次性令牌等额外检查
- `PARTNER`：标准安全检查
- `PUBLIC`：最严格，要求 DPoP 或 mTLS 等增强认证

---
### 3.8 `a2a_audit_log` — 审计日志

| 字段 | 类型 | 约束 | 描述 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY | |
| `timestamp` | TIMESTAMPTZ | NOT NULL DEFAULT CURRENT_TIMESTAMP | |
| `service_type` | VARCHAR(16) | NOT NULL | `AUTHZ` 或 `AGENT` |
| `caller_agent_id` | VARCHAR(64) | | 调用方 Agent ID |
| `target_agent_id` | VARCHAR(64) | | 目标 Agent ID |
| `user_id` | VARCHAR(64) | | 最终用户 ID（委派场景） |
| `scope` | VARCHAR(256) | | 请求的 Scope |
| `trace_id` | VARCHAR(64) | NOT NULL | 全链路追踪 ID |
| `jti` | VARCHAR(64) | | JWT ID |
| `decision` | VARCHAR(16) | NOT NULL | `ALLOW` 或 `DENY` |
| `deny_reason` | VARCHAR(64) | | 拒绝原因（如 `ACL_MISMATCH`、`TOKEN_EXPIRED`） |
| `http_status` | INT | | HTTP 状态码 |
| `bulk_mode` | BOOLEAN | DEFAULT FALSE | 是否批量授权模式 |
| `bulk_partial` | BOOLEAN | DEFAULT FALSE | 批量模式下是否部分拒绝 |
| `details` | JSONB | | 结构化附加信息，遵循标准 schema |
| INDEX | `idx_trace_id` ON (trace_id) | | |
| INDEX | `idx_jti` ON (jti) | | |
| INDEX | `idx_timestamp` ON (timestamp) | | |

**`details` 标准 JSON Schema**：
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "client_ip": { "type": "string" },
    "user_agent": { "type": "string" },
    "jti": { "type": "string" },
    "request_params": {
      "type": "object",
      "additionalProperties": true,
      "description": "脱敏后的请求参数（不含 client_secret, subject_token）"
    }
  },
  "required": ["client_ip"]
}
```

**防篡改机制**：不在数据库内维护哈希链，而是将审计日志同步到外部不可变存储（如 AWS S3 Object Lock、WAL 归档），定期进行哈希校验。

---
## 4. 权限优先级规则（应用层实现）

1. **特例授权**（`user_resource_permission`）  
   - 查询条件：`user_id = ? AND resource_type = ? AND resource_id = ? AND operation IN (?, '*') AND effect = 'DENY' AND revoked_at IS NULL AND (expires_at IS NULL OR expires_at > NOW())`  
   - 存在则直接拒绝。
   - 同理查询 `effect = 'ALLOW'`，存在则直接允许。

2. **分类批量授权**（`user_category_permission`）  
   - 获取资源所属分类列表（`resource_category_mapping`）  
   - 对每个分类检查：`user_id = ? AND category_id = ? AND (operation = ? OR operation = '*') AND (expires_at IS NULL OR expires_at > NOW())`  
   - 任一匹配则允许。若存在 `operation='*'` 记录，视为该分类下所有操作均已授权，无需检查具体操作。

3. **Agent 间 ACL**（`a2a_acl`）  
   - 仅用于 Token Exchange 时的 Agent 授权评估，不涉及用户资源。
   - 先检查 `denied_scope_patterns`，再检查 `allowed_scope_patterns`。

4. **默认拒绝**。

---
## 5. 数据一致性保障（无外键约束）

应用层负责维护数据完整性，所有跨表操作应在同一数据库事务中完成：

- **删除资源时**：删除 `resource_category_mapping` 中 `(resource_type, resource_id)` 的行，同时删除 `user_resource_permission` 中对应行。
- **删除分类时**：删除 `resource_category_mapping` 中 `category_id` 的行，同时删除 `user_category_permission` 中 `category_id` 的行。
- **删除客户端（Agent）时**：删除 `a2a_acl` 中涉及该 `client_id` 的行、`user_consent` 中 `client_id` 的行，以及 `oauth2_client_settings` 中对应的行。
- **插入 `resource_category_mapping` 时**：校验 `resource_type` 与 `category_id` 关联的分类表中的 `resource_type` 一致。
- **更新 `updated_at` 字段**：每次更新行时，应用层应显式设置 `updated_at = NOW()`（或使用数据库触发器）。

---
## 6. 性能优化建议

- 对 `resource_category_mapping` 的 `(resource_type, resource_id)` 查询使用主键前缀索引，性能良好。
- 高频访问的资源，可在应用层缓存其所属分类列表（Caffeine 本地缓存，TTL 5 分钟）。
- `user_category_permission` 查询通过 `(user_id, category_id)` 索引加速。
- 分类数量通常较少（< 100），权限判断循环开销可忽略。
- 审计日志建议使用分区表（按月）和异步写入，避免影响主业务。

---
## 7. 迁移策略

- 使用 **Flyway** 或 **Liquibase** 管理数据库 schema 变更，所有迁移脚本存放在 `db/migration` 目录。
- 命名规范：`V{版本号}__{描述}.sql`，例如 `V1__initial_schema.sql`。
- **向后兼容原则**：禁止删除列或修改已有列类型；新加列需有默认值或允许 NULL；索引添加使用 `CONCURRENTLY` 避免锁表。
- 从旧模型（单表 `user_resource_permission`）迁移到新模型（分类+特例）时，应用层双写过渡，最后弃用旧表。

---
## 8. 附录：超大 JWT 声明的可选方案

如确需在 JWT 中携带超过 300 个资源 ID（原则上建议避免），可采用 Redis 缓存方案：
1. 授权服务器生成唯一 key（如 `jwt:claim:{uuid}`），将大数据存入 Redis，TTL 等于 JWT 有效期。
2. JWT 中携带 `claim_ref: { "key": "...", "hash": "sha256(...)" }`。
3. 资源服务器根据 `claim_ref` 从 Redis 读取数据，并校验哈希。

此方案会增加网络依赖，仅在必要时启用。

---
## 9. 总结

本数据模型设计文档完整定义了 A2A 权限系统所需的 8 张核心表，涵盖了资源分类、批量授权、特例覆盖、Agent 间 ACL、用户委派同意、客户端安全配置和审计日志。所有字段、索引、约束、优先级规则和一致性保障均已明确，可直接用于生产环境数据库实施。