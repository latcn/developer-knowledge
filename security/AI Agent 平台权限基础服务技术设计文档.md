> **最终版本**：V8.7 | **技术栈**：JDK 21 + Spring Boot 3.5.3 + MyBatis 3.x + Redis 7.x + MySQL 8.0 + Spring Cloud OpenFeign + RocketMQ 4.9+

## 目录
1. [文档概述](#1-文档概述)
2. [核心设计原则](#2-核心设计原则)
3. [整体架构设计](#3-整体架构设计)
4. [模块划分与依赖关系](#4-模块划分与依赖关系)
5. [数据模型设计](#5-数据模型设计)
6. [权限计算引擎](#6-权限计算引擎)
7. [行级规则引擎](#7-行级规则引擎)
8. [智能数据访问层（Client）](#8-智能数据访问层client)
9. [权限管理操作与审计](#9-权限管理操作与审计)
10. [缓存与消息机制](#10-缓存与消息机制)
11. [权限版本号机制](#11-权限版本号机制)
12. [部署与配置](#12-部署与配置)
13. [附录](#13-附录)

## 1. 文档概述

### 1.1 文档定位

本文档定义 AI Agent 平台**权限基础服务**的完整技术设计方案，涵盖数据模型、模块划分、核心接口、缓存机制、部署配置等全方位内容。权限基础服务作为独立的微服务，提供权限数据的**写入、计算、缓存和查询**能力，供上层认证服务和各 Agent 节点调用。

### 1.2 核心能力

| 能力域                     | 说明                                                       |
| ----------------------- | -------------------------------------------------------- |
| **功能权限管理**              | 用户能否执行某个操作（如 `order:export`）的规则定义与存储                     |
| **数据权限管理**              | 用户能操作哪些数据行的规则定义（通过行级规则实现）                                |
| **角色与权限绑定**             | 角色定义、角色优先级、角色-权限关联（含行级规则）                                |
| **A2A 授权数据**            | Agent 注册信息、A2A ACL 白名单的存储与查询                             |
| **Token Exchange 数据聚合** | 一次调用返回 Token Exchange 所需的所有权限数据                          |
| **权限版本管理**              | 权限变更时递增版本号，支持上层服务实现 Token 实时失效                           |
| **操作审计追溯**              | 通过 `t_audit_log` 记录所有权限管理操作的完整审计轨迹                       |
| **版本变更追溯**              | 通过 `t_permission_version_history` 追溯 `perm_version` 变化原因 |
| **缓存与消息**               | 多级缓存（Caffeine + Redis）+ RocketMQ 异步解耦                    |


## 2. 核心设计原则

| 原则 | 说明 |
|------|------|
| **物理隔离** | `local-client` 和 `remote-client` 分属不同模块，上层调用方**永远不引入** `local-client`，编译期杜绝误用本地 DB 调用的可能 |
| **读写分离** | 控制面负责写入和计算，数据面负责读取和缓存 |
| **智能客户端** | Client 同时支持 Local（直连 DB）和 Remote（Feign 调用）两种模式 |
| **缓存优先** | 所有读取操作优先走多级缓存（Caffeine → Redis） |
| **最终一致性** | 权限变更通过 RocketMQ 广播，数据面缓存异步刷新 |
| **版本号驱动** | 通过 `perm_ver` 实现权限实时吊销通知 |
| **操作全审计** | 所有权限管理操作（无论成功/失败/权限是否变化）均记录审计日志 |
| **版本精准追溯** | 仅在用户最终权限集合实际变化时记录版本历史 |
| **接口契约纯净** | `a2a-permission-api` 不包含任何 Feign/Web 注解 |
| **消息统一** | 所有异步消息（权限变更通知）统一使用 RocketMQ |


## 3. 整体架构设计

### 3.1 部署架构

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          权限基础服务部署架构                                        │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │              上层调用方（认证服务 / Agent 节点）                            │   │
│  │  ┌─────────────────────────────────────────────────────────────────────┐   │   │
│  │  │  依赖：                                                             │   │   │
│  │  │  ├── a2a-permission-api (接口契约)                                  │   │   │
│  │  │  ├── a2a-permission-common-client (通用缓存)                        │   │   │
│  │  │  └── a2a-permission-remote-client (Feign 远程调用 + MQ 订阅)        │   │   │
│  │  │  ❌ 不引入 local-client（无 MySQL 依赖）                           │   │   │
│  │  │  配置：mode=remote                                                  │   │   │
│  │  └─────────────────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                             │
│                                      │ Feign 调用（通过 remote-client）            │
│                                      ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │              权限基础服务 - 控制面 (a2a-permission-service)                 │   │
│  │  ┌─────────────────────────────────────────────────────────────────────┐   │   │
│  │  │  依赖：                                                             │   │   │
│  │  │  ├── a2a-permission-api (DTO + 接口)                               │   │   │
│  │  │  ├── a2a-permission-common-client (通用缓存)                        │   │   │
│  │  │  └── a2a-permission-local-client (直连 MySQL)                      │   │   │
│  │  │  ❌ 不引入 remote-client（不需要远程调用）                         │   │   │
│  │  │  配置：mode=local                                                   │   │   │
│  │  │  职责：数据写入、权限计算、Redis 更新、发送 RocketMQ 消息           │   │   │
│  │  └─────────────────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                             │
│                                      ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                        Redis + MySQL + RocketMQ                            │   │
│  │  ┌───────────────────────────────────────────────────────────────────────┐ │   │
│  │  │  MySQL: 权限数据 + 操作审计日志 + 版本历史 (仅控制面访问)             │ │   │
│  │  │  Redis: 权限缓存 + 版本号                                            │ │   │
│  │  │  RocketMQ: Topic 权限变更通知 (控制面 → 数据面)                     │ │   │
│  │  └───────────────────────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 模块物理结构

```
a2a-permission/
├── a2a-permission-api/                         # 【契约层 - 纯接口JAR】
│   ├── model/                                  # DTO 定义
│   │   ├── TokenExchangePrepareRequest.java
│   │   ├── TokenExchangePrepareResponse.java
│   │   ├── UserFullPermissionDTO.java
│   │   ├── AgentDTO.java
│   │   ├── AclCheckResult.java
│   │   └── AuditLogDTO.java                   # 审计日志 DTO
│   └── spi/                                    # SPI 接口定义
│       └── PermissionQueryService.java         # 纯 Java 接口，无 Feign 注解
│
├── a2a-permission-common-client/               # 【通用缓存层 - 嵌入式JAR】
│   ├── cache/                                  # 缓存装饰器
│   │   └── CachedPermissionQueryService.java
│   ├── redis/                                  # Redis 操作封装
│   │   └── RedisCacheManager.java
│   └── config/
│       └── CommonClientAutoConfig.java
│
├── a2a-permission-local-client/                # 【本地实现 - 嵌入式JAR】
│   ├── local/
│   │   └── LocalPermissionQueryService.java
│   └── config/
│       └── LocalClientAutoConfig.java
│
├── a2a-permission-remote-client/               # 【远程实现 - 嵌入式JAR】
│   ├── remote/
│   │   └── RemotePermissionQueryService.java
│   ├── subscriber/                             # RocketMQ 订阅者
│   │   └── PermissionChangeSubscriber.java
│   └── config/
│       └── RemoteClientAutoConfig.java
│
└── a2a-permission-service/                     # 【控制面 - 独立微服务】
    ├── controller/                             # REST API
    │   └── PermissionController.java
    ├── service/                                # 业务逻辑
    │   ├── PermissionAdminService.java         # 增删改操作（含审计+版本）
    │   └── PermissionQueryServiceImpl.java     # 查询服务实现
    ├── engine/                                 # 权限计算引擎
    │   ├── FlattenEngine.java
    │   └── RowRuleMerger.java
    ├── producer/                               # RocketMQ 消息发送
    │   └── PermissionChangeProducer.java
    └── config/
        └── PermissionServiceConfig.java
```


## 4. 模块划分与依赖关系

### 4.1 依赖关系图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         模块依赖关系图                                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │              a2a-permission-api (纯契约层)                                   │   │
│  │  依赖：无  │  包含：PermissionQueryService 接口 + DTO + 常量               │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│              ▲                                                                      │
│              │ 依赖                                                                 │
│  ┌───────────┴─────────────────────────────────────────────────────────────────────┐│
│  │              a2a-permission-common-client (通用缓存层)                         ││
│  │  依赖：api + Redis + Caffeine  │  特点：无 MySQL/Feign/MQ 依赖               ││
│  └─────────────────────────────────────────────────────────────────────────────┘│
│              ▲                                   ▲                                │
│              │ 依赖                              │ 依赖                            │
│  ┌───────────┴─────────────────────────────┐  ┌─┴────────────────────────────────┐│
│  │  a2a-permission-local-client            │  │  a2a-permission-remote-client    ││
│  │  依赖：common + MyBatis + MySQL         │  │  依赖：common + Feign + MQ       ││
│  │  提供：LocalPermissionQueryService      │  │  提供：RemotePermissionQuery +   ││
│  │  使用方：仅控制面                       │  │        PermissionChangeSubscriber││
│  │  消息：❌ 不订阅                       │  │  使用方：上层调用方               ││
│  └─────────────────────────────────────────┘  └──────────────────────────────────┘│
│              │                                   ▲                                │
│              ▼                                   │                                │
│  ┌────────────────────────────────────────────────┴───────────────────────────────┐│
│  │  a2a-permission-service（控制面）              │  上层调用方（认证/Agent）      ││
│  │  ✅ 引入 local-client + MQ Client            │  ✅ 引入 remote-client          ││
│  │  ❌ 不引入 remote-client                      │  ❌ 不引入 local-client         ││
│  │  发送 MQ：✅ 是  │  接收 MQ：❌ 否          │  发送 MQ：❌ 否  │  接收 MQ：✅ ││
│  └─────────────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────────────┘
```


## 5. 数据模型设计

### 5.1 完整 DDL

```sql
-- ============================================================
-- 1. 部门表 (t_department)
-- ============================================================
CREATE TABLE t_department (
    id           BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '部门主键ID',
    dept_name    VARCHAR(128) NOT NULL UNIQUE COMMENT '部门名称',
    parent_id    BIGINT COMMENT '父部门ID，关联本表 id',
    status       TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1-有效, 0-无效',
    description  TEXT COMMENT '部门描述',
    created_by   BIGINT COMMENT '创建人ID，关联 t_user.id',
    updated_by   BIGINT COMMENT '最后修改人ID，关联 t_user.id',
    created_at   TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '创建时间',
    updated_at   TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6) COMMENT '最后修改时间',
    INDEX idx_parent_id (parent_id)
) COMMENT '部门表';

-- ============================================================
-- 2. 用户表 (t_user)
-- ============================================================
CREATE TABLE t_user (
    id                       BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '用户主键ID',
    username                 VARCHAR(64) NOT NULL UNIQUE COMMENT '用户登录名',
    password_hash            VARCHAR(256) NOT NULL COMMENT 'BCrypt 哈希密码',
    password_expires_at      TIMESTAMP(6) COMMENT '密码过期时间',
    password_failed_attempts TINYINT NOT NULL DEFAULT 0 COMMENT '连续失败次数',
    password_locked_until    TIMESTAMP(6) COMMENT '锁定到期时间',
    perm_version             BIGINT NOT NULL DEFAULT 0 COMMENT '权限版本号，每次用户最终权限集合变化时递增',
    status                   TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1-正常, 2-锁定, 3-禁用, 0-已删除',
    last_login_at            TIMESTAMP(6) COMMENT '最后登录时间',
    created_by               BIGINT COMMENT '创建人ID，关联 t_user.id',
    updated_by               BIGINT COMMENT '最后修改人ID，关联 t_user.id',
    created_at               TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '注册时间',
    updated_at               TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6) COMMENT '最后更新时间',
    INDEX idx_username (username),
    INDEX idx_status (status)
) COMMENT '用户表';

-- ============================================================
-- 3. 用户-部门关联表 (t_user_department)
-- ============================================================
CREATE TABLE t_user_department (
    id           BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '关联主键ID',
    user_id      BIGINT NOT NULL COMMENT '用户ID，关联 t_user.id',
    dept_id      BIGINT NOT NULL COMMENT '部门ID，关联 t_department.id',
    created_by   BIGINT COMMENT '创建人ID，关联 t_user.id',
    created_at   TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '关联创建时间',
    UNIQUE KEY uk_user_dept (user_id, dept_id),
    INDEX idx_user_id (user_id),
    INDEX idx_dept_id (dept_id)
) COMMENT '用户-部门关联表';

-- ============================================================
-- 4. 角色表 (t_role)
-- ============================================================
CREATE TABLE t_role (
    id           BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '角色主键ID',
    role_name    VARCHAR(64) NOT NULL UNIQUE COMMENT '角色名称',
    priority     INT NOT NULL DEFAULT 0 COMMENT '角色优先级：数值越大优先级越高',
    status       TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1-有效, 0-无效',
    description  TEXT COMMENT '角色描述',
    created_by   BIGINT COMMENT '创建人ID，关联 t_user.id',
    updated_by   BIGINT COMMENT '最后修改人ID，关联 t_user.id',
    created_at   TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '创建时间',
    updated_at   TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6) COMMENT '最后修改时间',
    INDEX idx_role_name (role_name)
) COMMENT '角色表';

-- ============================================================
-- 5. 用户-角色关联表 (t_user_role) - 授权记录
-- ============================================================
CREATE TABLE t_user_role (
    id           BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '关联主键ID',
    user_id      BIGINT NOT NULL COMMENT '用户ID，关联 t_user.id',
    role_id      BIGINT NOT NULL COMMENT '角色ID，关联 t_role.id',
    created_by   BIGINT COMMENT '授权人ID，关联 t_user.id',
    created_at   TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '授权时间',
    UNIQUE KEY uk_user_role (user_id, role_id),
    INDEX idx_user_id (user_id),
    INDEX idx_role_id (role_id)
) COMMENT '用户-角色关联表（授权记录）';

-- ============================================================
-- 6. 权限定义表 (t_permission)
-- ============================================================
CREATE TABLE t_permission (
    id                           BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '权限主键ID',
    permission_code              VARCHAR(64) NOT NULL UNIQUE COMMENT '权限码，如 order:read',
    action_code                  VARCHAR(32) NOT NULL COMMENT '操作类型：READ,CREATE,UPDATE,DELETE,EXPORT,EXECUTE,APPROVE',
    risk_level                   TINYINT NOT NULL DEFAULT 0 COMMENT '风险等级：0-低, 1-中, 2-高, 3-严重',
    mandatory_row_rule_template  JSON COMMENT '强制行级规则模板（安全红线）',
    description                  TEXT COMMENT '权限说明',
    created_by                   BIGINT COMMENT '创建人ID，关联 t_user.id',
    updated_by                   BIGINT COMMENT '最后修改人ID，关联 t_user.id',
    created_at                   TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '创建时间',
    updated_at                   TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6) COMMENT '最后修改时间',
    INDEX idx_permission_code (permission_code)
) COMMENT '权限定义表';

-- ============================================================
-- 7. 角色-权限关联表 (t_role_permission) - 核心表
-- ============================================================
CREATE TABLE t_role_permission (
    id                BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '关联主键ID',
    role_id           BIGINT NOT NULL COMMENT '角色ID，关联 t_role.id',
    permission_id     BIGINT NOT NULL COMMENT '权限ID，关联 t_permission.id',
    row_rule_template JSON COMMENT '行级规则模板，同一权限码在不同角色下可配置不同规则，实现"同权不同数据"',
    created_by        BIGINT COMMENT '创建人ID，关联 t_user.id',
    updated_by        BIGINT COMMENT '最后修改人ID，关联 t_user.id',
    created_at        TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '创建时间',
    updated_at        TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6) COMMENT '最后修改时间',
    UNIQUE KEY uk_role_permission (role_id, permission_id),
    INDEX idx_permission_id (permission_id)
) COMMENT '角色-权限关联表 - 行级数据规则绑定点';

-- ============================================================
-- 8. Agent 注册表 (t_agent)
-- ============================================================
CREATE TABLE t_agent (
    id                  BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT 'Agent 主键ID',
    client_id           VARCHAR(128) NOT NULL UNIQUE COMMENT 'OAuth2 client_id',
    client_secret_hash  VARCHAR(256) NOT NULL COMMENT 'BCrypt 哈希',
    secret_expires_at   TIMESTAMP(6) NOT NULL DEFAULT (CURRENT_TIMESTAMP + INTERVAL 90 DAY) COMMENT '密钥过期时间',
    agent_name          VARCHAR(128) NOT NULL COMMENT 'Agent 名称',
    framework_type      VARCHAR(32) NOT NULL COMMENT 'agentscope / spring-ai / custom',
    agent_card_url      VARCHAR(512) COMMENT 'Agent Card 地址（可选）',
    public_key          TEXT COMMENT 'Agent 公钥（PEM）',
    capabilities        JSON COMMENT '能力声明，值为 t_permission.permission_code 集合',
    status              TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1-正常, 2-暂停, 3-已注销',
    created_by          BIGINT COMMENT '创建人ID，关联 t_user.id',
    updated_by          BIGINT COMMENT '最后修改人ID，关联 t_user.id',
    created_at          TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '注册时间',
    updated_at          TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6) COMMENT '最后修改时间',
    INDEX idx_client_id (client_id)
) COMMENT 'Agent 注册表';

-- ============================================================
-- 9. A2A ACL 表 (t_a2a_acl)
-- ============================================================
CREATE TABLE t_a2a_acl (
    id                BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '关联主键ID',
    source_client_id  VARCHAR(128) NOT NULL COMMENT '调用方 client_id',
    target_client_id  VARCHAR(128) NOT NULL COMMENT '目标 client_id',
    allowed_scopes    JSON COMMENT '允许的 Scope 列表，是 capabilities 的子集。NULL 表示全部',
    created_by        BIGINT COMMENT '创建人ID，关联 t_user.id',
    updated_by        BIGINT COMMENT '最后修改人ID，关联 t_user.id',
    created_at        TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '创建时间',
    updated_at        TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6) COMMENT '最后修改时间',
    UNIQUE KEY uk_acl (source_client_id, target_client_id)
) COMMENT 'A2A ACL 白名单表';

-- ============================================================
-- 10. 操作审计表 (t_audit_log)
--     职责：记录所有权限管理操作的完整审计轨迹
--     记录时机：所有权限管理操作（无论权限是否变化，无论成功或失败）
--     记录内容：谁、何时、做了什么操作、结果如何、变更详情
-- ============================================================
CREATE TABLE t_audit_log (
    id                   BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '审计日志主键ID',
    trace_id             VARCHAR(64) COMMENT '全链路 Trace ID，用于跨服务调用关联',
    operation_type       VARCHAR(32) NOT NULL COMMENT '操作类型：ROLE_GRANT, ROLE_REVOKE, PERM_GRANT, PERM_REVOKE, ROLE_CREATE, ROLE_UPDATE, ROLE_DELETE, PERM_CREATE, PERM_UPDATE, PERM_DELETE, ROW_RULE_UPDATE, ACL_CREATE, ACL_UPDATE, ACL_DELETE, AGENT_REGISTER, AGENT_UPDATE, AGENT_SUSPEND, AGENT_ACTIVATE, AGENT_SECRET_ROTATE',
    operator_id          BIGINT COMMENT '操作人ID，关联 t_user.id',
    target_user_id       BIGINT COMMENT '目标用户ID（授权/撤销时）',
    target_role_id       BIGINT COMMENT '目标角色ID',
    target_permission_id BIGINT COMMENT '目标权限ID',
    target_client_id     VARCHAR(128) COMMENT '目标 Agent client_id（Agent/ACL 操作时）',
    operation_detail     JSON NOT NULL COMMENT '操作详情：SUCCESS 时存储变更前后数据；FAILED 时存储 {"reason":"错误信息"}；SKIPPED 时存储 {"reason":"跳过原因"}',
    operation_result     VARCHAR(16) NOT NULL COMMENT '操作结果：SUCCESS, FAILED, SKIPPED',
    client_ip            VARCHAR(64) COMMENT '客户端 IP',
    user_agent           VARCHAR(256) COMMENT 'User-Agent',
    created_at           TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '操作时间（数据库自动填充）',
    INDEX idx_operator_id (operator_id),
    INDEX idx_target_user_id (target_user_id),
    INDEX idx_operation_type (operation_type),
    INDEX idx_created_at (created_at),
    INDEX idx_trace_id (trace_id)
) COMMENT '操作审计表（记录所有权限管理操作，无论权限是否变化）';

-- ============================================================
-- 11. 权限版本变更历史表 (t_permission_version_history)
--     职责：追溯 perm_version 变化原因
--     记录时机：仅在用户最终权限集合变化时记录
--     记录内容：版本号变化、触发操作、关联的审计日志ID、权限快照
-- ============================================================
CREATE TABLE t_permission_version_history (
    id                   BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '历史记录主键ID',
    user_id              BIGINT NOT NULL COMMENT '用户ID，关联 t_user.id',
    old_version          BIGINT COMMENT '变更前的版本号',
    new_version          BIGINT NOT NULL COMMENT '变更后的版本号',
    trigger_operation    VARCHAR(32) COMMENT '触发操作类型（关联 t_audit_log.operation_type）',
    audit_log_id         BIGINT COMMENT '关联的审计日志ID（关联 t_audit_log.id），用于追溯触发此次版本变更的具体操作',
    affected_permissions JSON COMMENT '变更后的权限码列表（快照）。当权限码 > 1000 时，仅存储变更的增量 diff 而非全量，格式 {"added":["perm:new"],"removed":["perm:old"]}',
    created_at           TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '变更时间（数据库自动填充）',
    INDEX idx_user_id (user_id),
    INDEX idx_audit_log_id (audit_log_id),
    INDEX idx_created_at (created_at)
) COMMENT '权限版本变更历史表（仅记录权限变化时）';
```

### 5.2 两表职责对照

| 维度 | `t_audit_log`（操作审计） | `t_permission_version_history`（版本追溯） |
|------|---------------------------|-------------------------------------------|
| **职责** | 记录所有权限管理操作的审计轨迹 | 追溯 `perm_version` 变化原因 |
| **记录时机** | 所有权限管理操作（含失败、重复、跳过） | 仅用户最终权限集合**实际变化**时 |
| **记录内容** | 操作人、操作类型、操作对象、结果、变更详情 | 版本号变化、触发操作、权限快照 |
| **作用** | 合规审计、异常追溯、操作人追溯 | Token 失效判断、版本回溯 |
| **关联关系** | 独立存在 | 通过 `audit_log_id` 关联 `t_audit_log` |

### 5.3 两表关系图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    版本历史表 vs 操作审计表                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  操作：管理员给用户授予角色                                                         │
│  ├── 情况A：用户原本没有该角色 → 权限变化                                           │
│  │   ├── t_audit_log：记录"管理员授予了角色"，结果 SUCCESS                         │
│  │   └── t_permission_version_history：记录版本从 5 变成 6，关联 audit_log_id      │
│  │                                                                                 │
│  └── 情况B：用户已经拥有该角色 → 权限未变化                                         │
│      ├── t_audit_log：记录"管理员尝试授予角色"，结果 SKIPPED                       │
│      └── t_permission_version_history：❌ 不记录                                   │
│                                                                                     │
│  操作：管理员尝试授予角色但数据库异常导致失败                                       │
│  ├── t_audit_log：记录"管理员尝试授予角色"，结果 FAILED，记录错误信息              │
│  └── t_permission_version_history：❌ 不记录（事务回滚）                            │
│                                                                                     │
│  操作：管理员查询权限列表（只读操作）                                               │
│  ├── t_audit_log：❌ 不记录（只读查询不审计）                                      │
│  └── t_permission_version_history：❌ 不记录                                        │
└─────────────────────────────────────────────────────────────────────────────────────┘
```


## 6. 权限计算引擎

### 6.1 核心流程

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         权限计算流程                                                │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  输入：userId                                                                       │
│                                                                                     │
│  Step 1: 获取用户直接角色                                                          │
│  └── SELECT role_id FROM t_user_role WHERE user_id = #{userId}                    │
│                                    │                                                │
│                                    ▼                                                │
│  Step 2: 获取每个角色的权限配置（含行级规则）                                      │
│  └── SELECT p.permission_code, rp.row_rule_template, r.priority                   │
│      FROM t_role_permission rp JOIN t_permission p ON rp.permission_id = p.id     │
│      JOIN t_role r ON rp.role_id = r.id WHERE rp.role_id IN (...)                 │
│                                    │                                                │
│                                    ▼                                                │
│  Step 3: 行级规则合并（按角色优先级）                                              │
│  ├── ① 按角色优先级分组                                                            │
│  ├── ② 取最高优先级组                                                              │
│  ├── ③ 同级规则用 OR 合并                                                          │
│  └── ④ 低优先级规则忽略                                                            │
│                                    │                                                │
│                                    ▼                                                │
│  Step 4: 通配符展开                                                                │
│  └── order:* → order:read, order:create, order:update, order:export               │
│                                    │                                                │
│                                    ▼                                                │
│  Step 5: 叠加强制规则                                                              │
│  └── 从 t_permission.mandatory_row_rule_template 获取强制规则 AND 合并             │
│                                    │                                                │
│                                    ▼                                                │
│  输出：permissions(Set), rowRules(Map), permVersion(Long), roles(List)             │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 行级规则合并算法

```java
public class RowRuleMerger {
    /**
     * 合并用户多个角色的行级规则
     * 策略：取最高优先级角色的规则，同级用 OR 合并
     */
    public Map<String, String> mergeRules(List<RolePermission> rolePermissions) {
        Map<Integer, List<RolePermission>> groups = rolePermissions.stream()
            .collect(Collectors.groupingBy(rp -> rp.getRole().getPriority()));
        
        Integer maxPriority = groups.keySet().stream().max(Integer::compareTo).orElse(0);
        List<RolePermission> topGroup = groups.get(maxPriority);
        
        Map<String, List<String>> rulesByTable = new HashMap<>();
        for (RolePermission rp : topGroup) {
            Map<String, String> tableRules = rp.getRowRuleTemplate();
            if (tableRules != null && !tableRules.isEmpty()) {
                for (Map.Entry<String, String> entry : tableRules.entrySet()) {
                    rulesByTable.computeIfAbsent(entry.getKey(), k -> new ArrayList<>())
                        .add(entry.getValue());
                }
            }
        }
        
        Map<String, String> result = new HashMap<>();
        for (Map.Entry<String, List<String>> entry : rulesByTable.entrySet()) {
            List<String> rules = entry.getValue();
            if (rules.size() == 1) {
                result.put(entry.getKey(), rules.get(0));
            } else {
                result.put(entry.getKey(), rules.stream()
                    .map(r -> "(" + r + ")")
                    .collect(Collectors.joining(" OR ")));
            }
        }
        return result;
    }
}
```


## 7. 行级规则引擎

### 7.1 核心机制

行级规则在 SQL 执行时动态注入 WHERE 条件。模板绑定在 `t_role_permission` 表上，实现"同权不同数据"。

### 7.2 行级规则模板格式

```json
{
  "orders": "salesperson_id = #{user.id} AND status != 'deleted'",
  "order_items": "order_id IN (SELECT id FROM orders WHERE salesperson_id = #{user.id})"
}
```

### 7.3 支持的占位符

| 占位符 | 数据来源 | 说明 |
|--------|----------|------|
| `#{user.id}` | 用户属性 | 当前用户 ID |
| `#{user.username}` | 用户属性 | 当前用户登录名 |
| `#{user.department}` | 用户属性 | 当前用户部门 |
| `#{user.departments}` | 用户属性 | 用户所有部门（用于 IN 查询） |

### 7.4 行级规则模板校验器

```java
@Component
public class RowRuleValidator {
    
    private static final Set<String> DANGEROUS_KEYWORDS = Set.of(
        "UNION", "DROP", "DELETE", "INSERT", "UPDATE", 
        ";", "--", "/*", "*/", "EXEC", "EXECUTE"
    );
    
    public void validate(String tableName, String ruleTemplate) {
        String upper = ruleTemplate.toUpperCase();
        for (String keyword : DANGEROUS_KEYWORDS) {
            if (upper.contains(keyword)) {
                throw new SecurityException("危险关键字: " + keyword);
            }
        }
        
        Pattern pattern = Pattern.compile("#\\{([^}]+)\\}");
        Matcher matcher = pattern.matcher(ruleTemplate);
        while (matcher.find()) {
            String placeholder = matcher.group(1);
            if (!placeholder.startsWith("user.")) {
                throw new SecurityException("不允许的占位符: " + placeholder);
            }
        }
    }
}
```


## 8. 智能数据访问层（Client）

### 8.1 接口定义（a2a-permission-api）

```java
// 纯 Java 接口，无任何 Feign/Web 注解
public interface PermissionQueryService {
    
    // 聚合接口：Token Exchange 准备
    TokenExchangePrepareResponse prepareTokenExchange(TokenExchangePrepareRequest request);
    
    // 独立查询接口
    UserFullPermissionDTO getUserFullPermissions(Long userId);
    AgentDTO getAgent(String clientId);
    AclCheckResult checkAcl(String sourceClientId, String targetClientId);
}
```

### 8.2 聚合 DTO 定义

```java
@Data
public class TokenExchangePrepareRequest {
    private Long userId;
    private String clientId;
    private String targetAgent;
    private Set<String> requestedScopes;
}

@Data
public class TokenExchangePrepareResponse {
    private Long userId;
    private String username;
    private String permVersion;
    private Set<String> permissions;
    private Map<String, String> rowRules;
    private List<RoleInfo> roles;
    private AgentDTO agent;
    private AclCheckResult aclResult;
}
```

### 8.3 本地实现（local-client）

```java
@Component
@ConditionalOnProperty(name = "a2a.permission.mode", havingValue = "local")
public class LocalPermissionQueryService implements PermissionQueryService {
    
    @Override
    public TokenExchangePrepareResponse prepareTokenExchange(TokenExchangePrepareRequest request) {
        // 直连 MySQL 聚合查询（见第 6 章）
        // ...
    }
}
```

### 8.4 远程实现（remote-client）

```java
@FeignClient(name = "a2a-permission-service")
@ConditionalOnProperty(name = "a2a.permission.mode", havingValue = "remote", matchIfMissing = true)
public interface RemotePermissionQueryService extends PermissionQueryService {
    
    @Override
    @PostMapping("/api/v1/token/exchange/prepare")
    TokenExchangePrepareResponse prepareTokenExchange(@RequestBody TokenExchangePrepareRequest request);
    
    @Override
    @GetMapping("/api/v1/users/{userId}/permissions")
    UserFullPermissionDTO getUserFullPermissions(@PathVariable("userId") Long userId);
    
    @Override
    @GetMapping("/api/v1/agents/{clientId}")
    AgentDTO getAgent(@PathVariable("clientId") String clientId);
    
    @Override
    @GetMapping("/api/v1/acl/check")
    AclCheckResult checkAcl(@RequestParam("source") String sourceClientId,
                            @RequestParam("target") String targetClientId);
}
```


## 9. 权限管理操作与审计

### 9.1 操作类型枚举

```java
public enum OperationType {
    // 用户-角色授权
    ROLE_GRANT, ROLE_REVOKE,
    // 角色-权限关联
    PERM_GRANT, PERM_REVOKE,
    // 行级规则更新
    ROW_RULE_UPDATE,
    // 角色管理
    ROLE_CREATE, ROLE_UPDATE, ROLE_DELETE,
    // 权限定义管理
    PERM_CREATE, PERM_UPDATE, PERM_DELETE,
    // ACL 管理
    ACL_CREATE, ACL_UPDATE, ACL_DELETE,
    // Agent 管理
    AGENT_REGISTER, AGENT_UPDATE, AGENT_SUSPEND, AGENT_ACTIVATE, AGENT_SECRET_ROTATE
}
```

### 9.2 操作结果枚举

| 值 | 含义 | 说明 |
|----|------|------|
| `SUCCESS` | 操作成功 | 业务操作执行成功，权限可能变化也可能未变化 |
| `FAILED` | 操作失败 | 业务操作执行失败（数据库异常、校验失败等） |
| `SKIPPED` | 操作跳过 | 操作执行但权限未变化（如重复授权） |

### 9.3 operation_type 与版本变更的触发关系

| 操作类型 | 是否触发 perm_version 递增 | 说明 |
|----------|---------------------------|------|
| `ROLE_GRANT` | ✅ 是 | 用户新增角色，权限集合可能变化 |
| `ROLE_REVOKE` | ✅ 是 | 用户移除角色，权限集合可能变化 |
| `PERM_GRANT` | ✅ 是 | 角色新增权限，影响拥有该角色的所有用户 |
| `PERM_REVOKE` | ✅ 是 | 角色移除权限，影响拥有该角色的所有用户 |
| `ROW_RULE_UPDATE` | ✅ 是 | 行级规则变化，不影响权限码集合但影响数据范围，需递增版本号 |
| `ROLE_DELETE` | ✅ 是 | 删除角色，需清理用户关联并重新计算权限 |
| `PERM_DELETE` | ✅ 是 | 删除权限定义，需清理角色关联并重新计算权限 |
| `ROLE_CREATE` | ❌ 否 | 仅创建角色，未授权给用户 |
| `ROLE_UPDATE` | ❌ 否 | 仅更新角色名称/描述，不影响权限 |
| `PERM_CREATE` | ❌ 否 | 仅创建权限定义，未关联到角色 |
| `PERM_UPDATE` | ❌ 否 | 仅更新权限描述/风险等级（不含行级规则） |
| `ACL_*` | ❌ 否 | 不影响用户权限 |
| `AGENT_*` | ❌ 否 | 不影响用户权限 |

### 9.4 权限管理服务实现

```java
@Service
@Slf4j
@Transactional
public class PermissionAdminService {
    
    @Autowired
    private AuditLogService auditLogService;
    @Autowired
    private PermissionVersionHistoryDao versionHistoryDao;
    @Autowired
    private PermissionChangeProducer changeProducer;
    
    public void grantRole(Long userId, Long roleId, Long operatorId) {
        String traceId = MDC.get("traceId");
        
        try {
            // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            // 1. 检查是否已存在
            // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            if (userRoleDao.exists(userId, roleId)) {
                // 权限未变化 → 记录审计日志（SKIPPED），不记录版本历史
                auditLogService.record(AuditLogDTO.builder()
                    .traceId(traceId)
                    .operationType("ROLE_GRANT")
                    .operatorId(operatorId)
                    .targetUserId(userId)
                    .targetRoleId(roleId)
                    .operationResult("SKIPPED")
                    .operationDetail("{\"reason\":\"用户已拥有该角色，授予操作被忽略\"}")
                    .build()
                );
                return;
            }
            
            // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            // 2. 获取变更前数据
            // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            UserFullPermissionDTO beforeData = permissionEngine.calculate(userId);
            Long oldVersion = userDao.getPermVersion(userId);
            
            // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            // 3. 执行业务操作
            // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            userRoleDao.insert(userId, roleId, operatorId);
            UserFullPermissionDTO afterData = permissionEngine.calculate(userId);
            
            // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            // 4. 更新 Redis 缓存
            // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            String redisKey = "user:perm:" + userId;
            redisTemplate.opsForHash().putAll(redisKey, buildHash(afterData));
            redisTemplate.expire(redisKey, Duration.ofHours(24));
            
            // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            // 5. 递增权限版本号
            // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            Long newVersion = redisTemplate.opsForValue().increment("perm:ver:" + userId);
            userDao.updatePermVersion(userId, newVersion);
            
            // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            // 6. 记录审计日志（操作审计 - 记录所有操作）
            // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            Long auditLogId = auditLogService.record(AuditLogDTO.builder()
                .traceId(traceId)
                .operationType("ROLE_GRANT")
                .operatorId(operatorId)
                .targetUserId(userId)
                .targetRoleId(roleId)
                .operationResult("SUCCESS")
                .operationDetail(buildDetailJson(beforeData, afterData))
                .build()
            );
            
            // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            // 7. 记录版本历史（版本追溯 - 仅权限变化时）
            // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            versionHistoryDao.insert(VersionHistory.builder()
                .userId(userId)
                .oldVersion(oldVersion)
                .newVersion(newVersion)
                .triggerOperation("ROLE_GRANT")
                .auditLogId(auditLogId)
                .affectedPermissions(buildAffectedPermissions(afterData.getPermissions()))
                .build()
            );
            
            // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            // 8. 发送 RocketMQ 消息
            // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            changeProducer.sendPermissionChange(userId, newVersion);
            
        } catch (Exception e) {
            // 操作失败 → 记录审计日志（FAILED）
            auditLogService.record(AuditLogDTO.builder()
                .traceId(traceId)
                .operationType("ROLE_GRANT")
                .operatorId(operatorId)
                .targetUserId(userId)
                .targetRoleId(roleId)
                .operationResult("FAILED")
                .operationDetail("{\"reason\":\"" + e.getMessage() + "\"}")
                .build()
            );
            throw e;
        }
    }
}
```


## 10. 缓存与消息机制

### 10.1 多级缓存架构

| 层级 | 组件 | TTL | 失效方式 |
|------|------|-----|----------|
| L1 | Caffeine 本地缓存 | 300s | TTL 过期 + MQ 主动失效 |
| L2 | Redis 缓存 | 24h | 控制面写入时直接更新 |
| L3 | MySQL | - | 数据唯一真实来源 |

### 10.2 消息架构（RocketMQ）

```
控制面 (a2a-permission-service)
    │
    │ PermissionChangeProducer.sendPermissionChange(userId, newVersion)
    │ Topic: permission-change, Tag: USER_PERMISSION_CHANGED, Keys: userId
    ▼
RocketMQ Broker
    │ 按 userId 哈希路由到同一 Queue（保证同一用户消息有序）
    ▼
数据面 (remote-client 订阅者)
    │ PermissionChangeSubscriber.onMessage(message)
    │ 1. 解析 userId, newVersion
    │ 2. 清除该用户所有本地 Caffeine 缓存
    │ 3. 更新本地版本号
```

### 10.3 RocketMQ 生产者

```java
@Component
public class PermissionChangeProducer {
    
    public void sendPermissionChange(Long userId, Long newVersion) {
        PermissionChangeMessage message = PermissionChangeMessage.builder()
            .userId(userId).newVersion(newVersion).timestamp(System.currentTimeMillis()).build();
        
        rocketMQTemplate.syncSend(
            TOPIC + ":" + TAG, message,
            (mqs, msg, arg) -> {
                Long uid = (Long) arg;
                int index = (int) (uid % mqs.size());
                return mqs.get(index);
            },
            userId
        );
    }
}
```

### 10.4 RocketMQ 订阅者（remote-client）

```java
@Component
@ConditionalOnProperty(name = "a2a.permission.mode", havingValue = "remote")
@RocketMQMessageListener(topic = "permission-change", selectorExpression = "USER_PERMISSION_CHANGED")
public class PermissionChangeSubscriber implements RocketMQListener<String> {
    
    @Override
    public void onMessage(String messageBody) {
        PermissionChangeMessage msg = parse(messageBody);
        // 清除该用户所有本地缓存
        caffeineCache.asMap().keySet().stream()
            .filter(k -> ((String) k).contains(":" + msg.getUserId()))
            .forEach(caffeineCache::invalidate);
    }
}
```


## 11. 权限版本号机制

### 11.1 存储设计

```sql
-- t_user 表存储当前版本号
perm_version BIGINT NOT NULL DEFAULT 0

-- 独立历史表记录变更明细
t_permission_version_history
```

### 11.2 触发条件

`perm_version` **仅在用户最终权限集合发生变化时递增**：

- ✅ 角色授予/撤销（权限集合变化）
- ✅ 角色权限变更（影响拥有该角色的所有用户）
- ✅ 行级规则更新（影响数据范围）
- ❌ 角色创建/更新（不影响用户权限）
- ❌ 权限定义创建/更新（不影响用户权限）
- ❌ ACL/Agent 操作（不影响用户权限）

### 11.3 工作原理

```
管理员授予角色
    │
    ▼
1. INSERT t_user_role (授权记录)
    │
    ▼
2. 重新计算用户最终权限集合
    │
    ▼
3. 更新 Redis: user:perm:{userId}
    │
    ▼
4. 递增 perm_version: perm:ver:{userId} → newVersion
    │
    ▼
5. UPDATE t_user SET perm_version = newVersion
    │
    ▼
6. INSERT t_permission_version_history (版本变更记录)
    │
    ▼
7. INSERT t_audit_log (操作审计记录)
    │
    ▼
8. 发送 RocketMQ 消息: permission-change
```


## 12. 部署与配置

### 12.1 控制面配置（a2a-permission-service）

```yaml
spring:
  application:
    name: a2a-permission-service
  datasource:
    url: jdbc:mysql://mysql.agent.svc:3306/a2a_permission_db
    username: ${DB_USER}
    password: ${DB_PASSWORD}
  redis:
    host: redis.agent.svc
    port: 6379

rocketmq:
  name-server: rocketmq.agent.svc:9876
  producer:
    group: permission-change-producer

a2a:
  permission:
    mode: local
    cache:
      caffeine-ttl-seconds: 300
      redis-ttl-seconds: 86400
    row-rule:
      allowed-attr-keys: [id, username, department, region]
    mq:
      topic: permission-change
      tag: USER_PERMISSION_CHANGED
```

### 12.2 上层调用方配置（认证服务 / Agent 节点）

```yaml
spring:
  redis:
    host: redis.agent.svc
    port: 6379

rocketmq:
  name-server: rocketmq.agent.svc:9876
  consumer:
    group: permission-change-consumer

a2a:
  permission:
    mode: remote
    cache:
      caffeine-ttl-seconds: 300
    mq:
      topic: permission-change
      tag: USER_PERMISSION_CHANGED
```

### 12.3 启动检查清单

- [ ] 执行 DDL 创建所有表
- [ ] 初始化基础角色和权限定义
- [ ] 配置 Redis 和 RocketMQ 连接
- [ ] 验证控制面启动（mode=local）
- [ ] 验证上层调用方启动（mode=remote，**不引入 local-client**）
- [ ] 验证聚合接口 `POST /api/v1/token/exchange/prepare`
- [ ] 验证授权操作同时写入 `t_audit_log` 和 `t_permission_version_history`
- [ ] 验证重复授权写入 `SKIPPED` 审计日志但不记录版本历史
- [ ] 验证操作失败写入 `FAILED` 审计日志
- [ ] 验证 RocketMQ 消息发送与订阅


## 13. 附录

### 13.1 状态字段枚举

| 表 | 字段 | 值 | 含义 |
|----|------|---|------|
| t_user | status | 1 | 正常 |
| | | 2 | 锁定 |
| | | 3 | 禁用 |
| | | 0 | 已删除 |
| t_role | status | 1 | 有效 |
| | | 0 | 无效 |
| t_agent | status | 1 | 正常 |
| | | 2 | 暂停 |
| | | 3 | 已注销 |

### 13.2 风险等级枚举

| 值 | 含义 | 示例权限 |
|----|------|----------|
| 0 | 低 | `order:read` |
| 1 | 中 | `order:export` |
| 2 | 高 | `order:delete` |
| 3 | 严重 | `perm:grant` |

### 13.3 操作类型完整枚举

| 操作类型 | 说明 | 触发版本变更 |
|----------|------|-------------|
| `ROLE_GRANT` | 授予用户角色 | ✅ 是 |
| `ROLE_REVOKE` | 撤销用户角色 | ✅ 是 |
| `PERM_GRANT` | 授予角色权限 | ✅ 是 |
| `PERM_REVOKE` | 撤销角色权限 | ✅ 是 |
| `ROW_RULE_UPDATE` | 更新行级规则 | ✅ 是 |
| `ROLE_DELETE` | 删除角色 | ✅ 是 |
| `PERM_DELETE` | 删除权限定义 | ✅ 是 |
| `ROLE_CREATE` | 创建角色 | ❌ 否 |
| `ROLE_UPDATE` | 更新角色信息 | ❌ 否 |
| `PERM_CREATE` | 创建权限定义 | ❌ 否 |
| `PERM_UPDATE` | 更新权限定义 | ❌ 否 |
| `ACL_*` | ACL 管理 | ❌ 否 |
| `AGENT_*` | Agent 管理 | ❌ 否 |

### 13.4 两表职责总结

| 维度 | t_audit_log | t_permission_version_history |
|------|-------------|------------------------------|
| 记录时机 | 所有权限管理操作 | 仅权限实际变化时 |
| 记录内容 | 谁、何时、做什么、结果 | 版本变化、权限快照 |
| 主要用途 | 合规审计、异常追溯 | Token 失效判断、版本回溯 |
| 关联关系 | 独立存在 | 通过 audit_log_id 关联 |
