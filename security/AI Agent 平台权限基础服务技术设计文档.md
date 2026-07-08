
> **技术栈**：JDK 21 + Spring Boot 3.5.3 + MyBatis-Plus 3.5.9 + Redis 7.x + MySQL 8.0 + Spring Cloud OpenFeign + RocketMQ 5.x

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
11. [权限双版本号机制](#11-权限双版本号机制)
12. [部署与配置](#12-部署与配置)
13. [附录](#13-附录)
14. [V8.7 → V14.0 全量修正记录](#14-v87--v140-全量修正记录)

---

## 1. 文档概述

### 1.1 文档定位

本文档定义 AI Agent 平台**权限基础服务**的完整技术设计方案。权限基础服务作为独立的微服务，提供权限数据的**写入、计算、缓存和查询**能力，供上层认证服务和各 Agent 节点调用。

### 1.2 核心能力（V14.0 终版）

| 能力域 | 说明 |
| ------ | ---- |
| **功能权限管理** | 用户能否执行某个操作（如 `order:export`）的规则定义与存储 |
| **数据权限管理** | 用户能操作哪些数据行的规则定义（通过行级规则实现），支持 **ALLOW/DENY** 策略及**角色优先级** |
| **角色与权限绑定** | 角色定义、角色优先级、角色-权限关联（含行级规则），优先级在DENY/ALLOW冲突时生效 |
| **Token Exchange 数据聚合** | 一次调用返回 Token Exchange 所需的所有权限数据 |
| **双版本权限管理** | 用户个人版本（`user_perm_ver`）+ 角色版本（`role_version`）MD5聚合 + 乐观锁快速失败 |
| **操作审计追溯** | 通过 `t_audit_log` 记录所有权限管理操作的**增量变更（Diff）** 审计轨迹，使用RocketMQ外部队列保证不丢失 |
| **版本变更追溯** | 通过 `t_permission_version_history` 和 `t_role_permission_version_history` 追溯版本变化原因 |
| **缓存与消息** | 多级缓存（Caffeine `AsyncLoadingCache` + Redis）+ RocketMQ 动态ConsumerGroup伪广播 + 反向索引 + RemovalListener自动清理 |
| **降级与熔断** | 数据面集成Feign Fallback + Resilience4j熔断器，控制面不可用时使用Redis缓存降级 |


## 2. 核心设计原则

| 原则 | 说明 |
| ---- | ---- |
| **物理隔离** | `a2a-permission-remote`（纯远程调用）和 `a2a-permission-admin`（控制面）分属不同模块，上层调用方**永远不引入** `admin` 模块，编译期杜绝误用本地 DB 调用的可能 |
| **读写分离** | 控制面（Admin模块）负责写入和计算，数据面（Remote模块）负责读取和缓存 |
| **远程调用** | `remote` 模块仅提供 Feign 远程调用能力，**不包含任何本地 DB 实现** |
| **缓存优先** | 所有读取操作优先走多级缓存（Caffeine → Redis），使用 `AsyncLoadingCache` 防缓存击穿 |
| **最终一致性** | 权限变更通过 RocketMQ 广播，数据面缓存异步刷新；审计日志通过RocketMQ外部队列保证不丢失 |
| **双版本驱动** | 用户权限版本 + 角色权限版本 MD5聚合共同决定 Token 有效性 |
| **安全左移** | 行级规则模板必须通过 AST 语法树校验，运行时强制使用 MyBatis-Plus `apply` 参数绑定 |
| **审计轻量化** | 操作详情仅记录增量变化（Diff），使用强类型 `AuditDiff` 抽象类保证 JSON 结构严格一致 |
| **降级兜底** | 数据面集成 Feign Fallback 和熔断器，控制面不可用时使用 Redis 缓存降级鉴权 |
| **乐观锁快速失败** | 版本号更新采用乐观锁机制（`WHERE version = #{oldVersion}`），并发冲突时快速抛出异常 |


## 3. 整体架构设计

### 3.1 部署架构（V14.0 修正：remote 纯远程调用，admin 包含本地实现）

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
│  │  │  ├── a2a-permission-common (缓存基础设施)                           │   │   │
│  │  │  └── a2a-permission-remote (Feign 远程调用 + MQ 订阅 + 缓存)        │   │   │
│  │  │  ❌ 不引入 a2a-permission-admin（无 MySQL 依赖）                    │   │   │
│  │  │  配置：mode=remote                                                  │   │   │
│  │  │  V14.0 新增：Feign Fallback + Resilience4j 熔断器                  │   │   │
│  │  └─────────────────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                             │
│                                      │ Feign 调用（通过 remote 模块）              │
│                                      ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │           控制面 - a2a-permission-admin (独立微服务，包含本地实现)          │   │
│  │  ┌─────────────────────────────────────────────────────────────────────┐   │   │
│  │  │  依赖：                                                             │   │   │
│  │  │  ├── a2a-permission-api (DTO + 接口)                               │   │   │
│  │  │  ├── a2a-permission-common (通用缓存 + Redis)                      │   │   │
│  │  │  ├── MyBatis-Plus + MySQL (本地 DB 实现)                           │   │   │
│  │  │  └── RocketMQ (MQ 发送)                                            │   │   │
│  │  │  ❌ 不引入 a2a-permission-remote（不需要远程调用）                  │   │   │
│  │  │  配置：mode=local                                                   │   │   │
│  │  │  职责：数据写入、权限计算（含本地引擎）、Redis 更新、MQ发送         │   │   │
│  │  └─────────────────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                             │
│                                      ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                        Redis + MySQL + RocketMQ                            │   │
│  │  ┌───────────────────────────────────────────────────────────────────────┐ │   │
│  │  │  MySQL: 权限数据 + 操作审计日志（分区表）+ 版本历史                 │ │   │
│  │  │  Redis: 权限缓存 + 版本号 + 降级兜底数据                            │ │   │
│  │  │  RocketMQ: Topic: permission-change (USER/ROLE)                     │ │   │
│  │  │           Topic: audit-log (V14.0 新增审计日志外部队列)             │ │   │
│  │  └───────────────────────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 模块物理结构（四模块架构 - V14.0 修正）

```
a2a-permission/
├── a2a-permission-api/                         # 【契约层 - 纯接口JAR】
│   ├── model/                                  # DTO 定义
│   │   ├── TokenExchangePrepareRequest.java
│   │   ├── TokenExchangePrepareResponse.java
│   │   ├── UserFullPermissionDTO.java
│   │   ├── AgentDTO.java
│   │   ├── AclCheckResult.java
│   │   ├── AuditLogDTO.java
│   │   └── PermissionChangeMessage.java
│   ├── enums/                                  # 枚举定义
│   │   ├── OperationType.java
│   │   ├── OperationResult.java
│   │   ├── Effect.java
│   │   ├── ChangeType.java
│   │   ├── UserStatus.java
│   │   ├── RoleStatus.java
│   │   ├── AgentStatus.java
│   │   └── RiskLevel.java
│   └── spi/                                    # SPI 接口定义
│       └── PermissionQueryService.java         # 纯 Java 接口，无 Feign 注解
│
├── a2a-permission-common/                      # 【公共缓存层 - 嵌入式JAR】
│   ├── cache/                                  # 缓存装饰器 + 反向索引管理器
│   │   ├── CachedPermissionQueryService.java   # 使用 AsyncLoadingCache
│   │   ├── LocalCacheManager.java              # 反向索引 + RemovalListener
│   │   ├── RoleVersionCache.java               # 角色版本本地缓存
│   │   └── UserVersionCache.java               # 用户聚合版本本地缓存
│   ├── redis/                                  # Redis 操作封装
│   │   └── RedisCacheManager.java
│   └── config/
│       └── CommonAutoConfig.java
│
├── a2a-permission-remote/                      # 【远程调用层 - 嵌入式JAR】
│   ├── remote/                                 # Feign 远程调用
│   │   ├── RemotePermissionQueryService.java   # Feign 接口
│   │   └── PermissionQueryFallback.java        # V14.0 Feign Fallback
│   ├── subscriber/                             # RocketMQ 订阅者
│   │   └── PermissionChangeSubscriber.java
│   └── config/
│       └── RemoteAutoConfig.java
│
└── a2a-permission-admin/                       # 【控制面 - 独立微服务】
    ├── controller/                             # REST API
    │   └── PermissionController.java
    ├── service/                                # 业务逻辑
    │   ├── PermissionAdminService.java         # 增删改操作（含审计+版本）
    │   └── PermissionQueryServiceImpl.java     # 查询服务实现
    ├── engine/                                 # 【V14.0 新增】权限计算引擎（本地实现）
    │   ├── PermissionCalculator.java
    │   ├── RowRuleMerger.java
    │   ├── MandatoryRuleInjector.java
    │   ├── WildcardExpander.java
    │   ├── UserContextEnricher.java
    │   ├── CombinedVersionCalculator.java
    │   └── ExpressionOptimizer.java
    ├── security/                               # 【V14.0 新增】安全包
    │   └── RowRuleValidator.java
    ├── audit/                                  # 审计强类型
    │   ├── AuditDiff.java                      # 抽象基类 + defaultImpl
    │   ├── RoleGrantDiff.java
    │   ├── PermGrantDiff.java
    │   ├── RowRuleUpdateDiff.java
    │   └── ErrorDiff.java
    ├── mapper/                                 # MyBatis Mapper（本地DB访问）
    │   ├── UserMapper.java
    │   ├── RoleMapper.java
    │   ├── UserRoleMapper.java
    │   ├── PermissionMapper.java
    │   ├── RolePermissionMapper.java
    │   ├── DepartmentMapper.java
    │   └── AuditLogMapper.java
    ├── producer/                               # RocketMQ 消息发送
    │   ├── PermissionChangeProducer.java
    │   └── AuditLogProducer.java
    ├── consumer/                               # RocketMQ 消息消费
    │   └── AuditLogConsumer.java
    └── config/
        └── AdminAutoConfig.java
```

### 3.3 模块物理隔离说明（V14.0 修正）

| 模块 | 使用方 | 依赖关系 | 物理隔离说明 |
|------|--------|----------|-------------|
| `a2a-permission-api` | 所有模块 | 无 | 纯接口/DTO，无任何实现依赖 |
| `a2a-permission-common` | remote、admin | api + Redis + Caffeine | 无 MySQL/Feign/MQ 依赖 |
| `a2a-permission-remote` | 上层调用方 | api + common + Feign + MQ | **纯远程调用，无 MySQL 依赖，无本地 DB 实现** |
| `a2a-permission-admin` | 独立微服务 | api + common + MyBatis-Plus + MySQL + Web + RocketMQ | **包含完整本地实现（Engine + Mapper + Security）** |

> **关键约束**（V14.0 修正）：
> - 上层调用方（认证服务/Agent节点）**只依赖** `a2a-permission-api` + `a2a-permission-common` + `a2a-permission-remote`，**永远不引入** `a2a-permission-admin`
> - `a2a-permission-admin` **包含完整的本地 DB 访问能力**（MyBatis-Plus Mapper + 权限计算引擎 + 安全校验）
> - `a2a-permission-remote` **不包含任何本地 DB 实现**，仅提供 Feign 远程调用接口


## 4. 模块划分与依赖关系（V14.0 修正）

### 4.1 依赖关系图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         模块依赖关系图（V14.0 四模块架构）                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │              a2a-permission-api (纯契约层)                                   │   │
│  │  依赖：无  │  包含：PermissionQueryService 接口 + DTO + 枚举 + 常量        │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│              ▲                                                                      │
│              │ 依赖                                                                 │
│  ┌───────────┴─────────────────────────────────────────────────────────────────────┐│
│  │              a2a-permission-common (公共缓存层)                                ││
│  │  依赖：api + Redis + Caffeine  │  特点：无 MySQL/Feign/MQ 依赖               ││
│  └─────────────────────────────────────────────────────────────────────────────┘│
│              ▲                                   ▲                                │
│              │ 依赖                              │ 依赖                            │
│  ┌───────────┴─────────────────────────────┐  ┌─┴────────────────────────────────┐│
│  │  a2a-permission-remote（远程调用层）    │  │  a2a-permission-admin（控制面）  ││
│  │  ┌─────────────────────────────────────┐ │  │  ┌─────────────────────────────┐ ││
│  │  │ remote包：Feign接口 + Fallback     │ │  │  │ controller + service        │ ││
│  │  │ subscriber：MQ 订阅者              │ │  │  │ engine（权限计算引擎）       │ ││
│  │  └─────────────────────────────────────┘ │  │  │ security（AST安全校验）     │ ││
│  │  依赖：api + common + Feign + MQ       │  │  │ mapper（MyBatis DB访问）     │ ││
│  │  ❌ 不含 MySQL / MyBatis               │  │  │ audit + producer + consumer  │ ││
│  │  ❌ 不含 controller / web              │  │  └─────────────────────────────┘ ││
│  └─────────────────────────────────────────┘  │  依赖：api + common + MyBatis   ││
│              │                               │       + MySQL + Web + RocketMQ   ││
│              │                               │  ❌ 不引入 remote 模块            ││
│              ▼                               └──────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────────────────────────────┐│
│  │  使用方                  │  部署形态                                           ││
│  ├──────────────────────────┼───────────────────────────────────────────────────┤│
│  │  上层调用方              │  依赖 api + common + remote（mode=remote）         ││
│  │  （认证/Agent）          │  ❌ 不引入 admin                                   ││
│  │                          │  职责：远程 Feign 调用 + MQ 订阅 + 本地缓存        ││
│  ├──────────────────────────┼───────────────────────────────────────────────────┤│
│  │  a2a-permission-admin    │  依赖 api + common + MyBatis + MySQL（mode=local）││
│  │  （独立微服务）          │  ❌ 不引入 remote                                 ││
│  │                          │  职责：写入 + 计算（本地引擎）+ Redis更新 + MQ发送 ││
│  └──────────────────────────┴───────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 模块详细说明（V14.0 修正）

| 模块 | 职责 | 关键依赖 | 排除依赖 |
|------|------|----------|----------|
| **a2a-permission-api** | 定义 DTO、枚举、SPI 接口 | 无 | 无 |
| **a2a-permission-common** | 缓存基础设施（Caffeine + Redis）、反向索引、版本缓存 | api、spring-boot-starter-data-redis、caffeine | MySQL、Feign、MQ、Web |
| **a2a-permission-remote** | **纯远程调用**：Feign接口 + Fallback + MQ订阅 | api、common、feign、resilience4j、rocketmq | **MySQL、MyBatis-Plus、Web、Controller** |
| **a2a-permission-admin** | **控制面微服务（含本地实现）**：REST API、权限计算引擎、安全管理、MyBatis DB访问、审计日志、MQ发送 | api、common、mybatis-plus、mysql、web、rocketmq | remote模块（不需要远程调用） |

### 4.3 四模块架构的优势

| 优势 | 说明 |
|------|------|
| **依赖最小化** | 上层调用方仅依赖 `api` + `common` + `remote`，不引入 `admin`，编译期杜绝误用 |
| **职责清晰** | `remote` 模块**纯远程调用**（无本地DB），`admin` 模块**包含完整本地实现**（引擎+Mapper+Security） |
| **复用最大化** | `common` 模块被 `remote` 和 `admin` 共同复用，避免重复代码 |
| **部署独立** | `admin` 模块作为独立微服务部署，`remote` 模块作为嵌入式JAR被上层调用方依赖 |
| **安全隔离** | `admin` 模块的本地实现（Engine + Mapper）不会泄漏到上层调用方 |

### 4.4 模块加载控制（V14.0 修正）

```java
// admin 模块配置（本地模式，包含所有本地实现）
// a2a-permission-admin 启动时自动加载所有本地组件
a2a.permission.mode: local

// 上层调用方配置（远程模式）
// a2a-permission-remote 模块中的 Feign 接口自动生效
a2a.permission.mode: remote
```


## 5. 数据模型设计（V14.0 最终 DDL）

### 5.1 完整 DDL（V14.0 修正）

```sql
-- ============================================================
-- 1. 部门表 (t_department) - V14.0 修正唯一约束
-- ============================================================
CREATE TABLE t_department (
    id           BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '部门主键ID',
    dept_name    VARCHAR(128) NOT NULL COMMENT '部门名称',
    parent_id    BIGINT COMMENT '父部门ID，关联本表 id',
    path         VARCHAR(512) COMMENT '部门路径，由应用层维护，如 /1/5/20/',
    status       TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1-有效, 0-无效',
    description  TEXT COMMENT '部门描述',
    created_by   BIGINT COMMENT '创建人ID，关联 t_user.id',
    updated_by   BIGINT COMMENT '最后修改人ID，关联 t_user.id',
    created_at   TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '创建时间',
    updated_at   TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6) COMMENT '最后修改时间',
    UNIQUE KEY uk_parent_dept_name (parent_id, dept_name),
    INDEX idx_path (path(100))
) COMMENT '部门表';

-- ============================================================
-- 2. 用户表 (t_user) - V14.0 移除冗余索引
-- ============================================================
CREATE TABLE t_user (
    id                       BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '用户主键ID',
    username                 VARCHAR(64) NOT NULL UNIQUE COMMENT '用户登录名',
    password_hash            VARCHAR(256) NOT NULL COMMENT 'BCrypt 哈希密码',
    password_expires_at      TIMESTAMP(6) COMMENT '密码过期时间',
    password_failed_attempts TINYINT NOT NULL DEFAULT 0 COMMENT '连续失败次数',
    password_locked_until    TIMESTAMP(6) COMMENT '锁定到期时间',
    perm_version             BIGINT NOT NULL DEFAULT 0 COMMENT '用户个人权限版本号',
    status                   TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1-正常, 2-锁定, 3-禁用, 0-已删除',
    last_login_at            TIMESTAMP(6) COMMENT '最后登录时间',
    created_by               BIGINT COMMENT '创建人ID，关联 t_user.id',
    updated_by               BIGINT COMMENT '最后修改人ID，关联 t_user.id',
    created_at               TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '注册时间',
    updated_at               TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6) COMMENT '最后修改时间'
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
-- 4. 角色表 (t_role) - V14.0 移除冗余索引
-- ============================================================
CREATE TABLE t_role (
    id           BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '角色主键ID',
    role_name    VARCHAR(64) NOT NULL UNIQUE COMMENT '角色名称',
    priority     INT NOT NULL DEFAULT 0 COMMENT '角色优先级：数值越大优先级越高（DENY覆盖ALLOW决策）',
    role_version BIGINT NOT NULL DEFAULT 0 COMMENT '角色自身权限版本号',
    status       TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1-有效, 0-无效',
    description  TEXT COMMENT '角色描述',
    created_by   BIGINT COMMENT '创建人ID，关联 t_user.id',
    updated_by   BIGINT COMMENT '最后修改人ID，关联 t_user.id',
    created_at   TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '创建时间',
    updated_at   TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6) COMMENT '最后修改时间'
) COMMENT '角色表';

-- ============================================================
-- 5. 用户-角色关联表 (t_user_role)
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
) COMMENT '用户-角色关联表';

-- ============================================================
-- 6. 权限定义表 (t_permission) - V14.0 移除冗余索引
-- ============================================================
CREATE TABLE t_permission (
    id                           BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '权限主键ID',
    permission_code              VARCHAR(64) NOT NULL UNIQUE COMMENT '权限码，如 order:read',
    action_code                  VARCHAR(32) NOT NULL COMMENT '操作类型',
    risk_level                   TINYINT NOT NULL DEFAULT 0 COMMENT '风险等级',
    mandatory_row_rule_template  JSON COMMENT '强制行级规则模板（安全红线）',
    description                  TEXT COMMENT '权限说明',
    created_by                   BIGINT COMMENT '创建人ID，关联 t_user.id',
    updated_by                   BIGINT COMMENT '最后修改人ID，关联 t_user.id',
    created_at                   TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '创建时间',
    updated_at                   TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6) COMMENT '最后修改时间'
) COMMENT '权限定义表';

-- ============================================================
-- 7. 角色-权限关联表 (t_role_permission)
-- ============================================================
CREATE TABLE t_role_permission (
    id                BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '关联主键ID',
    role_id           BIGINT NOT NULL COMMENT '角色ID，关联 t_role.id',
    permission_id     BIGINT NOT NULL COMMENT '权限ID，关联 t_permission.id',
    effect            TINYINT NOT NULL DEFAULT 1 COMMENT '效果：1-ALLOW(允许), 0-DENY(拒绝)',
    row_rule_template JSON COMMENT '行级规则模板',
    created_by        BIGINT COMMENT '创建人ID，关联 t_user.id',
    updated_by        BIGINT COMMENT '最后修改人ID，关联 t_user.id',
    created_at        TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '创建时间',
    updated_at        TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6) COMMENT '最后修改时间',
    UNIQUE KEY uk_role_permission (role_id, permission_id),
    INDEX idx_permission_id (permission_id)
) COMMENT '角色-权限关联表';

-- ============================================================
-- 8. Agent 注册表 (t_agent) - V14.0 移除冗余索引
-- ============================================================
CREATE TABLE t_agent (
    id                  BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT 'Agent 主键ID',
    client_id           VARCHAR(128) NOT NULL UNIQUE COMMENT 'OAuth2 client_id',
    client_secret_hash  VARCHAR(256) NOT NULL COMMENT 'BCrypt 哈希',
    secret_expires_at   TIMESTAMP(6) NOT NULL DEFAULT (CURRENT_TIMESTAMP + INTERVAL 90 DAY) COMMENT '密钥过期时间',
    agent_name          VARCHAR(128) NOT NULL COMMENT 'Agent 名称',
    framework_type      VARCHAR(32) NOT NULL COMMENT 'agentscope / spring-ai / custom',
    agent_card_url      VARCHAR(512) COMMENT 'Agent Card 地址',
    public_key          TEXT COMMENT 'Agent 公钥（PEM）',
    capabilities        JSON COMMENT '能力声明',
    status              TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1-正常, 2-暂停, 3-已注销',
    created_by          BIGINT COMMENT '创建人ID，关联 t_user.id',
    updated_by          BIGINT COMMENT '最后修改人ID，关联 t_user.id',
    created_at          TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '注册时间',
    updated_at          TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6) COMMENT '最后修改时间'
) COMMENT 'Agent 注册表';

-- ============================================================
-- 9. A2A ACL 表 (t_a2a_acl) - 不变
-- ============================================================
CREATE TABLE t_a2a_acl (
    id                BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '关联主键ID',
    source_client_id  VARCHAR(128) NOT NULL COMMENT '调用方 client_id',
    target_client_id  VARCHAR(128) NOT NULL COMMENT '目标 client_id',
    allowed_scopes    JSON COMMENT '允许的 Scope 列表',
    created_by        BIGINT COMMENT '创建人ID，关联 t_user.id',
    updated_by        BIGINT COMMENT '最后修改人ID，关联 t_user.id',
    created_at        TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '创建时间',
    updated_at        TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6) COMMENT '最后修改时间',
    UNIQUE KEY uk_acl (source_client_id, target_client_id)
) COMMENT 'A2A ACL 白名单表';

-- ============================================================
-- 10. 操作审计表 (t_audit_log) - V14.0 减少索引 + 分区设计
-- ============================================================
CREATE TABLE t_audit_log (
    id                   BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '审计日志主键ID',
    trace_id             VARCHAR(64) COMMENT '全链路 Trace ID',
    operation_type       VARCHAR(32) NOT NULL COMMENT '操作类型',
    operator_id          BIGINT COMMENT '操作人ID',
    target_user_id       BIGINT COMMENT '目标用户ID',
    target_role_id       BIGINT COMMENT '目标角色ID',
    target_permission_id BIGINT COMMENT '目标权限ID',
    target_client_id     VARCHAR(128) COMMENT '目标 Agent client_id',
    operation_detail     JSON NOT NULL COMMENT '操作详情（强类型AuditDiff序列化）',
    operation_result     VARCHAR(16) NOT NULL COMMENT 'SUCCESS, FAILED, SKIPPED',
    client_ip            VARCHAR(64) COMMENT '客户端 IP',
    user_agent           VARCHAR(256) COMMENT 'User-Agent',
    created_at           TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '操作时间',
    INDEX idx_created_at (created_at),
    INDEX idx_operation_type (operation_type),
    INDEX idx_target_user_id (target_user_id)
) COMMENT '操作审计表';

-- ============================================================
-- 11. 用户权限版本变更历史表 (t_permission_version_history)
-- ============================================================
CREATE TABLE t_permission_version_history (
    id                   BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '历史记录主键ID',
    user_id              BIGINT NOT NULL COMMENT '用户ID',
    old_version          BIGINT COMMENT '变更前的版本号',
    new_version          BIGINT NOT NULL COMMENT '变更后的版本号',
    trigger_operation    VARCHAR(32) COMMENT '触发操作类型',
    audit_log_id         BIGINT COMMENT '关联的审计日志ID',
    affected_permissions JSON COMMENT '变更后的权限码列表（增量Diff）',
    created_at           TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '变更时间',
    INDEX idx_user_id (user_id),
    INDEX idx_created_at (created_at)
) COMMENT '用户权限版本变更历史表';

-- ============================================================
-- 12. 角色权限版本变更历史表 (t_role_permission_version_history)
-- ============================================================
CREATE TABLE t_role_permission_version_history (
    id                BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '历史记录主键ID',
    role_id           BIGINT NOT NULL COMMENT '角色ID',
    old_version       BIGINT COMMENT '旧版本号',
    new_version       BIGINT NOT NULL COMMENT '新版本号',
    trigger_operation VARCHAR(32) COMMENT '触发操作',
    audit_log_id      BIGINT COMMENT '关联操作审计日志ID',
    affected_perms    JSON COMMENT '变更的权限码列表（增量）',
    created_at        TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) COMMENT '变更时间',
    INDEX idx_role_id (role_id),
    INDEX idx_created_at (created_at)
) COMMENT '角色权限版本变更历史表';
```

### 5.2 核心表职责映射（V14.0 更新）

| 表名 | 核心职责 |
|------|----------|
| `t_user` | 存储 `perm_version`（个人专属变更） |
| `t_role` | 存储 `role_version`（角色自身权限变更），乐观锁更新 |
| `t_user_role` | 用户与角色的绑定关系 |
| `t_role_permission` | **新增 `effect` 字段**，支持 `ALLOW` / `DENY` |
| `t_permission_version_history` | 记录**用户级别**权限变更原因 |
| `t_role_permission_version_history` | 记录**角色级别**权限变更原因 |


## 6. 权限计算引擎（V14.0 位于 admin 模块）

> **V14.0 修正**：权限计算引擎（`engine` 包）从 `remote` 模块移至 `admin` 模块，因为只有控制面需要执行本地权限计算。`remote` 模块仅提供远程调用能力，不包含任何本地计算逻辑。

### 6.1 核心流程（V14.0 最终版）

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         权限计算流程（V14.0 Pipeline）                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  输入：userId                                                                       │
│                                                                                     │
│  Step 1: 获取用户直接角色                                                          │
│  └── SELECT role_id FROM t_user_role WHERE user_id = #{userId}                    │
│                                    │                                                │
│                                    ▼                                                │
│  Step 2: 获取每个角色的权限配置（含行级规则 & effect & 优先级）                    │
│  └── SELECT p.permission_code, rp.row_rule_template, rp.effect, r.priority        │
│      FROM t_role_permission rp JOIN t_permission p ON rp.permission_id = p.id     │
│      JOIN t_role r ON rp.role_id = r.id WHERE rp.role_id IN (...)                 │
│                                    │                                                │
│                                    ▼                                                │
│  Step 3: 【V14.0 时序修正】通配符展开 + 权限存在性校验                            │
│  └── order:* → 仅展开 t_permission 中已定义的具体权限码                           │
│                                    │                                                │
│                                    ▼                                                │
│  Step 4: 行级规则合并（V14.0 引入角色优先级）                                      │
│  ├── ① 按 (表, 权限码) 分组                                                       │
│  ├── ② 同一组内：按角色优先级排序，高优先级 DENY 覆盖低优先级 ALLOW               │
│  ├── ③ 将所有生效的 ALLOW 规则按表名 OR 合并                                      │
│  └── ④ 将所有生效的 DENY 规则按表名 OR 合并                                       │
│                                    │                                                │
│                                    ▼                                                │
│  Step 5: 【V14.0 新增】恒真/恒假表达式短路优化                                     │
│  ├── ① 若合并结果为 "1=0" → 直接拦截查询，无需继续                                │
│  └── ② 若合并结果为 "1=1" → 移除 WHERE 条件                                       │
│                                    │                                                │
│                                    ▼                                                │
│  Step 6: 强制规则注入（V14.0 修正获取逻辑）                                        │
│  └── 从 t_permission.mandatory_row_rule_template 获取，按权限码注入               │
│                                    │                                                │
│                                    ▼                                                │
│  Step 7: 计算双版本号聚合（MD5）                                                   │
│                                    │                                                │
│                                    ▼                                                │
│  输出：permissions(Set), rowRules(Map), combinedVersion(String)                   │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 行级规则合并算法（V14.0 引入角色优先级）

```java
// 位于 a2a-permission-admin 模块 engine 包
package io.github.latcn.a2a.permission.admin.engine;

@Component
public class RowRuleMerger {
    /**
     * V14.0 引入角色优先级逻辑
     * - 按 (表, 权限码) 分组，同一组内按角色优先级排序
     * - 高优先级角色的 DENY 覆盖低优先级角色的 ALLOW
     * - 同级规则使用 OR 合并
     */
    public Map<String, String> mergeRules(List<PermissionInfo> permissions, Map<Long, RoleInfo> roleMap) {
        // 1. 按 (表, 权限码) 分组
        Map<String, Map<String, List<PermissionInfo>>> grouped = groupByTableAndPerm(permissions);
        
        // 2. 冲突解决：高优先级DENY覆盖低优先级ALLOW
        Map<String, Map<String, List<PermissionInfo>>> resolved = resolveConflicts(grouped, roleMap);
        
        // 3. 分别构建 ALLOW 和 DENY 规则
        Map<String, List<String>> allowMap = new HashMap<>();
        Map<String, List<String>> denyMap = new HashMap<>();
        
        for (Map.Entry<String, Map<String, List<PermissionInfo>>> tableEntry : resolved.entrySet()) {
            String table = tableEntry.getKey();
            for (List<PermissionInfo> list : tableEntry.getValue().values()) {
                for (PermissionInfo p : list) {
                    if (p.getEffect() == Effect.ALLOW) {
                        allowMap.computeIfAbsent(table, k -> new ArrayList<>())
                                .add(p.getRowRuleTemplate().get(table));
                    } else {
                        denyMap.computeIfAbsent(table, k -> new ArrayList<>())
                                .add(p.getRowRuleTemplate().get(table));
                    }
                }
            }
        }

        // 4. 生成最终规则（V14.0 修复纯DENY逻辑）
        Map<String, String> result = new HashMap<>();
        for (String table : allowMap.keySet()) {
            String allowPart = allowMap.get(table).stream()
                .map(r -> "(" + r + ")")
                .collect(Collectors.joining(" OR "));
            if (denyMap.containsKey(table)) {
                String denyPart = denyMap.get(table).stream()
                    .map(r -> "(" + r + ")")
                    .collect(Collectors.joining(" OR "));
                result.put(table, "(" + allowPart + ") AND NOT (" + denyPart + ")");
            } else {
                result.put(table, allowPart);
            }
        }
        return result;
    }
}
```

### 6.3 通配符展开 + 权限存在性校验（V14.0 修正）

```java
// 位于 a2a-permission-admin 模块 engine 包
package io.github.latcn.a2a.permission.admin.engine;

@Component
public class WildcardExpander {
    
    @Autowired
    private PermissionMapper permissionMapper;

    public List<PermissionInfo> expand(List<PermissionInfo> permissions) {
        Set<String> definedPerms = permissionMapper.selectAllPermissionCodes();
        List<PermissionInfo> expanded = new ArrayList<>();
        
        for (PermissionInfo perm : permissions) {
            String code = perm.getPermissionCode();
            if (code.endsWith(":*")) {
                String prefix = code.substring(0, code.length() - 2);
                for (String defined : definedPerms) {
                    if (defined.startsWith(prefix + ":") && !defined.equals(code)) {
                        PermissionInfo cloned = cloneWithNewCode(perm, defined);
                        expanded.add(cloned);
                    }
                }
            } else {
                expanded.add(perm);
            }
        }
        return expanded;
    }
}
```

### 6.4 恒真/恒假短路优化（V14.0 新增）

```java
// 位于 a2a-permission-admin 模块 engine 包
package io.github.latcn.a2a.permission.admin.engine;

@Component
public class ExpressionOptimizer {
    public Map<String, String> optimize(Map<String, String> rowRules) {
        Map<String, String> optimized = new HashMap<>();
        for (Map.Entry<String, String> entry : rowRules.entrySet()) {
            String trimmed = entry.getValue().replaceAll("\\s+", "");
            if (trimmed.equals("1=0") || trimmed.equals("(1=0)")) {
                optimized.put(entry.getKey(), "1=0");
            } else if (trimmed.equals("1=1") || trimmed.equals("(1=1)")) {
                optimized.put(entry.getKey(), "1=1");
            } else {
                optimized.put(entry.getKey(), entry.getValue());
            }
        }
        return optimized;
    }
}
```

### 6.5 强制规则注入（V14.0 修正获取逻辑）

```java
// 位于 a2a-permission-admin 模块 engine 包
package io.github.latcn.a2a.permission.admin.engine;

@Component
public class MandatoryRuleInjector {
    
    @Autowired
    private PermissionMapper permissionMapper;

    public String inject(String businessRule, String permCode) {
        if (businessRule == null || "1=0".equals(businessRule.trim())) {
            return "1=0";
        }
        String mandatoryRule = permissionMapper.selectMandatoryRuleByPermCode(permCode);
        if (mandatoryRule == null || mandatoryRule.trim().isEmpty() || "1=1".equals(mandatoryRule.trim())) {
            return businessRule;
        }
        return "(" + businessRule + ") AND (" + mandatoryRule + ")";
    }
}
```

### 6.6 双版本号聚合计算器（V14.0 MD5）

```java
// 位于 a2a-permission-admin 模块 engine 包
package io.github.latcn.a2a.permission.admin.engine;

@Component
public class CombinedVersionCalculator {
    public String calculate(Long userId, Long userPermVersion, List<RoleInfo> roles) {
        StringBuilder sb = new StringBuilder("u:").append(userPermVersion);
        roles.sort(Comparator.comparing(RoleInfo::getId));
        for (RoleInfo role : roles) {
            sb.append("|r:").append(role.getId()).append(":").append(role.getRoleVersion());
        }
        return DigestUtils.md5DigestAsHex(sb.toString().getBytes(StandardCharsets.UTF_8));
    }
}
```

### 6.7 权限计算主入口（V14.0 Pipeline - 位于 admin 模块）

```java
// 位于 a2a-permission-admin 模块 engine 包
package io.github.latcn.a2a.permission.admin.engine;

@Component
public class PermissionCalculator {
    
    @Autowired
    private UserRoleMapper userRoleMapper;
    @Autowired
    private RolePermissionMapper rolePermissionMapper;
    @Autowired
    private PermissionMapper permissionMapper;
    @Autowired
    private UserMapper userMapper;
    @Autowired
    private WildcardExpander wildcardExpander;
    @Autowired
    private RowRuleMerger rowRuleMerger;
    @Autowired
    private MandatoryRuleInjector mandatoryRuleInjector;
    @Autowired
    private ExpressionOptimizer expressionOptimizer;
    @Autowired
    private CombinedVersionCalculator versionCalculator;

    public UserFullPermissionDTO calculate(Long userId) {
        // 1. 获取用户角色
        List<RoleInfo> roles = userRoleMapper.selectRolesByUserId(userId);
        if (roles.isEmpty()) {
            return UserFullPermissionDTO.builder()
                .userId(userId)
                .permissions(Collections.emptySet())
                .rowRules(Collections.emptyMap())
                .combinedVersion("0_0")
                .build();
        }

        // 2. 获取角色权限配置
        List<Long> roleIds = roles.stream().map(RoleInfo::getId).collect(Collectors.toList());
        List<PermissionInfo> permissions = rolePermissionMapper.selectPermissionsByRoleIds(roleIds);

        // 3. 通配符展开 + 权限存在性校验
        List<PermissionInfo> expanded = wildcardExpander.expand(permissions);

        // 4. 行级规则合并（引入优先级）
        Map<Long, RoleInfo> roleMap = roles.stream().collect(Collectors.toMap(RoleInfo::getId, r -> r));
        Map<String, String> rowRules = rowRuleMerger.mergeRules(expanded, roleMap);

        // 5. 恒真/恒假短路优化
        rowRules = expressionOptimizer.optimize(rowRules);

        // 6. 强制规则注入（按权限码获取）
        Map<String, String> finalRowRules = new HashMap<>();
        for (Map.Entry<String, String> entry : rowRules.entrySet()) {
            String permCode = entry.getKey();
            String injected = mandatoryRuleInjector.inject(entry.getValue(), permCode);
            finalRowRules.put(permCode, injected);
        }

        // 7. 计算双版本号
        Long userPermVersion = userMapper.selectPermVersion(userId);
        String combinedVersion = versionCalculator.calculate(userId, userPermVersion, roles);

        // 8. 提取权限码集合
        Set<String> permCodes = expanded.stream()
            .map(PermissionInfo::getPermissionCode)
            .collect(Collectors.toSet());

        return UserFullPermissionDTO.builder()
            .userId(userId)
            .permissions(permCodes)
            .rowRules(finalRowRules)
            .roles(roles)
            .combinedVersion(combinedVersion)
            .userPermVersion(userPermVersion)
            .roleVersions(roles.stream().collect(Collectors.toMap(RoleInfo::getId, RoleInfo::getRoleVersion)))
            .build();
    }
}
```


## 7. 行级规则引擎（V14.0 位于 admin 模块）

> **V14.0 修正**：安全校验包（`security` 包）和行级规则引擎相关组件从 `remote` 模块移至 `admin` 模块，因为只有控制面需要执行本地安全校验和 SQL 绑定。

### 7.1 核心机制

行级规则在 SQL 执行时动态注入 WHERE 条件。模板绑定在 `t_role_permission` 表上，实现"同权不同数据"。

### 7.2 行级规则模板格式（V14.0 支持子查询 + 表名白名单）

```json
{
  "orders": "salesperson_id = #{user.id} AND status != 'deleted'",
  "order_items": "order_id IN (SELECT id FROM orders WHERE salesperson_id = #{user.id})"
}
```

### 7.3 支持的占位符（V14.0 扩展）

| 占位符 | 数据来源 | 说明 |
|--------|----------|------|
| `#{user.id}` | 用户属性 | 当前用户 ID |
| `#{user.username}` | 用户属性 | 当前用户登录名 |
| `#{user.department}` | 用户属性 | 当前用户部门（单一） |
| `#{user.dept_ids_with_children}` | 用户属性 | 用户所有部门及子部门 ID 集合（用于 IN 查询） |

### 7.4 AST 校验（V14.0 子查询表名白名单 + 系统变量拦截）

```java
// 位于 a2a-permission-admin 模块 security 包
package io.github.latcn.a2a.permission.admin.security;

@Component
public class RowRuleValidator {
    
    @Value("${a2a.permission.row-rule.allowed-subquery-tables:}")
    private Set<String> allowedSubQueryTables;

    private static final Set<String> DANGEROUS_FUNCTIONS = Set.of(
        "SLEEP", "BENCHMARK", "LOAD_FILE", "INTO_OUTFILE",
        "UUID", "RANDOM_BYTES", "UPDATEXML", "EXTRACTVALUE",
        "GET_LOCK", "RELEASE_LOCK"
    );

    public void validate(String ruleTemplate) {
        if (ruleTemplate == null || ruleTemplate.trim().isEmpty()) return;

        try {
            // 检测多语句
            Statements statements = CCJSqlParserUtil.parseStatements("SELECT * FROM dummy WHERE " + ruleTemplate);
            if (statements.size() > 1) {
                throw new SecurityException("禁止多语句执行");
            }

            Select select = (Select) CCJSqlParserUtil.parse("SELECT * FROM dummy WHERE " + ruleTemplate);
            PlainSelect plain = (PlainSelect) select.getSelectBody();
            Expression where = plain.getWhere();
            if (where == null) {
                throw new SecurityException("规则模板必须包含有效的WHERE条件");
            }

            StrictSecurityVisitor visitor = new StrictSecurityVisitor();
            where.accept(visitor);

        } catch (JSQLParserException e) {
            throw new SecurityException("规则模板解析失败: " + ruleTemplate, e);
        }
    }

    private class StrictSecurityVisitor extends ExpressionVisitorAdapter {
        @Override
        public void visit(Variable variable) {
            if (variable.getName() != null && variable.getName().startsWith("@@")) {
                throw new SecurityException("禁止使用系统变量: " + variable.getName());
            }
        }

        @Override
        public void visit(Comment comment) {
            throw new SecurityException("禁止使用 SQL 注释");
        }

        @Override
        public void visit(SubSelect subSelect) {
            if (subSelect.getSelectBody() instanceof PlainSelect) {
                PlainSelect inner = (PlainSelect) subSelect.getSelectBody();
                if (inner.getJoins() != null && !inner.getJoins().isEmpty()) {
                    throw new SecurityException("禁止子查询中使用JOIN");
                }
                if (inner.getSelectItems() == null || inner.getSelectItems().size() != 1) {
                    throw new SecurityException("子查询必须仅返回单列");
                }
                // V14.0 子查询表名白名单
                if (inner.getFromItem() instanceof Table) {
                    String tableName = ((Table) inner.getFromItem()).getName();
                    if (!allowedSubQueryTables.isEmpty() && !allowedSubQueryTables.contains(tableName)) {
                        throw new SecurityException("禁止子查询访问未授权的表: " + tableName);
                    }
                }
                if (inner.getWhere() != null) {
                    inner.getWhere().accept(this);
                }
            }
        }

        @Override
        public void visit(Function function) {
            String name = function.getName().toUpperCase();
            if (DANGEROUS_FUNCTIONS.contains(name)) {
                throw new SecurityException("禁止使用危险函数: " + name);
            }
        }

        @Override
        public void visit(LikeExpression expr) {
            throw new SecurityException("用户行级规则中禁止使用 LIKE 操作符");
        }

        @Override
        public void visit(RegExpExpression expr) {
            throw new SecurityException("禁止使用 REGEXP/RLIKE 操作符");
        }
    }
}
```

### 7.5 预编译绑定实现（V14.0 先校验后绑定）

```java
// 位于 a2a-permission-admin 模块 engine 包
package io.github.latcn.a2a.permission.admin.engine;

@Component
public class RowRulePreparedBinder {
    
    @Autowired
    private RowRuleValidator rowRuleValidator;

    public PreparedSql bindRowRule(String rowRule, UserContextEnricher.UserContext ctx) {
        // 1. 先校验（替换为虚拟值）
        String dummySql = replacePlaceholdersWithDummy(rowRule);
        rowRuleValidator.validate(dummySql);

        // 2. 提取参数并替换为 {0} 占位符
        Pattern pattern = Pattern.compile("#\\{user\\.([^}]+)\\}");
        Matcher matcher = pattern.matcher(rowRule);
        StringBuffer sqlBuffer = new StringBuffer();
        List<Object> params = new ArrayList<>();
        int index = 0;

        while (matcher.find()) {
            String attrName = matcher.group(1);
            Object value = ctx.getAttribute("user." + attrName);
            if (value == null) {
                throw new IllegalArgumentException("缺失用户属性: user." + attrName);
            }
            matcher.appendReplacement(sqlBuffer, "{" + index + "}");
            params.add(value);
            index++;
        }
        matcher.appendTail(sqlBuffer);

        return new PreparedSql(sqlBuffer.toString(), params);
    }

    private String replacePlaceholdersWithDummy(String sql) {
        return sql.replaceAll("#\\{user\\.id\\}", "1")
                  .replaceAll("#\\{user\\.username\\}", "'dummy_user'")
                  .replaceAll("#\\{user\\.dept_ids_with_children\\}", "(1,2,3)");
    }

    @lombok.Value
    public static class PreparedSql {
        String sql;
        List<Object> params;
    }
}
```

### 7.6 部门层级穿透实现（Redis 缓存优化）

```java
// 位于 a2a-permission-admin 模块 engine 包
package io.github.latcn.a2a.permission.admin.engine;

@Component
public class UserContextEnricher {
    
    @Autowired
    private DepartmentMapper deptMapper;
    @Autowired
    private UserMapper userMapper;
    @Autowired
    private RedisCacheManager redisCacheManager;
    
    private static final String DEPT_SUB_CACHE_KEY = "dept:sub:";

    public UserContext enrich(Long userId) {
        List<Long> deptIds = deptMapper.selectDeptIdsByUserId(userId);
        Set<Long> allDeptIds = new HashSet<>(deptIds);
        for (Long deptId : deptIds) {
            String cacheKey = DEPT_SUB_CACHE_KEY + deptId;
            List<Long> subIds = redisCacheManager.getDeptSubIds(deptId);
            if (subIds == null) {
                subIds = deptMapper.selectSubDeptIdsByPath(deptId);
                redisCacheManager.setDeptSubIds(deptId, subIds);
            }
            allDeptIds.addAll(subIds);
        }
        String username = userMapper.selectUsername(userId);
        return UserContext.builder()
            .userId(userId)
            .username(username)
            .deptIdsWithChildren(new ArrayList<>(allDeptIds))
            .build();
    }

    @lombok.Value
    @lombok.Builder
    public static class UserContext {
        Long userId;
        String username;
        List<Long> deptIdsWithChildren;
        
        public Object getAttribute(String key) {
            if ("user.id".equals(key)) return userId;
            if ("user.username".equals(key)) return username;
            if ("user.dept_ids_with_children".equals(key)) return deptIdsWithChildren;
            return null;
        }
    }
}
```

### 7.7 部门 Path 维护（V14.0 并发控制 + 分批级联更新）

```java
// 位于 a2a-permission-admin 模块 service 包
package io.github.latcn.a2a.permission.admin.service;

@Service
@Slf4j
public class DepartmentService {
    
    @Autowired
    private DepartmentMapper deptMapper;
    @Autowired
    private RedissonClient redissonClient;
    
    private static final String DEPT_PATH_LOCK_PREFIX = "dept:path:lock:";
    private static final int BATCH_SIZE = 1000;

    @Transactional
    public void createDepartment(Department dept) {
        if (dept.getParentId() != null) {
            deptMapper.selectForUpdate(dept.getParentId());
        }
        deptMapper.insert(dept);
        String parentPath = dept.getParentId() == null ? "" : deptMapper.selectPath(dept.getParentId());
        String currentPath = parentPath + "/" + dept.getId();
        deptMapper.updatePath(dept.getId(), currentPath);
    }

    @Transactional
    public void moveDepartment(Long deptId, Long newParentId) {
        String lockKey = DEPT_PATH_LOCK_PREFIX + deptId;
        RLock lock = redissonClient.getLock(lockKey);
        try {
            if (!lock.tryLock(5, 10, TimeUnit.SECONDS)) {
                throw new ConcurrentModificationException("部门正在被其他操作修改");
            }
            
            String oldPath = deptMapper.selectPath(deptId);
            String newParentPath = newParentId == null ? "" : deptMapper.selectPath(newParentId);
            String newPath = newParentPath + "/" + deptId;
            
            deptMapper.updateParentAndPath(deptId, newParentId, newPath);
            
            // 分批级联更新子孙部门
            String oldPrefix = oldPath + "/";
            String newPrefix = newPath + "/";
            int offset = 0;
            while (true) {
                List<Long> childIds = deptMapper.selectChildIdsByPathPrefix(oldPrefix, offset, BATCH_SIZE);
                if (childIds.isEmpty()) break;
                deptMapper.batchCascadeUpdatePath(childIds, oldPrefix, newPrefix);
                offset += BATCH_SIZE;
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("获取分布式锁被中断", e);
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```


## 8. 智能数据访问层（Client）

### 8.1 接口定义（a2a-permission-api）- **完全保留**

```java
// 位于 a2a-permission-api 模块 spi 包
package io.github.latcn.a2a.permission.api.spi;

import io.github.latcn.a2a.permission.api.dto.*;

public interface PermissionQueryService {
    TokenExchangePrepareResponse prepareTokenExchange(TokenExchangePrepareRequest request);
    UserFullPermissionDTO getUserFullPermissions(Long userId);
    AgentDTO getAgent(String clientId);
    AclCheckResult checkAcl(String sourceClientId, String targetClientId);
}
```

### 8.2 本地实现（admin 模块）- V14.0 控制面使用

```java
// 位于 a2a-permission-admin 模块 service 包
package io.github.latcn.a2a.permission.admin.service;

@Service
@ConditionalOnProperty(name = "a2a.permission.mode", havingValue = "local")
public class LocalPermissionQueryService implements PermissionQueryService {
    
    @Autowired
    private PermissionCalculator permissionCalculator;

    @Override
    public TokenExchangePrepareResponse prepareTokenExchange(TokenExchangePrepareRequest request) {
        UserFullPermissionDTO perm = permissionCalculator.calculate(request.getUserId());
        return TokenExchangePrepareResponse.builder()
            .userId(perm.getUserId())
            .username(perm.getUsername())
            .combinedVersion(perm.getCombinedVersion())
            .permissions(perm.getPermissions())
            .rowRules(perm.getRowRules())
            .roles(perm.getRoles())
            .build();
    }

    @Override
    public UserFullPermissionDTO getUserFullPermissions(Long userId) {
        return permissionCalculator.calculate(userId);
    }

    @Override
    public AgentDTO getAgent(String clientId) {
        // TODO: 实现 Agent 查询逻辑
        return null;
    }

    @Override
    public AclCheckResult checkAcl(String sourceClientId, String targetClientId) {
        // TODO: 实现 ACL 检查逻辑
        return null;
    }
}
```

### 8.3 远程实现（remote 模块）- V14.0 上层调用方使用

```java
// 位于 a2a-permission-remote 模块 remote 包
package io.github.latcn.a2a.permission.remote.remote;

@FeignClient(
    name = "a2a-permission-admin",
    fallbackFactory = PermissionQueryFallbackFactory.class
)
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

### 8.4 Feign Fallback + 熔断器（remote 模块）

```java
// 位于 a2a-permission-remote 模块 remote 包
package io.github.latcn.a2a.permission.remote.remote;

@Component
@Slf4j
public class PermissionQueryFallback implements PermissionQueryService {
    
    @Autowired
    private RedisCacheManager redisCacheManager;

    @Override
    public UserFullPermissionDTO getUserFullPermissions(Long userId) {
        log.warn("控制面不可用，使用 Redis 缓存降级: userId={}", userId);
        UserFullPermissionDTO cached = redisCacheManager.getUserPermissions(userId);
        if (cached != null) {
            return cached;
        }
        return UserFullPermissionDTO.builder()
            .userId(userId)
            .permissions(Collections.emptySet())
            .rowRules(Collections.emptyMap())
            .combinedVersion("0_0")
            .build();
    }

    @Override
    public TokenExchangePrepareResponse prepareTokenExchange(TokenExchangePrepareRequest request) {
        log.warn("控制面不可用，降级返回最小权限: userId={}", request.getUserId());
        return TokenExchangePrepareResponse.builder()
            .userId(request.getUserId())
            .permissions(Collections.emptySet())
            .rowRules(Collections.emptyMap())
            .combinedVersion("0_0")
            .build();
    }

    @Override
    public AgentDTO getAgent(String clientId) {
        return null;
    }

    @Override
    public AclCheckResult checkAcl(String sourceClientId, String targetClientId) {
        return AclCheckResult.builder()
            .allowed(false)
            .sourceClientId(sourceClientId)
            .targetClientId(targetClientId)
            .reason("服务降级")
            .build();
    }
}
```


## 9. 权限管理操作与审计（V14.0 完整实现）

### 9.1 操作类型枚举（完全保留 V8.7）

```java
// 位于 a2a-permission-api 模块 enums 包
package io.github.latcn.a2a.permission.api.enums;

public enum OperationType {
    ROLE_GRANT, ROLE_REVOKE,
    PERM_GRANT, PERM_REVOKE,
    ROW_RULE_UPDATE,
    ROLE_CREATE, ROLE_UPDATE, ROLE_DELETE,
    PERM_CREATE, PERM_UPDATE, PERM_DELETE,
    ACL_CREATE, ACL_UPDATE, ACL_DELETE,
    AGENT_REGISTER, AGENT_UPDATE, AGENT_SUSPEND, AGENT_ACTIVATE, AGENT_SECRET_ROTATE
}
```

### 9.2 强类型审计 Diff（V14.0 完整定义）

```java
// 位于 a2a-permission-admin 模块 audit 包
package io.github.latcn.a2a.permission.admin.audit;

@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, include = JsonTypeInfo.As.PROPERTY,
              property = "diffType", defaultImpl = ErrorDiff.class)
@JsonSubTypes({
    @JsonSubTypes.Type(value = RoleGrantDiff.class, name = "ROLE_GRANT"),
    @JsonSubTypes.Type(value = PermGrantDiff.class, name = "PERM_GRANT"),
    @JsonSubTypes.Type(value = ErrorDiff.class, name = "ERROR")
})
@Data
public abstract class AuditDiff {
    protected String operationType;
    protected Long operatorId;
    protected Long timestamp;
    protected String result;
}
```

### 9.3 权限管理服务实现（admin 模块）

```java
// 位于 a2a-permission-admin 模块 service 包
package io.github.latcn.a2a.permission.admin.service;

@Service
@Slf4j
@Transactional
public class PermissionAdminService {
    
    @Autowired
    private UserRoleMapper userRoleMapper;
    @Autowired
    private RolePermissionMapper rolePermissionMapper;
    @Autowired
    private UserMapper userMapper;
    @Autowired
    private RoleMapper roleMapper;
    @Autowired
    private AuditLogService auditLogService;
    @Autowired
    private PermissionChangeProducer changeProducer;
    @Autowired
    private ObjectMapper objectMapper;

    public void grantRole(Long userId, Long roleId, Long operatorId) {
        try {
            if (userRoleMapper.exists(userId, roleId)) {
                ErrorDiff diff = new ErrorDiff();
                diff.setReason("用户已拥有该角色");
                diff.setResult("SKIPPED");
                auditLogService.record(AuditLogDTO.builder()
                    .operationType(OperationType.ROLE_GRANT)
                    .operationResult(OperationResult.SKIPPED)
                    .operationDetail(objectMapper.writeValueAsString(diff))
                    .build());
                return;
            }

            Long oldVersion = userMapper.selectPermVersion(userId);
            userRoleMapper.insert(userId, roleId, operatorId);
            
            int updated = userMapper.incrementVersionIfMatch(userId, oldVersion);
            if (updated == 0) {
                ErrorDiff diff = new ErrorDiff();
                diff.setReason("用户数据已被他人修改，请刷新后重试");
                diff.setResult("FAILED");
                auditLogService.recordFailed(AuditLogDTO.builder()
                    .operationType(OperationType.ROLE_GRANT)
                    .operationResult(OperationResult.FAILED)
                    .operationDetail(objectMapper.writeValueAsString(diff))
                    .build());
                throw new ConcurrentModificationException("用户数据已被他人修改，请刷新后重试");
            }
            Long newVersion = userMapper.selectPermVersion(userId);

            RoleGrantDiff diff = new RoleGrantDiff();
            diff.setUserId(userId);
            diff.setAddedRoleIds(Collections.singletonList(roleId));
            diff.setOldUserPermVersion(oldVersion);
            diff.setNewUserPermVersion(newVersion);
            diff.setResult("SUCCESS");
            
            auditLogService.record(AuditLogDTO.builder()
                .operationType(OperationType.ROLE_GRANT)
                .operationResult(OperationResult.SUCCESS)
                .operationDetail(objectMapper.writeValueAsString(diff))
                .build());

            changeProducer.sendUserPermissionChange(userId, newVersion);

        } catch (Exception e) {
            ErrorDiff diff = new ErrorDiff();
            diff.setReason(e.getMessage());
            diff.setResult("FAILED");
            auditLogService.recordFailed(AuditLogDTO.builder()
                .operationType(OperationType.ROLE_GRANT)
                .operationResult(OperationResult.FAILED)
                .operationDetail(objectMapper.writeValueAsString(diff))
                .build());
            throw e;
        }
    }

    // grantPermissionToRole, revokeRole, revokePermissionFromRole 类似实现
}
```


## 10. 缓存与消息机制（V14.0 完整实现）

### 10.1 多级缓存架构

| 层级 | 组件 | TTL | 失效方式 |
|------|------|-----|----------|
| L1 | Caffeine `AsyncLoadingCache` | 10min | TTL 过期 + 版本校验 + RemovalListener 清理索引 |
| L1-辅助 | RoleVersionCache（本地） | 24h | MQ ROLE 消息更新 |
| L1-辅助 | UserVersionCache（本地） | 24h | MQ USER 消息更新 |
| L2 | Redis 缓存 | 24h | 控制面写入时直接更新 |
| L3 | MySQL | - | 数据唯一真实来源 |

### 10.2 反向索引 + RemovalListener（common 模块）

```java
// 位于 a2a-permission-common 模块 cache 包
package io.github.latcn.a2a.permission.common.cache;

@Component
@Slf4j
public class LocalCacheManager {
    
    private final AsyncLoadingCache<String, UserFullPermissionDTO> cache;
    private final Map<Long, Set<String>> userIndex = new ConcurrentHashMap<>();
    private final PermissionQueryService delegate;

    public LocalCacheManager(PermissionQueryService delegate) {
        this.delegate = delegate;
        this.cache = Caffeine.newBuilder()
            .maximumSize(10000)
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .removalListener((String key, UserFullPermissionDTO value, RemovalCause cause) -> {
                if (value != null && value.getUserId() != null) {
                    Long userId = value.getUserId();
                    Set<String> keys = userIndex.get(userId);
                    if (keys != null) {
                        keys.remove(key);
                        if (keys.isEmpty()) {
                            userIndex.remove(userId);
                        }
                    }
                }
            })
            .buildAsync((key, executor) -> {
                Long userId = Long.parseLong(key.substring(key.lastIndexOf(':') + 1));
                UserFullPermissionDTO dto = delegate.getUserFullPermissions(userId);
                if (dto != null) {
                    userIndex.computeIfAbsent(userId, k -> ConcurrentHashMap.newKeySet()).add(key);
                }
                return dto;
            });
    }

    public UserFullPermissionDTO get(Long userId) {
        String cacheKey = "user:perm:" + userId;
        try {
            return cache.get(cacheKey).join();
        } catch (Exception e) {
            log.error("加载用户权限失败: userId={}", userId, e);
            return null;
        }
    }

    public void evictUser(Long userId) {
        Set<String> keys = userIndex.remove(userId);
        if (keys != null && !keys.isEmpty()) {
            cache.synchronous().invalidateAll(keys);
        }
    }
}
```

### 10.3 RocketMQ 订阅者（remote 模块）

```java
// 位于 a2a-permission-remote 模块 subscriber 包
package io.github.latcn.a2a.permission.remote.subscriber;

@Component
@Slf4j
@ConditionalOnProperty(name = "a2a.permission.mode", havingValue = "remote")
public class PermissionChangeSubscriber {
    
    @Autowired
    private LocalCacheManager localCacheManager;
    @Autowired
    private RoleVersionCache roleVersionCache;
    @Autowired
    private UserVersionCache userVersionCache;
    @Autowired
    private ObjectMapper objectMapper;

    @Bean
    public DefaultMQPushConsumer permissionChangeConsumer(
            RocketMQTemplate rocketMQTemplate) throws MQClientException {
        
        String instanceId = ManagementFactory.getRuntimeMXBean().getName().split("@")[0];
        String consumerGroup = "perm-cache-group-" + instanceId;
        
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer(consumerGroup);
        consumer.setNamesrvAddr(rocketMQTemplate.getProducer().getNamesrvAddr());
        consumer.subscribe("permission-change", "USER || ROLE");
        consumer.registerMessageListener((List<MessageExt> msgs, ConsumeConcurrentlyContext ctx) -> {
            for (MessageExt msg : msgs) {
                try {
                    String body = new String(msg.getBody(), StandardCharsets.UTF_8);
                    onMessage(body);
                } catch (Exception e) {
                    log.error("消费消息失败", e);
                    return ConsumeConcurrentlyStatus.RECONSUME_LATER;
                }
            }
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        });
        consumer.start();
        log.info("RocketMQ Consumer started: group={}", consumerGroup);
        return consumer;
    }

    public void onMessage(String messageBody) {
        try {
            PermissionChangeMessage msg = objectMapper.readValue(messageBody, PermissionChangeMessage.class);
            if (msg.getType() == ChangeType.USER) {
                localCacheManager.evictUser(msg.getUserId());
                userVersionCache.evict(msg.getUserId());
            } else if (msg.getType() == ChangeType.ROLE) {
                roleVersionCache.put(msg.getRoleId(), msg.getNewVersion());
            }
        } catch (Exception e) {
            log.error("解析消息失败: {}", messageBody, e);
        }
    }
}
```


## 11. 权限双版本号机制（V14.0 乐观锁）

### 11.1 存储设计

| 维度 | 存储位置 | 更新机制 |
|------|----------|----------|
| 用户维度 | `t_user.perm_version` | 乐观锁：`UPDATE ... SET version = version + 1 WHERE id = #{id} AND version = #{oldVer}` |
| 角色维度 | `t_role.role_version` | 乐观锁：`UPDATE ... SET version = version + 1 WHERE id = #{id} AND version = #{oldVer}` |

### 11.2 Mapper 乐观锁实现（admin 模块 resources/mapper）

```xml
<!-- 位于 a2a-permission-admin 模块 resources/mapper/RoleMapper.xml -->
<update id="incrementVersionIfMatch">
    UPDATE t_role
    SET role_version = role_version + 1, updated_at = NOW()
    WHERE id = #{roleId} AND role_version = #{oldVersion}
</update>

<!-- 位于 a2a-permission-admin 模块 resources/mapper/UserMapper.xml -->
<update id="incrementVersionIfMatch">
    UPDATE t_user
    SET perm_version = perm_version + 1, updated_at = NOW()
    WHERE id = #{userId} AND perm_version = #{oldVersion}
</update>
```

### 11.3 MD5 聚合算法（admin 模块 engine 包）

```java
// 位于 a2a-permission-admin 模块 engine 包
package io.github.latcn.a2a.permission.admin.engine;

@Component
public class CombinedVersionCalculator {
    public String calculate(Long userId, Long userPermVersion, List<RoleInfo> roles) {
        StringBuilder sb = new StringBuilder("u:").append(userPermVersion);
        roles.sort(Comparator.comparing(RoleInfo::getId));
        for (RoleInfo role : roles) {
            sb.append("|r:").append(role.getId()).append(":").append(role.getRoleVersion());
        }
        return DigestUtils.md5DigestAsHex(sb.toString().getBytes(StandardCharsets.UTF_8));
    }
}
```


## 12. 部署与配置

### 12.1 控制面配置（a2a-permission-admin）

```yaml
spring:
  application:
    name: a2a-permission-admin
  datasource:
    url: jdbc:mysql://localhost:3306/a2a_permission_db
    username: ${DB_USER:root}
    password: ${DB_PASSWORD:root}
    driver-class-name: com.mysql.cj.jdbc.Driver
  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}

rocketmq:
  name-server: ${ROCKETMQ_NAMESRV:localhost:9876}
  producer:
    group: permission-change-producer

a2a:
  permission:
    mode: local
    row-rule:
      allowed-subquery-tables: orders,order_items,customers,products

resilience4j:
  circuitbreaker:
    instances:
      permissionService:
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 3
        slidingWindowSize: 10
```

### 12.2 上层调用方配置（认证服务 / Agent 节点）

```yaml
spring:
  application:
    name: authentication-service
  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}

rocketmq:
  name-server: ${ROCKETMQ_NAMESRV:localhost:9876}

a2a:
  permission:
    mode: remote

resilience4j:
  circuitbreaker:
    instances:
      permissionService:
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 3
        slidingWindowSize: 10
```


## 13. 附录

### 13.1 状态字段枚举

| 表 | 字段 | 值 | 含义 |
|----|------|---|------|
| t_user | status | 1 | 正常 |
| | | 2 | 锁定 |
| | | 3 | 禁用 |
| | | 0 | 已删除（逻辑删除） |
| t_role | status | 1 | 有效 |
| | | 0 | 无效 |
| t_agent | status | 1 | 正常 |
| | | 2 | 暂停 |
| | | 3 | 已注销 |

### 13.2 Effect 枚举

| 值 | 含义 |
|----|------|
| 1 | ALLOW（允许） |
| 0 | DENY（拒绝） |

### 13.3 操作类型与版本变更对照表（V14.0 最终）

| 操作类型 | 触发用户版本变更 | 触发角色版本变更 | 更新机制 |
|----------|:---:|:---:|----------|
| `ROLE_GRANT` | ✅ | ❌ | 乐观锁 |
| `ROLE_REVOKE` | ✅ | ❌ | 乐观锁 |
| `PERM_GRANT` | ❌ | ✅ | 乐观锁 |
| `PERM_REVOKE` | ❌ | ✅ | 乐观锁 |
| `ROW_RULE_UPDATE` | ❌ | ✅ | 乐观锁 |
| `ROLE_CREATE` | ❌ | ❌ | N/A |
| `ROLE_UPDATE` | ❌ | ❌ | N/A |
| `ROLE_DELETE` | ❌ | ✅ | 乐观锁 |
| `PERM_CREATE` | ❌ | ❌ | N/A |
| `PERM_UPDATE` | ❌ | ❌ | N/A |
| `PERM_DELETE` | ❌ | ✅ | 乐观锁 |
| `ACL_*` | ❌ | ❌ | N/A |
| `AGENT_*` | ❌ | ❌ | N/A |

### 13.4 模块依赖关系速查表（V14.0 修正）

| 模块 | 依赖 | 被依赖方 | 使用方 | 是否包含本地DB |
|------|------|----------|--------|:---:|
| `a2a-permission-api` | 无 | common, remote, admin | 所有模块 | ❌ |
| `a2a-permission-common` | api | remote, admin | remote, admin | ❌ |
| `a2a-permission-remote` | api, common | 上层调用方 | 上层调用方 | ❌ |
| `a2a-permission-admin` | api, common, MyBatis-Plus | 无 | 独立微服务 | ✅ |
