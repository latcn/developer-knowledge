# Arch-Base 架构底座 技术设计文档

**版本**：v1.0.0 | **状态**：正式发布

## 1. 项目概述

### 1.1 项目定位

Arch-Base 是一个**可插拔、非侵入式**的 Java 架构基础设施集合。它不生产业务逻辑，而是为 DDD（领域驱动设计）、CQRS（命令查询职责分离）提供**可选的、开箱即用的基类、工具包与适配器**。

### 1.2 核心哲学

| 原则                        | 说明                                 |
| :------------------------ | :--------------------------------- |
| **按需取用（Opt-in）**          | 不强迫用户全盘接受，模块可独立引入，所有功能默认关闭，须显式开启   |
| **技术中立（Tech-Agnostic）**   | 核心层（Core）零框架依赖，仅定义 SPI 和标记接口       |
| **防腐隔离（Anti-Corruption）** | 严格区分业务语义（Entity）与技术实现（PO/注解）       |
| **克制务实**                  | 不为架构而架构，能用 Spring 原生机制解决的问题绝不重复造轮子 |

### 1.3 命名规范总览

| 类型         | 命名规则                                   | 示例                                              |
| :--------- | :------------------------------------- | :---------------------------------------------- |
| **接口**     | 前缀 `I`                                 | `ICommand`、`IRepository`、`IQueryBus`            |
| **抽象类**    | 前缀 `Abstract` 或后缀 `Base`               | `AbstractBaseRepository`、`BaseEntity`           |
| **异常类**    | 后缀 `Exception`                         | `BaseException`                                 |
| **SPI 接口** | 前缀 `I`（除 `ExceptionHandler` 作为函数式接口特例） | `IBusInterceptor`、`IRepositoryFactory`          |
| **实现类**    | 后缀说明具体技术                               | `InMemoryCommandBus`、`MybatisRepositoryFactory` |

### 1.4 适用场景

- 需要 DDD 战术建模基类的项目
- 需要 CQRS 读写分离架构的中大型系统
- 希望统一异常处理、分页规范、对象转换规范的技术团队
- 希望保留架构演进空间但又不被框架绑架的项目

### 1.5 项目坐标

| 项目          | 值                          |
| :---------- | :------------------------- |
| **GroupId** | `io.github.latcn`          |
| **根包名**     | `io.github.latcn.archbase` |
| **版本**      | `1.0.0`                    |

## 2. 整体架构原则

### 2.1 模块间依赖关系全景

整个架构遵循**自上而下的严格单向依赖**原则，杜绝循环依赖：

```text
┌─────────────────────────────────────────────────────────────────┐
│                      用户业务应用层 (User Project)              │
└─────────────────────────────┬───────────────────────────────────┘
                              │ (按需引入 Starter)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      archbase-starter                           │
│                  (自动配置路由，条件装配)                         │
└─────────────────────────────┬───────────────────────────────────┘
                              │ (聚合与组装)
        ┌─────────────────────┼─────────────────────┬────────────┐
        ▼                     ▼                     ▼            ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐   ┌──────────────┐
│ archbase-     │   │ archbase-     │   │ archbase-     │   │ archbase-    │
│ web-spring    │   │ data-mybatis  │   │ cqrs          │   │ foundation   │
│ (可选)        │   │ (可选)        │   │ (可选)        │   │ (可选)       │
└───────────────┘   └───────────────┘   └───────────────┘   └──────────────┘
                                                                    │
                                                                    ▼
                                                          ┌──────────────┐
                                                          │ archbase-    │
                                                          │ core         │
                                                          │ (强依赖)     │
                                                          └──────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      archbase-example                          │
│                      (示例模块，依赖 starter)                    │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 核心设计原则

| 原则           | 说明                                                                         |
| :----------- | :------------------------------------------------------------------------- |
| **显式优于隐式**   | 所有功能模块必须通过 `application.yml` 显式配置开启（`enabled: true`），杜绝类路径扫描导致的隐式启动        |
| **上下文传递无感化** | 禁止在核心包中使用 `ThreadLocal` 传递上下文，不适配响应式/虚拟线程；上下文通过方法参数显式传递或使用 Reactor Context |
| **基类最小化**    | 领域基类仅保留业务唯一标识（ID），审计、乐观锁等基础设施关注点通过 AOP 或组合式接口实现，严禁侵入领域层                    |
| **SPI 优先**   | 核心层只定义接口，具体实现由独立的适配器模块提供                                                   |

## 3. 模块划分与详细设计

### 3.1 模块总览

| 模块             | ArtifactId              | 可选性  | 说明                      |
| :------------- | :---------------------- | :--- | :---------------------- |
| 核心契约层          | `archbase-core`         | 强依赖  | 零依赖纯 JDK，定义标记接口、异常、SPI  |
| DDD 地基层        | `archbase-foundation`   | 可选   | DDD 战术建模基类、仓储契约、分页模型    |
| CQRS 引擎层       | `archbase-cqrs`         | 可选   | 命令/查询总线、拦截器 SPI         |
| MyBatis 适配器    | `archbase-data-mybatis` | 可选   | MyBatis-Plus 仓储实现、转换器基类 |
| Spring Web 适配器 | `archbase-web-spring`   | 可选   | 统一响应、全局异常、参数解析          |
| 自动配置入口         | `archbase-starter`      | 推荐   | Spring Boot 自动配置，按需装配   |
| 示例模块           | `archbase-example`      | 仅供参考 | 展示各模块组合使用方式             |

### 3.2 `archbase-core`（核心契约层）

**坐标**：

```xml
<groupId>io.github.latcn</groupId>
<artifactId>archbase-core</artifactId>
<version>1.0.0</version>
```

**依赖**：无（纯 JDK 8+）

**包结构**：

```
io.github.latcn.archbase.core
├── api                          # 标记接口
│   ├── ICommand.java
│   ├── IQuery.java
│   └── IResponse.java
├── exception                    # 异常体系
│   ├── IErrorCode.java
│   ├── BaseException.java
│   └── ErrorCode.java           # 框架内置错误码
└── spi                          # SPI 扩展点
    └── ExceptionHandler.java    # @FunctionalInterface
```

**职责**：定义架构中最顶层的标记接口（Marker Interface）、SPI 抽象和标准异常基类。不包含任何业务实体或技术实现。

#### 核心类/接口详情

| 类/接口               | 类型         | 说明                                                |
| :----------------- | :--------- | :------------------------------------------------ |
| `ICommand`         | 接口（空标记）    | 标记 CQRS 的命令请求对象，所有命令类必须实现                         |
| `IQuery`           | 接口（空标记）    | 标记 CQRS 的查询请求对象，所有查询类必须实现                         |
| `IResponse`        | 接口（空标记）    | 标记 CQRS 的响应对象，所有响应类必须实现                           |
| `IErrorCode`       | 接口         | 业务错误码契约，定义 `getCode()` 和 `getMessage()`           |
| `BaseException`    | 类          | 全局基础异常，支持错误码 + 上下文参数链式设置                          |
| `ErrorCode`        | 枚举         | 框架内置错误码（如 `HANDLER_NOT_FOUND`、`ENTITY_NOT_FOUND`） |
| `ExceptionHandler` | SPI（函数式接口） | **特例**：不加 `I` 前缀，便于 Lambda 使用                     |

#### 伪代码实现

```java
// ===== 标记接口 =====
package io.github.latcn.archbase.core.api;

/**
 * CQRS 命令标记接口
 * 所有命令对象必须实现此接口，表示一个会改变系统状态的请求
 * 
 * @param <R> 命令执行后返回的响应类型
 */
public interface ICommand<R> {
    // 空标记接口，仅用于类型约束
}

/**
 * CQRS 查询标记接口
 * 所有查询对象必须实现此接口，表示一个只读请求
 * 
 * @param <R> 查询返回的响应类型
 */
public interface IQuery<R> {
    // 空标记接口，仅用于类型约束
}

/**
 * CQRS 响应标记接口
 * 所有响应对象必须实现此接口
 */
public interface IResponse {
    // 空标记接口，仅用于类型约束
}

// ===== 错误码契约 =====
package io.github.latcn.archbase.core.exception;

/**
 * 业务错误码接口
 * 业务方通过枚举实现此接口，统一错误码规范
 * 
 * <p>示例：
 * <pre>
 * public enum BizErrorCode implements IErrorCode {
 *     ORDER_NOT_FOUND("B001", "订单不存在"),
 *     INSUFFICIENT_BALANCE("B002", "余额不足");
 *     
 *     private final String code;
 *     private final String message;
 *     // 构造器、getter...
 * }
 * </pre>
 */
public interface IErrorCode {
    /**
     * 获取错误码
     */
    String getCode();
    
    /**
     * 获取错误信息
     */
    String getMessage();
}

// ===== 基础异常 =====
/**
 * 架构基础异常
 * 支持错误码 + 上下文参数链式设置
 * 
 * <p>使用示例：
 * <pre>
 * throw BaseException.of(BizErrorCode.ORDER_NOT_FOUND)
 *         .set("orderId", orderId)
 *         .set("userId", userId);
 * </pre>
 */
public class BaseException extends RuntimeException {
    private final String code;
    private final Map<String, Object> context = new LinkedHashMap<>();

    private BaseException(IErrorCode errorCode, Throwable cause) {
        super(errorCode.getMessage(), cause);
        this.code = errorCode.getCode();
    }

    private BaseException(String code, String message, Throwable cause) {
        super(message, cause);
        this.code = code;
    }

    // ===== 静态工厂方法 =====
    
    /**
     * 根据错误码创建异常
     */
    public static BaseException of(IErrorCode errorCode) {
        return new BaseException(errorCode, null);
    }

    /**
     * 根据错误码创建异常，并包装已有异常（异常链）
     */
    public static BaseException wrap(Throwable cause, IErrorCode errorCode) {
        return new BaseException(errorCode, cause);
    }

    /**
     * 根据错误码和自定义消息创建异常
     */
    public static BaseException of(IErrorCode errorCode, String customMessage) {
        BaseException ex = new BaseException(errorCode, null);
        return ex.set("detail", customMessage);
    }

    /**
     * 根据错误码和消息创建异常（用于动态错误码场景）
     */
    public static BaseException of(String code, String message) {
        return new BaseException(code, message, null);
    }

    // ===== 链式上下文 =====
    
    /**
     * 链式设置上下文参数
     */
    public BaseException set(String key, Object value) {
        this.context.put(key, value);
        return this;
    }

    /**
     * 批量设置上下文
     */
    public BaseException set(Map<String, Object> context) {
        this.context.putAll(context);
        return this;
    }

    // ===== Getter =====
    
    public String getCode() {
        return code;
    }

    public Map<String, Object> getContext() {
        return Collections.unmodifiableMap(context);
    }
}

// ===== 框架内置错误码 =====
/**
 * 框架内置错误码
 */
public enum ErrorCode implements IErrorCode {
    HANDLER_NOT_FOUND("ARCH-001", "未找到对应的处理器"),
    ENTITY_NOT_FOUND("ARCH-002", "实体不存在"),
    INVALID_PARAMETER("ARCH-003", "参数校验失败"),
    SYSTEM_ERROR("ARCH-999", "系统内部错误");

    private final String code;
    private final String message;

    ErrorCode(String code, String message) {
        this.code = code;
        this.message = message;
    }

    @Override
    public String getCode() {
        return code;
    }

    @Override
    public String getMessage() {
        return message;
    }
}

// ===== 异常处理器 SPI =====
package io.github.latcn.archbase.core.spi;

/**
 * 异常处理 SPI
 * 用户可自定义实现，将业务异常转换为统一的响应格式
 * 
 * <p>此接口不加 I 前缀，作为 @FunctionalInterface 便于 Lambda 表达式使用：
 * <pre>
 * ExceptionHandler handler = (throwable) -> {
 *     if (throwable instanceof BaseException) {
 *         return Result.fail(((BaseException) throwable).getCode(), ...);
 *     }
 *     return Result.fail("500", "系统错误");
 * };
 * </pre>
 */
@FunctionalInterface
public interface ExceptionHandler {
    /**
     * 处理异常，返回标准化错误响应
     * 
     * @param exception 捕获的异常
     * @return 统一的错误响应对象（由用户定义类型，如 Result）
     */
    Object handle(Throwable exception);
}
```

### 3.3 `archbase-foundation`（DDD 地基层）

**坐标**：

```xml
<groupId>io.github.latcn</groupId>
<artifactId>archbase-foundation</artifactId>
<version>1.0.0</version>
<dependencies>
    <dependency>
        <groupId>io.github.latcn</groupId>
        <artifactId>archbase-core</artifactId>
        <version>1.0.0</version>
    </dependency>
</dependencies>
```

**依赖**：`archbase-core`

**包结构**：

```
io.github.latcn.archbase.foundation
├── entity                        # 实体基类
│   └── BaseEntity.java
├── valueobject                   # 值对象
│   └── IValueObject.java
├── event                         # 领域事件
│   └── BaseDomainEvent.java
├── repository                    # 仓储契约
│   ├── IRepository.java
│   └── IRepositoryFactory.java
├── pagination                    # 分页模型
│   ├── PageQuery.java
│   └── PageResult.java
└── assembler                     # 转换器 SPI
    └── IAssembler.java
```

**职责**：提供 DDD 战术建模的最小可用基类和通用工具，严格不包含 ORM 依赖。

#### 核心类/接口详情

| 类/接口                      | 类型      | 说明                                              |
| :------------------------ | :------ | :---------------------------------------------- |
| `BaseEntity<ID>`          | 抽象类     | 仅包含 `private ID id;` 及基于 ID 的 `equals/hashCode` |
| `IValueObject`            | 接口（空标记） | 标记值对象，无强制方法                                     |
| `BaseDomainEvent`         | 抽象类     | 包含 `eventId`、`occurredAt`、`sourceId`            |
| `IRepository<Entity, ID>` | 接口      | 定义 `save`、`findById`、`delete` 等纯业务语义方法          |
| `IRepositoryFactory`      | SPI 接口  | 定义 `getMapper(Class)` 等方法，供数据源适配器实现             |
| `IAssembler<Entity, PO>`  | 接口      | 定义 `toPO(Entity)` 和 `toEntity(PO)`              |
| `PageQuery`               | 类       | 分页查询参数（`pageNum`、`pageSize`、排序字段）               |
| `PageResult<T>`           | 类       | 分页结果（`total`、`records`）                         |

#### 伪代码实现

```java
// ===== 基础实体 =====
package io.github.latcn.archbase.foundation.entity;

import java.util.Objects;

/**
 * DDD 基础实体
 * 仅包含 ID 和基于 ID 的相等性判断，不包含技术字段
 * 
 * <p>设计原则：
 * <ul>
 *   <li>领域实体只关心业务标识，不关心数据库主键策略</li>
 *   <li>审计字段（createTime、updateTime）通过 AOP 或 MP 自动填充</li>
 *   <li>乐观锁字段（version）按需通过组合式接口添加</li>
 * </ul>
 * 
 * @param <ID> 实体标识类型
 */
public abstract class BaseEntity<ID> {
    private ID id;

    public ID getId() {
        return id;
    }

    public void setId(ID id) {
        this.id = id;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        BaseEntity<?> that = (BaseEntity<?>) o;
        return Objects.equals(id, that.id);
    }

    @Override
    public int hashCode() {
        return Objects.hashCode(id);
    }

    @Override
    public String toString() {
        return getClass().getSimpleName() + "{id=" + id + "}";
    }
}

// ===== 值对象标记接口 =====
package io.github.latcn.archbase.foundation.valueobject;

/**
 * 值对象标记接口
 * 
 * <p>值对象特征：
 * <ul>
 *   <li>通过属性值判断相等性（而非 ID）</li>
 *   <li>不可变（immutable）</li>
 *   <li>无唯一标识</li>
 * </ul>
 * 
 * <p>示例：
 * <pre>
 * public class Address implements IValueObject {
 *     private final String province;
 *     private final String city;
 *     private final String street;
 *     
 *     public Address(String province, String city, String street) {
 *         this.province = province;
 *         this.city = city;
 *         this.street = street;
 *     }
 *     // getter...
 * }
 * </pre>
 */
public interface IValueObject {
    // 空标记接口
}

// ===== 领域事件基类 =====
package io.github.latcn.archbase.foundation.event;

import java.time.Instant;
import java.util.UUID;

/**
 * 领域事件基类
 * 
 * <p>领域事件记录业务领域中发生的事实，用于：
 * <ul>
 *   <li>聚合间解耦通信</li>
 *   <li>事件溯源（Event Sourcing）</li>
 *   <li>异步后续处理（如发短信、更新统计）</li>
 * </ul>
 */
public abstract class BaseDomainEvent {
    private final String eventId;
    private final Instant occurredAt;
    private final String sourceId;

    protected BaseDomainEvent(String sourceId) {
        this.eventId = UUID.randomUUID().toString();
        this.occurredAt = Instant.now();
        this.sourceId = sourceId;
    }

    protected BaseDomainEvent(String sourceId, String eventId, Instant occurredAt) {
        this.sourceId = sourceId;
        this.eventId = eventId;
        this.occurredAt = occurredAt;
    }

    public String getEventId() {
        return eventId;
    }

    public Instant getOccurredAt() {
        return occurredAt;
    }

    public String getSourceId() {
        return sourceId;
    }

    @Override
    public String toString() {
        return getClass().getSimpleName() + "{eventId=" + eventId + ", sourceId=" + sourceId + "}";
    }
}

// ===== 仓储接口 =====
package io.github.latcn.archbase.foundation.repository;

import io.github.latcn.archbase.foundation.entity.BaseEntity;
import io.github.latcn.archbase.foundation.pagination.PageQuery;
import io.github.latcn.archbase.foundation.pagination.PageResult;

import java.util.List;
import java.util.Optional;

/**
 * 仓储接口（业务语义）
 * 定义聚合根的持久化契约，属于领域层接口，由基础设施层实现
 * 
 * @param <Entity> 聚合根类型（必须继承 BaseEntity）
 * @param <ID>     标识类型
 */
public interface IRepository<Entity extends BaseEntity<ID>, ID> {
    /**
     * 保存实体（新增或更新）
     */
    void save(Entity entity);

    /**
     * 根据 ID 查找实体
     */
    Entity findById(ID id);

    /**
     * 根据 ID 查找实体（返回 Optional）
     */
    Optional<Entity> findOptionalById(ID id);

    /**
     * 删除实体
     */
    void delete(Entity entity);

    /**
     * 根据 ID 删除
     */
    void deleteById(ID id);

    /**
     * 检查实体是否存在
     */
    boolean existsById(ID id);

    /**
     * 查询所有
     */
    List<Entity> findAll();

    /**
     * 分页查询
     */
    PageResult<Entity> pageQuery(PageQuery pageQuery);
}

// ===== 仓储工厂 SPI =====
package io.github.latcn.archbase.foundation.repository;

/**
 * 仓储工厂 SPI
 * 数据源适配器实现此接口，提供 Mapper/DAO 获取能力
 * 
 * <p>用户切换 ORM（如 JPA、R2DBC）时，只需实现此 SPI，
 * 框架会自动适配不同数据源的访问方式。
 */
public interface IRepositoryFactory {
    /**
     * 获取 Mapper/DAO 实例
     * 
     * @param mapperClass Mapper 接口类型
     * @param <T> Mapper 类型
     * @return Mapper 实例
     */
    <T> T getMapper(Class<T> mapperClass);

    /**
     * 获取实体转换器
     */
    <Entity, PO> IAssembler<Entity, PO> getAssembler();
}

// ===== 转换器接口 =====
package io.github.latcn.archbase.foundation.assembler;

/**
 * 实体 <-> PO 转换器接口
 * 
 * <p>推荐使用 MapStruct 实现此接口，编译期生成实现类，无反射损耗：
 * <pre>
 * {@code @Mapper}(componentModel = "spring")
 * public interface OrderAssembler extends IAssembler<Order, OrderPO> {
 *     // 自动生成实现
 * }
 * </pre>
 * 
 * @param <Entity> 领域实体类型
 * @param <PO>     持久化对象类型
 */
public interface IAssembler<Entity, PO> {
    /**
     * 实体转 PO
     */
    PO toPO(Entity entity);

    /**
     * PO 转实体
     */
    Entity toEntity(PO po);

    /**
     * 批量转换：实体列表转 PO 列表
     */
    default List<PO> toPOList(List<Entity> entities) {
        return entities.stream()
                .map(this::toPO)
                .collect(Collectors.toList());
    }

    /**
     * 批量转换：PO 列表转实体列表
     */
    default List<Entity> toEntityList(List<PO> pos) {
        return pos.stream()
                .map(this::toEntity)
                .collect(Collectors.toList());
    }
}

// ===== 分页查询参数 =====
package io.github.latcn.archbase.foundation.pagination;

/**
 * 分页查询参数
 * 
 * <p>置于 foundation 而非 core 的原因：
 * 分页是实现细节（Offset/Limit 模型），
 * 不应污染零依赖的核心契约
 */
public class PageQuery {
    /** 页码，从 1 开始 */
    private int pageNum = 1;
    /** 每页大小，默认 10 */
    private int pageSize = 10;
    /** 排序字段 */
    private String sortField;
    /** 排序方向：ASC / DESC */
    private String sortOrder;

    public static PageQuery of(int pageNum, int pageSize) {
        PageQuery query = new PageQuery();
        query.pageNum = Math.max(1, pageNum);
        query.pageSize = Math.max(1, Math.min(100, pageSize));
        return query;
    }

    public static PageQuery of(int pageNum, int pageSize, String sortField, String sortOrder) {
        PageQuery query = of(pageNum, pageSize);
        query.sortField = sortField;
        query.sortOrder = sortOrder;
        return query;
    }

    // getter/setter...
    public int getPageNum() { return pageNum; }
    public void setPageNum(int pageNum) { this.pageNum = Math.max(1, pageNum); }
    public int getPageSize() { return pageSize; }
    public void setPageSize(int pageSize) { this.pageSize = Math.max(1, Math.min(100, pageSize)); }
    public String getSortField() { return sortField; }
    public void setSortField(String sortField) { this.sortField = sortField; }
    public String getSortOrder() { return sortOrder; }
    public void setSortOrder(String sortOrder) { this.sortOrder = sortOrder; }

    /**
     * 计算偏移量（用于数据库查询）
     */
    public long getOffset() {
        return (long) (pageNum - 1) * pageSize;
    }
}

// ===== 分页结果 =====
package io.github.latcn.archbase.foundation.pagination;

import java.util.Collections;
import java.util.List;

/**
 * 分页结果
 */
public class PageResult<T> {
    private long total;
    private List<T> records;

    public PageResult() {
        this.records = Collections.emptyList();
    }

    public PageResult(long total, List<T> records) {
        this.total = total;
        this.records = records != null ? records : Collections.emptyList();
    }

    public static <T> PageResult<T> of(long total, List<T> records) {
        return new PageResult<>(total, records);
    }

    public static <T> PageResult<T> empty() {
        return new PageResult<>(0, Collections.emptyList());
    }

    // getter/setter...
    public long getTotal() { return total; }
    public void setTotal(long total) { this.total = total; }
    public List<T> getRecords() { return records; }
    public void setRecords(List<T> records) { this.records = records != null ? records : Collections.emptyList(); }

    /**
     * 判断是否为空结果
     */
    public boolean isEmpty() {
        return total == 0 || records.isEmpty();
    }

    /**
     * 映射转换
     */
    public <R> PageResult<R> map(Function<T, R> mapper) {
        List<R> mapped = records.stream()
                .map(mapper)
                .collect(Collectors.toList());
        return new PageResult<>(total, mapped);
    }
}
```

### 3.4 `archbase-cqrs`（CQRS 引擎层）

**坐标**：

```xml
<groupId>io.github.latcn</groupId>
<artifactId>archbase-cqrs</artifactId>
<version>1.0.0</version>
<dependencies>
    <dependency>
        <groupId>io.github.latcn</groupId>
        <artifactId>archbase-core</artifactId>
        <version>1.0.0</version>
    </dependency>
    <dependency>
        <groupId>io.github.latcn</groupId>
        <artifactId>archbase-foundation</artifactId>
        <version>1.0.0</version>
        <optional>true</optional>
    </dependency>
</dependencies>
```

**依赖**：`archbase-core`（可选依赖 `archbase-foundation`）

**包结构**：

```
io.github.latcn.archbase.cqrs
├── bus                           # 总线接口
│   ├── ICommandBus.java
│   ├── IQueryBus.java
│   ├── ICommandHandler.java
│   └── IQueryHandler.java
├── interceptor                   # 拦截器
│   ├── IBusInterceptor.java
│   └── BusInvocation.java
├── config                        # 配置 SPI
│   ├── IBusConfigurer.java
│   ├── CommandBusConfigurer.java
│   └── QueryBusConfigurer.java
├── impl                          # 默认实现
│   ├── InMemoryCommandBus.java
│   └── InMemoryQueryBus.java
└── registry                      # 处理器注册
    └── HandlerRegistry.java
```

**职责**：提供同步内存总线作为默认实现，通过 Pipeline（责任链）模式支持统一拦截，同时暴露异步/分布式扩展 SPI。**作为高级可选组件，不强制替代传统 AOP。**

#### 核心类/接口详情

| 类/接口                    | 类型     | 说明                          |
| :---------------------- | :----- | :-------------------------- |
| `ICommandBus`           | 接口     | 定义 `dispatch(ICommand)` 方法  |
| `IQueryBus`             | 接口     | 定义 `dispatch(IQuery)` 方法    |
| `ICommandHandler<C, R>` | 接口     | 命令处理器，由用户实现                 |
| `IQueryHandler<Q, R>`   | 接口     | 查询处理器，由用户实现                 |
| `IBusInterceptor`       | SPI 接口 | 定义拦截链的编排能力                  |
| `IBusConfigurer`        | SPI 接口 | 允许用户覆盖默认的总线实现               |
| `InMemoryCommandBus`    | 类      | 默认实现，基于 `ConcurrentHashMap` |
| `InMemoryQueryBus`      | 类      | 默认实现，基于 `ConcurrentHashMap` |
| `HandlerRegistry`       | 类      | 自动扫描并注册处理器                  |

#### 伪代码实现

```java
// ===== 命令总线接口 =====
package io.github.latcn.archbase.cqrs.bus;

import io.github.latcn.archbase.core.api.ICommand;
import io.github.latcn.archbase.core.api.IResponse;

/**
 * 命令总线
 * 精准路由 ICommand 到唯一的 ICommandHandler
 * 
 * <p>命令总线 vs 事件总线：
 * <ul>
 *   <li>命令总线：一对一路由，必须有且仅有一个处理器</li>
 *   <li>事件总线：一对多广播，零个或多个监听器</li>
 * </ul>
 * 
 * @see ICommand
 * @see ICommandHandler
 */
public interface ICommandBus {
    /**
     * 分发命令到对应的处理器
     * 
     * @param command 命令对象
     * @param <R> 响应类型
     * @return 处理结果
     * @throws BaseException 若无对应处理器或执行失败
     */
    <R extends IResponse> R dispatch(ICommand<R> command);
}

// ===== 查询总线接口 =====
package io.github.latcn.archbase.cqrs.bus;

import io.github.latcn.archbase.core.api.IQuery;
import io.github.latcn.archbase.core.api.IResponse;

/**
 * 查询总线
 * 精准路由 IQuery 到唯一的 IQueryHandler
 * 
 * <p>查询总线的核心价值：
 * <ul>
 *   <li>统一拦截管道（缓存、日志、监控）</li>
 *   <li>屏蔽数据源细节（MySQL/ES/Redis）</li>
 *   <li>统一异常处理</li>
 * </ul>
 */
public interface IQueryBus {
    /**
     * 分发查询到对应的处理器
     * 
     * @param query 查询对象
     * @param <R> 响应类型
     * @return 查询结果
     * @throws BaseException 若无对应处理器或执行失败
     */
    <R extends IResponse> R dispatch(IQuery<R> query);
}

// ===== 命令处理器接口 =====
package io.github.latcn.archbase.cqrs.bus;

import io.github.latcn.archbase.core.api.ICommand;
import io.github.latcn.archbase.core.api.IResponse;

/**
 * 命令处理器
 * 由业务方实现，包含具体的业务逻辑
 * 
 * @param <C> 命令类型
 * @param <R> 响应类型
 */
@FunctionalInterface
public interface ICommandHandler<C extends ICommand<R>, R extends IResponse> {
    /**
     * 处理命令
     * 
     * @param command 命令对象
     * @return 响应结果
     */
    R handle(C command);
}

// ===== 查询处理器接口 =====
package io.github.latcn.archbase.cqrs.bus;

import io.github.latcn.archbase.core.api.IQuery;
import io.github.latcn.archbase.core.api.IResponse;

/**
 * 查询处理器
 * 由业务方实现，包含具体的查询逻辑
 * 
 * @param <Q> 查询类型
 * @param <R> 响应类型
 */
@FunctionalInterface
public interface IQueryHandler<Q extends IQuery<R>, R extends IResponse> {
    /**
     * 处理查询
     * 
     * @param query 查询对象
     * @return 查询结果
     */
    R handle(Q query);
}

// ===== 总线拦截器 =====
package io.github.latcn.archbase.cqrs.interceptor;

/**
 * 总线拦截器 SPI
 * 
 * <p>应用场景：
 * <ul>
 *   <li>日志记录（统一打印请求/响应）</li>
 *   <li>性能监控（统计执行耗时）</li>
 *   <li>缓存（查询缓存命中则直接返回）</li>
 *   <li>权限校验（统一鉴权）</li>
 * </ul>
 */
@FunctionalInterface
public interface IBusInterceptor {
    /**
     * 拦截总线调用
     * 
     * @param invocation 调用上下文，包含目标处理器和请求对象
     * @param <R> 响应类型
     * @return 处理结果
     * @throws Throwable 拦截器可抛出异常
     */
    <R> R intercept(IBusInvocation<R> invocation) throws Throwable;
}

/**
 * 总线调用上下文
 * 封装处理器、请求对象和拦截链
 */
public interface IBusInvocation<R> {
    /**
     * 继续执行调用链
     */
    R proceed() throws Throwable;

    /**
     * 获取当前请求对象
     */
    Object getRequest();

    /**
     * 获取目标处理器
     */
    Object getHandler();
}

// ===== 内存命令总线实现 =====
package io.github.latcn.archbase.cqrs.impl;

import io.github.latcn.archbase.core.api.ICommand;
import io.github.latcn.archbase.core.api.IResponse;
import io.github.latcn.archbase.core.exception.BaseException;
import io.github.latcn.archbase.core.exception.ErrorCode;
import io.github.latcn.archbase.cqrs.bus.ICommandBus;
import io.github.latcn.archbase.cqrs.bus.ICommandHandler;
import io.github.latcn.archbase.cqrs.interceptor.IBusInterceptor;
import io.github.latcn.archbase.cqrs.interceptor.IBusInvocation;

import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * 内存命令总线默认实现
 * 
 * <p>特点：
 * <ul>
 *   <li>基于 ConcurrentHashMap 存储处理器映射</li>
 *   <li>支持拦截器链（Pipeline 模式）</li>
 *   <li>线程安全</li>
 * </ul>
 */
public class InMemoryCommandBus implements ICommandBus {
    private final Map<Class<?>, ICommandHandler<?, ?>> registry = new ConcurrentHashMap<>();
    private final List<IBusInterceptor> interceptors;

    public InMemoryCommandBus() {
        this(null);
    }

    public InMemoryCommandBus(List<IBusInterceptor> interceptors) {
        this.interceptors = interceptors != null ? interceptors : Collections.emptyList();
    }

    /**
     * 注册命令处理器
     */
    public <C extends ICommand<R>, R extends IResponse> void register(
            Class<C> commandType, ICommandHandler<C, R> handler) {
        registry.put(commandType, handler);
    }

    @SuppressWarnings("unchecked")
    @Override
    public <R extends IResponse> R dispatch(ICommand<R> command) {
        Class<?> commandType = command.getClass();
        ICommandHandler<Object, R> handler = 
                (ICommandHandler<Object, R>) registry.get(commandType);
        
        if (handler == null) {
            throw BaseException.of(ErrorCode.HANDLER_NOT_FOUND)
                    .set("command", commandType.getName());
        }

        // 构建调用链
        CommandInvocation<R> invocation = new CommandInvocation<>(handler, command, interceptors);
        try {
            return invocation.proceed();
        } catch (BaseException e) {
            throw e;
        } catch (Throwable e) {
            throw BaseException.wrap(e, ErrorCode.SYSTEM_ERROR)
                    .set("command", commandType.getName());
        }
    }

    /**
     * 命令调用上下文
     */
    private static class CommandInvocation<R extends IResponse> implements IBusInvocation<R> {
        private final ICommandHandler<Object, R> handler;
        private final ICommand<R> command;
        private final List<IBusInterceptor> interceptors;
        private int currentIndex = 0;

        CommandInvocation(ICommandHandler<Object, R> handler, 
                          ICommand<R> command, 
                          List<IBusInterceptor> interceptors) {
            this.handler = handler;
            this.command = command;
            this.interceptors = interceptors;
        }

        @Override
        public R proceed() throws Throwable {
            if (currentIndex < interceptors.size()) {
                IBusInterceptor interceptor = interceptors.get(currentIndex++);
                return interceptor.intercept(this);
            } else {
                return handler.handle(command);
            }
        }

        @Override
        public Object getRequest() {
            return command;
        }

        @Override
        public Object getHandler() {
            return handler;
        }
    }
}

// ===== 处理器注册器 =====
package io.github.latcn.archbase.cqrs.registry;

import io.github.latcn.archbase.cqrs.bus.ICommandBus;
import io.github.latcn.archbase.cqrs.bus.ICommandHandler;
import io.github.latcn.archbase.cqrs.bus.IQueryBus;
import io.github.latcn.archbase.cqrs.bus.IQueryHandler;
import org.springframework.context.ApplicationContext;
import org.springframework.core.GenericTypeResolver;

import java.util.Map;

/**
 * 处理器注册器
 * 自动扫描 Spring 容器中的 ICommandHandler/IQueryHandler 实现
 */
public class HandlerRegistry {
    private final ApplicationContext applicationContext;
    private final ICommandBus commandBus;
    private final IQueryBus queryBus;

    public HandlerRegistry(ApplicationContext applicationContext, 
                           ICommandBus commandBus, 
                           IQueryBus queryBus) {
        this.applicationContext = applicationContext;
        this.commandBus = commandBus;
        this.queryBus = queryBus;
    }

    @SuppressWarnings("unchecked")
    public void registerAll() {
        // 注册命令处理器
        Map<String, ICommandHandler> commandHandlers = 
                applicationContext.getBeansOfType(ICommandHandler.class);
        for (ICommandHandler handler : commandHandlers.values()) {
            Class<?> commandType = resolveCommandType(handler.getClass());
            if (commandBus instanceof InMemoryCommandBus) {
                ((InMemoryCommandBus) commandBus).register(commandType, handler);
            }
        }

        // 注册查询处理器
        Map<String, IQueryHandler> queryHandlers = 
                applicationContext.getBeansOfType(IQueryHandler.class);
        for (IQueryHandler handler : queryHandlers.values()) {
            // 类似注册逻辑...
        }
    }

    private Class<?> resolveCommandType(Class<?> handlerClass) {
        // 使用 Spring 的 GenericTypeResolver 解析泛型参数
        Class<?>[] genericTypes = GenericTypeResolver.resolveTypeArguments(
                handlerClass, ICommandHandler.class);
        return genericTypes != null && genericTypes.length > 0 ? genericTypes[0] : null;
    }
}
```

### 3.5 `archbase-data-mybatis`（MyBatis-Plus 适配器层）

**坐标**：

```xml
<groupId>io.github.latcn</groupId>
<artifactId>archbase-data-mybatis</artifactId>
<version>1.0.0</version>
<dependencies>
    <dependency>
        <groupId>io.github.latcn</groupId>
        <artifactId>archbase-foundation</artifactId>
        <version>1.0.0</version>
    </dependency>
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-boot-starter</artifactId>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

**依赖**：`archbase-foundation` + `com.baomidou:mybatis-plus-boot-starter`（provided）

**包结构**：

```
io.github.latcn.archbase.data.mybatis
├── repository                     # 仓储实现
│   └── AbstractBaseRepository.java
├── assembler                      # 转换器
│   └── IAssembler.java            # 已移至 foundation，此处保留引用
├── factory                        # 仓储工厂
│   └── MybatisRepositoryFactory.java
├── handler                        # 审计填充
│   └── AutoFillMetaObjectHandler.java
└── page                           # 分页适配
    └── MybatisPageHelper.java
```

**职责**：将 `foundation` 的抽象仓储落地为 MyBatis-Plus 的具体实现，实现对象转换与自动填充。

#### 核心类/接口详情

| 类/接口                                     | 类型  | 说明                                                |
| :--------------------------------------- | :-- | :------------------------------------------------ |
| `AbstractBaseRepository<Entity, PO, ID>` | 抽象类 | 实现 `IRepository`，内部注入 `BaseMapper` 和 `IAssembler` |
| `MybatisRepositoryFactory`               | 类   | 实现 `IRepositoryFactory` SPI                       |
| `AutoFillMetaObjectHandler`              | 类   | 通过 MP 的元对象处理器自动填充审计字段                             |
| `MybatisPageHelper`                      | 工具类 | MP 分页与 PageQuery 互转工具                             |

#### 伪代码实现

```java
// ===== 基础仓储实现 =====
package io.github.latcn.archbase.data.mybatis.repository;

import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import io.github.latcn.archbase.core.exception.BaseException;
import io.github.latcn.archbase.core.exception.ErrorCode;
import io.github.latcn.archbase.foundation.assembler.IAssembler;
import io.github.latcn.archbase.foundation.entity.BaseEntity;
import io.github.latcn.archbase.foundation.pagination.PageQuery;
import io.github.latcn.archbase.foundation.pagination.PageResult;
import io.github.latcn.archbase.foundation.repository.IRepository;
import io.github.latcn.archbase.data.mybatis.page.MybatisPageHelper;
import org.springframework.beans.factory.annotation.Autowired;

import java.util.List;
import java.util.Optional;
import java.util.function.Consumer;

/**
 * 基础仓储实现抽象类
 * 封装 Entity <-> PO 转换及通用 CRUD
 * 
 * <p>子类只需继承此抽象类，并指定 Entity、PO、ID 类型即可，
 * 无需编写任何 CRUD 样板代码。
 * 
 * @param <Entity>  领域实体类型（继承 BaseEntity）
 * @param <PO>      持久化对象类型（MyBatis-Plus 映射类）
 * @param <ID>      标识类型
 */
public abstract class AbstractBaseRepository<Entity extends BaseEntity<ID>, PO, ID>
        implements IRepository<Entity, ID> {

    @Autowired
    protected BaseMapper<PO> baseMapper;

    @Autowired
    protected IAssembler<Entity, PO> assembler;

    // ===== 基础 CRUD =====

    @Override
    public void save(Entity entity) {
        PO po = assembler.toPO(entity);
        if (entity.getId() == null) {
            baseMapper.insert(po);
            setIdToEntity(entity, po);
        } else {
            baseMapper.updateById(po);
        }
    }

    @Override
    public Entity findById(ID id) {
        PO po = baseMapper.selectById(id);
        if (po == null) {
            throw BaseException.of(ErrorCode.ENTITY_NOT_FOUND)
                    .set("id", id)
                    .set("entity", getEntityClass().getSimpleName());
        }
        return assembler.toEntity(po);
    }

    @Override
    public Optional<Entity> findOptionalById(ID id) {
        try {
            return Optional.of(findById(id));
        } catch (BaseException e) {
            if (ErrorCode.ENTITY_NOT_FOUND.getCode().equals(e.getCode())) {
                return Optional.empty();
            }
            throw e;
        }
    }

    @Override
    public void delete(Entity entity) {
        PO po = assembler.toPO(entity);
        baseMapper.deleteById(po.getId());
    }

    @Override
    public void deleteById(ID id) {
        baseMapper.deleteById(id);
    }

    @Override
    public boolean existsById(ID id) {
        return baseMapper.selectCount(
                new LambdaQueryWrapper<PO>().eq(getIdFieldName(), id)
        ) > 0;
    }

    @Override
    public List<Entity> findAll() {
        List<PO> pos = baseMapper.selectList(null);
        return assembler.toEntityList(pos);
    }

    @Override
    public PageResult<Entity> pageQuery(PageQuery pageQuery) {
        Page<PO> page = MybatisPageHelper.toPage(pageQuery);
        LambdaQueryWrapper<PO> wrapper = buildQueryWrapper(pageQuery);
        Page<PO> result = baseMapper.selectPage(page, wrapper);
        List<Entity> entities = assembler.toEntityList(result.getRecords());
        return PageResult.of(result.getTotal(), entities);
    }

    // ===== 扩展分页方法 =====

    /**
     * 带条件的分页查询（供子类使用）
     */
    protected PageResult<Entity> pageQuery(PageQuery pageQuery, 
            Consumer<LambdaQueryWrapper<PO>> conditionBuilder) {
        Page<PO> page = MybatisPageHelper.toPage(pageQuery);
        LambdaQueryWrapper<PO> wrapper = new LambdaQueryWrapper<>();
        conditionBuilder.accept(wrapper);
        // 处理排序
        MybatisPageHelper.applySort(pageQuery, wrapper);
        Page<PO> result = baseMapper.selectPage(page, wrapper);
        List<Entity> entities = assembler.toEntityList(result.getRecords());
        return PageResult.of(result.getTotal(), entities);
    }

    // ===== 可重写方法 =====

    /**
     * 构建默认查询条件（可被子类重写）
     */
    protected LambdaQueryWrapper<PO> buildQueryWrapper(PageQuery pageQuery) {
        LambdaQueryWrapper<PO> wrapper = new LambdaQueryWrapper<>();
        MybatisPageHelper.applySort(pageQuery, wrapper);
        return wrapper;
    }

    /**
     * 设置 ID 到实体（用于新增后回写 ID）
     * 子类可重写此方法优化性能
     */
    protected void setIdToEntity(Entity entity, PO po) {
        // 默认通过反射设置 ID
        try {
            Method setIdMethod = entity.getClass().getMethod("setId", Object.class);
            Method getIdMethod = po.getClass().getMethod("getId");
            Object id = getIdMethod.invoke(po);
            setIdMethod.invoke(entity, id);
        } catch (Exception e) {
            // 忽略，由子类处理
        }
    }

    /**
     * 获取实体类型（用于错误信息）
     */
    protected Class<Entity> getEntityClass() {
        // 通过泛型解析
        return (Class<Entity>) GenericTypeResolver.resolveTypeArguments(
                getClass(), AbstractBaseRepository.class)[0];
    }

    /**
     * 获取 ID 字段名（用于 existsById）
     */
    protected String getIdFieldName() {
        return "id";
    }
}

// ===== MyBatis 分页工具 =====
package io.github.latcn.archbase.data.mybatis.page;

import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import io.github.latcn.archbase.foundation.pagination.PageQuery;

import java.util.function.Consumer;

/**
 * MyBatis-Plus 分页工具
 * 提供 PageQuery <-> MP Page 的互转
 */
public final class MybatisPageHelper {

    private MybatisPageHelper() {}

    /**
     * 将 PageQuery 转换为 MP Page
     */
    public static <T> Page<T> toPage(PageQuery pageQuery) {
        return new Page<>(pageQuery.getPageNum(), pageQuery.getPageSize());
    }

    /**
     * 应用排序到 QueryWrapper
     */
    public static <T> void applySort(PageQuery pageQuery, LambdaQueryWrapper<T> wrapper) {
        String sortField = pageQuery.getSortField();
        String sortOrder = pageQuery.getSortOrder();
        if (sortField != null && !sortField.isEmpty()) {
            if ("DESC".equalsIgnoreCase(sortOrder)) {
                wrapper.orderByDesc(sortField);
            } else {
                wrapper.orderByAsc(sortField);
            }
        }
    }
}

// ===== MyBatis 仓储工厂 =====
package io.github.latcn.archbase.data.mybatis.factory;

import io.github.latcn.archbase.foundation.assembler.IAssembler;
import io.github.latcn.archbase.foundation.repository.IRepositoryFactory;
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;

/**
 * MyBatis-Plus 仓储工厂实现
 */
@Component
public class MybatisRepositoryFactory implements IRepositoryFactory, ApplicationContextAware {

    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    @SuppressWarnings("unchecked")
    @Override
    public <T> T getMapper(Class<T> mapperClass) {
        return applicationContext.getBean(mapperClass);
    }

    @SuppressWarnings("unchecked")
    @Override
    public <Entity, PO> IAssembler<Entity, PO> getAssembler() {
        return applicationContext.getBean(IAssembler.class);
    }
}

// ===== 审计字段自动填充 =====
package io.github.latcn.archbase.data.mybatis.handler;

import com.baomidou.mybatisplus.core.handlers.MetaObjectHandler;
import org.apache.ibatis.reflection.MetaObject;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;

/**
 * MyBatis-Plus 元对象填充器
 * 自动填充 createTime、updateTime、isDeleted 等审计字段
 * 
 * <p>不侵入领域实体，仅对 PO 层生效
 */
@Component
public class AutoFillMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        // 自动填充创建时间
        this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "isDeleted", Integer.class, 0);
        // 可扩展：填充创建人
        // this.strictInsertFill(metaObject, "createBy", Long.class, getCurrentUserId());
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        // 自动填充更新时间
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
        // 可扩展：填充更新人
        // this.strictUpdateFill(metaObject, "updateBy", Long.class, getCurrentUserId());
    }

    /**
     * 获取当前用户 ID（子类可重写）
     * 默认返回 null，不填充
     */
    protected Long getCurrentUserId() {
        return null;
    }
}
```

### 3.6 `archbase-web-spring`（Spring Web 适配器层）

**坐标**：

```xml
<groupId>io.github.latcn</groupId>
<artifactId>archbase-web-spring</artifactId>
<version>1.0.0</version>
<dependencies>
    <dependency>
        <groupId>io.github.latcn</groupId>
        <artifactId>archbase-core</artifactId>
        <version>1.0.0</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

**依赖**：`archbase-core` + `org.springframework.boot:spring-boot-starter-web`（provided）

**包结构**：

```
io.github.latcn.archbase.web.spring
├── result                         # 统一响应
│   ├── Result.java
│   └── ResultCode.java
├── exception                      # 异常处理
│   ├── GlobalExceptionHandler.java
│   └── DefaultExceptionHandler.java
├── resolver                       # 参数解析
│   └── RequestParamResolver.java
└── context                        # 上下文（不包含 ThreadLocal）
    └── RequestContext.java        # 显式传参模式
```

**职责**：统一 HTTP 层返回结果、全局异常拦截，并提供响应式支持（WebFlux 适配）。

#### 核心类/接口详情

| 类/接口                      | 类型  | 说明                                     |
| :------------------------ | :-- | :------------------------------------- |
| `Result<T>`               | 类   | 统一响应体工具类（`code`、`msg`、`data`）          |
| `ResultCode`              | 枚举  | 内置响应码（如 `SUCCESS`、`FAIL`）              |
| `GlobalExceptionHandler`  | 抽象类 | 全局异常处理基类，实现 `ExceptionHandler` SPI 的调用 |
| `DefaultExceptionHandler` | 类   | 默认异常处理实现，继承 `GlobalExceptionHandler`   |
| `RequestParamResolver`    | 类   | 支持将 `ICommand`/`IQuery` 自动反序列化         |
| `RequestContext`          | 类   | 请求上下文（显式传参模式，无 ThreadLocal）            |

#### 伪代码实现

```java
// ===== 统一响应 =====
package io.github.latcn.archbase.web.spring.result;

import io.github.latcn.archbase.core.exception.IErrorCode;

/**
 * 统一响应结果
 * 
 * @param <T> 响应数据类型
 */
public class Result<T> {
    private String code;
    private String msg;
    private T data;
    private Long timestamp;

    private Result() {
        this.timestamp = System.currentTimeMillis();
    }

    // ===== 成功响应 =====

    public static <T> Result<T> success(T data) {
        Result<T> result = new Result<>();
        result.code = ResultCode.SUCCESS.getCode();
        result.msg = ResultCode.SUCCESS.getMessage();
        result.data = data;
        return result;
    }

    public static <T> Result<T> success() {
        return success(null);
    }

    // ===== 失败响应 =====

    public static <T> Result<T> fail(String code, String msg) {
        Result<T> result = new Result<>();
        result.code = code;
        result.msg = msg;
        return result;
    }

    public static <T> Result<T> fail(IErrorCode errorCode) {
        return fail(errorCode.getCode(), errorCode.getMessage());
    }

    public static <T> Result<T> fail(IErrorCode errorCode, T data) {
        Result<T> result = fail(errorCode);
        result.data = data;
        return result;
    }

    // ===== 判断 =====

    public boolean isSuccess() {
        return ResultCode.SUCCESS.getCode().equals(code);
    }

    // ===== getter/setter =====

    public String getCode() { return code; }
    public void setCode(String code) { this.code = code; }
    public String getMsg() { return msg; }
    public void setMsg(String msg) { this.msg = msg; }
    public T getData() { return data; }
    public void setData(T data) { this.data = data; }
    public Long getTimestamp() { return timestamp; }
    public void setTimestamp(Long timestamp) { this.timestamp = timestamp; }

    @Override
    public String toString() {
        return "Result{code=" + code + ", msg=" + msg + ", data=" + data + "}";
    }
}

// ===== 内置响应码 =====
package io.github.latcn.archbase.web.spring.result;

/**
 * 内置响应码
 */
public enum ResultCode implements IErrorCode {
    SUCCESS("00000", "操作成功"),
    FAIL("B9999", "操作失败");

    private final String code;
    private final String message;

    ResultCode(String code, String message) {
        this.code = code;
        this.message = message;
    }

    @Override
    public String getCode() {
        return code;
    }

    @Override
    public String getMessage() {
        return message;
    }
}

// ===== 全局异常处理基类 =====
package io.github.latcn.archbase.web.spring.exception;

import io.github.latcn.archbase.core.exception.BaseException;
import io.github.latcn.archbase.core.exception.IErrorCode;
import io.github.latcn.archbase.core.spi.ExceptionHandler;
import io.github.latcn.archbase.web.spring.result.Result;
import io.github.latcn.archbase.web.spring.result.ResultCode;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;

import jakarta.validation.ConstraintViolationException;

/**
 * 全局异常处理基类
 * 
 * <p>子类可重写此基类的方法，自定义特定异常的处理逻辑，
 * 无需重写整个类，避免 Java 单继承限制。
 */
@ControllerAdvice
public abstract class GlobalExceptionHandler {

    protected final Logger log = LoggerFactory.getLogger(getClass());

    /**
     * 自定义异常处理器 SPI
     * 用户可通过注入实现类扩展异常处理逻辑
     */
    @Autowired(required = false)
    private ExceptionHandler customExceptionHandler;

    /**
     * 处理 BaseException
     */
    @ExceptionHandler(BaseException.class)
    @ResponseBody
    public Result<?> handleBaseException(BaseException e) {
        log.warn("BaseException: code={}, msg={}, context={}", 
                e.getCode(), e.getMessage(), e.getContext());
        
        // 优先使用自定义处理器
        if (customExceptionHandler != null) {
            Object result = customExceptionHandler.handle(e);
            if (result instanceof Result) {
                return (Result<?>) result;
            }
        }
        // 默认处理
        return Result.fail(e.getCode(), e.getMessage());
    }

    /**
     * 处理 JSR-303 参数校验异常
     */
    @ExceptionHandler(ConstraintViolationException.class)
    @ResponseBody
    public Result<?> handleConstraintViolation(ConstraintViolationException e) {
        log.warn("Validation failed: {}", e.getMessage());
        String message = e.getConstraintViolations().stream()
                .map(v -> v.getPropertyPath() + ": " + v.getMessage())
                .collect(Collectors.joining("; "));
        return Result.fail(ResultCode.FAIL.getCode(), message);
    }

    /**
     * 处理通用异常
     */
    @ExceptionHandler(Exception.class)
    @ResponseBody
    public Result<?> handleGenericException(Exception e) {
        log.error("Unexpected error", e);
        return Result.fail("500", "系统内部错误");
    }

    /**
     * 子类可重写此方法自定义日志级别
     */
    protected void logError(Exception e) {
        log.error("Exception occurred", e);
    }

    /**
     * 子类可扩展自定义错误码映射
     */
    protected String mapToErrorCode(Exception e) {
        return "500";
    }
}

// ===== 默认异常处理器 =====
package io.github.latcn.archbase.web.spring.exception;

import org.springframework.stereotype.Component;

/**
 * 默认全局异常处理实现
 * 提供开箱即用的异常处理能力
 */
@Component
public class DefaultExceptionHandler extends GlobalExceptionHandler {
    // 直接继承基类，无需额外实现
}

// ===== 参数解析器 =====
package io.github.latcn.archbase.web.spring.resolver;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.github.latcn.archbase.core.api.ICommand;
import io.github.latcn.archbase.core.api.IQuery;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import jakarta.servlet.http.HttpServletRequest;
import java.io.IOException;
import java.util.Map;
import java.util.stream.Collectors;

/**
 * 命令/查询参数解析器
 * 支持将 HTTP Body 或 QueryString 自动反序列化为 ICommand/IQuery
 */
@Component
public class RequestParamResolver {

    private final ObjectMapper objectMapper;

    @Autowired
    public RequestParamResolver(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    /**
     * 从请求中解析命令对象（从 Body 读取 JSON）
     */
    public <T extends ICommand<?>> T resolveCommand(HttpServletRequest request, Class<T> commandType) 
            throws IOException {
        String body = request.getReader().lines().collect(Collectors.joining());
        return objectMapper.readValue(body, commandType);
    }

    /**
     * 从请求中解析查询对象（从 QueryString 或 Body 读取）
     */
    public <T extends IQuery<?>> T resolveQuery(HttpServletRequest request, Class<T> queryType) 
            throws IOException {
        if (isJsonBody(request)) {
            return resolveCommand(request, queryType);
        }
        // 从 QueryString 解析
        Map<String, String[]> params = request.getParameterMap();
        return objectMapper.convertValue(params, queryType);
    }

    private boolean isJsonBody(HttpServletRequest request) {
        String contentType = request.getContentType();
        return contentType != null && contentType.contains("application/json");
    }
}

// ===== 请求上下文（显式传参模式） =====
package io.github.latcn.archbase.web.spring.context;

/**
 * 请求上下文
 * 
 * <p>设计原则：
 * <ul>
 *   <li>不使用 ThreadLocal（不适配 WebFlux 和虚拟线程）</li>
 *   <li>通过方法参数显式传递，或使用 Reactor Context</li>
 *   <li>用户可通过过滤器将上下文存入 Reactor Context</li>
 * </ul>
 * 
 * <p>使用示例（WebFlux）：
 * <pre>
 * Mono.deferContextual(ctx -> {
 *     RequestContext context = ctx.get(RequestContext.class);
 *     return service.doSomething(context.getUserId());
 * });
 * </pre>
 */
public class RequestContext {
    private final String traceId;
    private final Long userId;
    private final String tenantId;

    private RequestContext(Builder builder) {
        this.traceId = builder.traceId;
        this.userId = builder.userId;
        this.tenantId = builder.tenantId;
    }

    public static Builder builder() {
        return new Builder();
    }

    // getter...
    public String getTraceId() { return traceId; }
    public Long getUserId() { return userId; }
    public String getTenantId() { return tenantId; }

    public static class Builder {
        private String traceId;
        private Long userId;
        private String tenantId;

        public Builder traceId(String traceId) {
            this.traceId = traceId;
            return this;
        }

        public Builder userId(Long userId) {
            this.userId = userId;
            return this;
        }

        public Builder tenantId(String tenantId) {
            this.tenantId = tenantId;
            return this;
        }

        public RequestContext build() {
            return new RequestContext(this);
        }
    }
}
```

### 3.7 `archbase-starter`（自动配置入口层）

**坐标**：

```xml
<groupId>io.github.latcn</groupId>
<artifactId>archbase-starter</artifactId>
<version>1.0.0</version>
<dependencies>
    <!-- 核心模块（强依赖） -->
    <dependency>
        <groupId>io.github.latcn</groupId>
        <artifactId>archbase-core</artifactId>
        <version>1.0.0</version>
    </dependency>
    <dependency>
        <groupId>io.github.latcn</groupId>
        <artifactId>archbase-foundation</artifactId>
        <version>1.0.0</version>
    </dependency>
    
    <!-- 可选模块（按需引入） -->
    <dependency>
        <groupId>io.github.latcn</groupId>
        <artifactId>archbase-cqrs</artifactId>
        <version>1.0.0</version>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>io.github.latcn</groupId>
        <artifactId>archbase-data-mybatis</artifactId>
        <version>1.0.0</version>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>io.github.latcn</groupId>
        <artifactId>archbase-web-spring</artifactId>
        <version>1.0.0</version>
        <optional>true</optional>
    </dependency>
</dependencies>
```

**依赖**：所有模块（cqrs、data-mybatis、web-spring 为 optional）

**包结构**：

```
io.github.latcn.archbase.starter
├── autoconfigure                  # 自动配置类
│   ├── ArchBaseAutoConfiguration.java
│   ├── CqrsAutoConfiguration.java
│   ├── MybatisAutoConfiguration.java
│   └── WebAutoConfiguration.java
└── properties                     # 配置属性
    ├── ArchBaseProperties.java
    ├── CqrsProperties.java
    ├── MybatisProperties.java
    └── WebProperties.java
```

**职责**：基于显式配置开关进行模块装配，遵循“按需取用”原则。

#### 核心类详情

| 类                           | 说明                                                                    |
| :-------------------------- | :-------------------------------------------------------------------- |
| `ArchBaseAutoConfiguration` | 主入口，带 `@ConditionalOnProperty(prefix="archbase", name="enabled")`     |
| `CqrsAutoConfiguration`     | 仅当 `archbase.cqrs.enabled=true` 时生效                                   |
| `MybatisAutoConfiguration`  | 仅当 `archbase.mybatis.enabled=true` 时生效                                |
| `WebAutoConfiguration`      | 仅当 `archbase.web.enabled=true` 时生效                                    |
| `ArchBaseProperties`        | 全局配置属性（`archbase.enabled`、`archbase.application-name`）                |
| `CqrsProperties`            | CQRS 配置属性（`archbase.cqrs.enabled`、`archbase.cqrs.mode`）               |
| `MybatisProperties`         | MyBatis 配置属性（`archbase.mybatis.enabled`、`archbase.mybatis.auto-fill`） |
| `WebProperties`             | Web 配置属性（`archbase.web.enabled`、`archbase.web.exception-handler`）     |

#### 伪代码实现

```java
// ===== 配置属性 =====
package io.github.latcn.archbase.starter.properties;

import org.springframework.boot.context.properties.ConfigurationProperties;

/**
 * Arch-Base 全局配置属性
 */
@ConfigurationProperties(prefix = "archbase")
public class ArchBaseProperties {
    /**
     * 是否启用 Arch-Base 功能（全局开关）
     * 默认：false（按需取用）
     */
    private boolean enabled = false;

    /**
     * 应用名称（用于日志、监控）
     */
    private String applicationName;

    // getter/setter...
}

/**
 * CQRS 配置属性
 */
@ConfigurationProperties(prefix = "archbase.cqrs")
public class CqrsProperties {
    /**
     * 是否启用 CQRS 模块
     * 默认：false
     */
    private boolean enabled = false;

    /**
     * 总线模式：memory（内存）、kafka、rabbitmq
     * 默认：memory
     */
    private String mode = "memory";

    // getter/setter...
}

// ===== 主自动配置 =====
package io.github.latcn.archbase.starter.autoconfigure;

import io.github.latcn.archbase.starter.properties.ArchBaseProperties;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;

/**
 * Arch-Base 主自动配置
 * 所有功能默认关闭，须显式启用
 * 
 * <p>配置示例：
 * <pre>
 * archbase:
 *   enabled: true
 *   cqrs:
 *     enabled: true
 *   mybatis:
 *     enabled: true
 *   web:
 *     enabled: true
 * </pre>
 */
@Configuration
@ConditionalOnProperty(prefix = "archbase", name = "enabled", havingValue = "true", matchIfMissing = false)
@EnableConfigurationProperties(ArchBaseProperties.class)
@Import({
    CqrsAutoConfiguration.class,
    MybatisAutoConfiguration.class,
    WebAutoConfiguration.class
})
public class ArchBaseAutoConfiguration {
    // 主配置类，无需额外实现
}

// ===== CQRS 自动配置 =====
package io.github.latcn.archbase.starter.autoconfigure;

import io.github.latcn.archbase.cqrs.bus.ICommandBus;
import io.github.latcn.archbase.cqrs.bus.IQueryBus;
import io.github.latcn.archbase.cqrs.impl.InMemoryCommandBus;
import io.github.latcn.archbase.cqrs.impl.InMemoryQueryBus;
import io.github.latcn.archbase.cqrs.registry.HandlerRegistry;
import io.github.latcn.archbase.starter.properties.CqrsProperties;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.List;

/**
 * CQRS 自动配置
 * 仅当 archbase.cqrs.enabled=true 时生效
 */
@Configuration
@ConditionalOnProperty(prefix = "archbase.cqrs", name = "enabled", havingValue = "true", matchIfMissing = false)
@ConditionalOnClass(name = "io.github.latcn.archbase.cqrs.bus.ICommandBus")
@EnableConfigurationProperties(CqrsProperties.class)
public class CqrsAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public ICommandBus commandBus(List<IBusInterceptor> interceptors) {
        return new InMemoryCommandBus(interceptors);
    }

    @Bean
    @ConditionalOnMissingBean
    public IQueryBus queryBus(List<IBusInterceptor> interceptors) {
        return new InMemoryQueryBus(interceptors);
    }

    /**
     * 处理器注册器
     * 自动扫描并注册 Command/Query Handler
     */
    @Bean
    @ConditionalOnMissingBean
    public HandlerRegistry handlerRegistry(
            ApplicationContext applicationContext,
            ICommandBus commandBus,
            IQueryBus queryBus) {
        HandlerRegistry registry = new HandlerRegistry(applicationContext, commandBus, queryBus);
        registry.registerAll();
        return registry;
    }
}

// ===== MyBatis-Plus 自动配置 =====
package io.github.latcn.archbase.starter.autoconfigure;

import com.baomidou.mybatisplus.core.handlers.MetaObjectHandler;
import io.github.latcn.archbase.data.mybatis.factory.MybatisRepositoryFactory;
import io.github.latcn.archbase.data.mybatis.handler.AutoFillMetaObjectHandler;
import io.github.latcn.archbase.foundation.repository.IRepositoryFactory;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * MyBatis-Plus 自动配置
 * 仅当 archbase.mybatis.enabled=true 时生效
 */
@Configuration
@ConditionalOnProperty(prefix = "archbase.mybatis", name = "enabled", havingValue = "true", matchIfMissing = false)
@ConditionalOnClass(name = "com.baomidou.mybatisplus.core.mapper.BaseMapper")
public class MybatisAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public IRepositoryFactory repositoryFactory() {
        return new MybatisRepositoryFactory();
    }

    @Bean
    @ConditionalOnMissingBean
    public MetaObjectHandler metaObjectHandler() {
        return new AutoFillMetaObjectHandler();
    }
}

// ===== Web 自动配置 =====
package io.github.latcn.archbase.starter.autoconfigure;

import io.github.latcn.archbase.web.spring.exception.DefaultExceptionHandler;
import io.github.latcn.archbase.web.spring.exception.GlobalExceptionHandler;
import io.github.latcn.archbase.web.spring.resolver.RequestParamResolver;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * Web 自动配置
 * 仅当 archbase.web.enabled=true 时生效
 */
@Configuration
@ConditionalOnProperty(prefix = "archbase.web", name = "enabled", havingValue = "true", matchIfMissing = false)
@ConditionalOnClass(name = "org.springframework.web.bind.annotation.ControllerAdvice")
public class WebAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(GlobalExceptionHandler.class)
    public GlobalExceptionHandler globalExceptionHandler() {
        return new DefaultExceptionHandler();
    }

    @Bean
    @ConditionalOnMissingBean
    public RequestParamResolver requestParamResolver(ObjectMapper objectMapper) {
        return new RequestParamResolver(objectMapper);
    }
}
```

### 3.8 `archbase-example`（示例模块）

**坐标**：

```xml
<groupId>io.github.latcn</groupId>
<artifactId>archbase-example</artifactId>
<version>1.0.0</version>
<dependencies>
    <!-- 引入 Starter，按需启用各模块 -->
    <dependency>
        <groupId>io.github.latcn</groupId>
        <artifactId>archbase-starter</artifactId>
        <version>1.0.0</version>
    </dependency>
    <!-- 使用 Spring Boot Web 作为基础 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- 使用 MyBatis-Plus -->
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-boot-starter</artifactId>
    </dependency>
    <!-- H2 内存数据库（示例使用） -->
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

**依赖**：`archbase-starter` + Spring Boot Web + MyBatis-Plus + H2

**包结构**：

```
io.github.latcn.archbase.example
├── ArchBaseExampleApplication.java      # Spring Boot 启动类
├── domain                               # 领域层
│   ├── model
│   │   ├── Order.java                   # 订单实体
│   │   └── OrderItem.java               # 订单项值对象
│   ├── repository
│   │   └── IOrderRepository.java        # 订单仓储接口
│   └── event
│       └── OrderCreatedEvent.java       # 订单创建事件
├── application                          # 应用层
│   ├── command
│   │   ├── CreateOrderCommand.java
│   │   └── PayOrderCommand.java
│   ├── query
│   │   └── OrderPageQuery.java
│   └── handler
│       ├── CreateOrderHandler.java
│       ├── PayOrderHandler.java
│       └── OrderPageQueryHandler.java
├── infrastructure                       # 基础设施层
│   ├── persistence
│   │   ├── po
│   │   │   └── OrderPO.java
│   │   ├── mapper
│   │   │   └── OrderMapper.java
│   │   ├── assembler
│   │   │   └── OrderAssembler.java
│   │   └── repository
│   │       └── OrderRepositoryImpl.java
│   └── config
│       └── MybatisPlusConfig.java
└── interfaces                           # 接口层
    └── controller
        └── OrderController.java
```

**职责**：展示 Arch-Base 各组件的组合使用方式，提供可运行的示例代码。

#### 示例代码

```java
// ===== 启动类 =====
package io.github.latcn.archbase.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ArchBaseExampleApplication {
    public static void main(String[] args) {
        SpringApplication.run(ArchBaseExampleApplication.class, args);
    }
}

// ===== 领域实体 =====
package io.github.latcn.archbase.example.domain.model;

import io.github.latcn.archbase.foundation.entity.BaseEntity;
import io.github.latcn.archbase.core.exception.BaseException;
import io.github.latcn.archbase.example.domain.valueobject.OrderItem;

import java.math.BigDecimal;
import java.util.List;

public class Order extends BaseEntity<Long> {
    private Long userId;
    private BigDecimal amount;
    private OrderStatus status;
    private List<OrderItem> items;

    public static Order create(Long userId, List<OrderItem> items) {
        Order order = new Order();
        order.userId = userId;
        order.items = items;
        order.amount = items.stream()
                .map(OrderItem::getSubtotal)
                .reduce(BigDecimal.ZERO, BigDecimal::add);
        order.status = OrderStatus.PENDING;
        return order;
    }

    public void pay() {
        if (status != OrderStatus.PENDING) {
            throw BaseException.of(ErrorCode.ORDER_STATUS_ERROR)
                    .set("currentStatus", status)
                    .set("expectedStatus", OrderStatus.PENDING);
        }
        this.status = OrderStatus.PAID;
    }

    public void cancel() {
        if (status == OrderStatus.PAID) {
            throw BaseException.of(ErrorCode.ORDER_CANNOT_CANCEL)
                    .set("status", status);
        }
        this.status = OrderStatus.CANCELLED;
    }

    // getter/setter...
}

// ===== 仓储接口 =====
package io.github.latcn.archbase.example.domain.repository;

import io.github.latcn.archbase.foundation.repository.IRepository;
import io.github.latcn.archbase.foundation.pagination.PageQuery;
import io.github.latcn.archbase.foundation.pagination.PageResult;
import io.github.latcn.archbase.example.domain.model.Order;

public interface IOrderRepository extends IRepository<Order, Long> {
    PageResult<Order> queryByUserId(PageQuery pageQuery, Long userId);
}

// ===== 命令 =====
package io.github.latcn.archbase.example.application.command;

import io.github.latcn.archbase.core.api.ICommand;
import io.github.latcn.archbase.example.application.response.OrderIdResponse;

import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotNull;
import java.util.List;

public class CreateOrderCommand implements ICommand<OrderIdResponse> {
    @NotNull
    private Long userId;

    @NotNull
    @Size(min = 1)
    private List<OrderItemDTO> items;

    // getter/setter...
}

// ===== 命令处理器 =====
package io.github.latcn.archbase.example.application.handler;

import io.github.latcn.archbase.cqrs.bus.ICommandHandler;
import io.github.latcn.archbase.example.application.command.CreateOrderCommand;
import io.github.latcn.archbase.example.application.response.OrderIdResponse;
import io.github.latcn.archbase.example.domain.model.Order;
import io.github.latcn.archbase.example.domain.repository.IOrderRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class CreateOrderHandler implements ICommandHandler<CreateOrderCommand, OrderIdResponse> {

    @Autowired
    private IOrderRepository orderRepository;

    @Override
    public OrderIdResponse handle(CreateOrderCommand command) {
        List<OrderItem> items = command.getItems().stream()
                .map(dto -> new OrderItem(dto.getProductId(), dto.getQuantity(), dto.getPrice()))
                .collect(Collectors.toList());

        Order order = Order.create(command.getUserId(), items);
        orderRepository.save(order);

        return new OrderIdResponse(order.getId());
    }
}

// ===== 仓储实现 =====
package io.github.latcn.archbase.example.infrastructure.persistence.repository;

import io.github.latcn.archbase.data.mybatis.repository.AbstractBaseRepository;
import io.github.latcn.archbase.foundation.pagination.PageQuery;
import io.github.latcn.archbase.foundation.pagination.PageResult;
import io.github.latcn.archbase.example.domain.model.Order;
import io.github.latcn.archbase.example.domain.repository.IOrderRepository;
import io.github.latcn.archbase.example.infrastructure.persistence.mapper.OrderMapper;
import io.github.latcn.archbase.example.infrastructure.persistence.po.OrderPO;
import org.springframework.stereotype.Repository;

@Repository
public class OrderRepositoryImpl 
        extends AbstractBaseRepository<Order, OrderPO, Long> 
        implements IOrderRepository {

    @Override
    public PageResult<Order> queryByUserId(PageQuery pageQuery, Long userId) {
        return pageQuery(pageQuery, wrapper -> 
            wrapper.eq(OrderPO::getUserId, userId)
        );
    }
}

// ===== Controller =====
package io.github.latcn.archbase.example.interfaces.controller;

import io.github.latcn.archbase.cqrs.bus.ICommandBus;
import io.github.latcn.archbase.cqrs.bus.IQueryBus;
import io.github.latcn.archbase.web.spring.result.Result;
import io.github.latcn.archbase.example.application.command.CreateOrderCommand;
import io.github.latcn.archbase.example.application.command.PayOrderCommand;
import io.github.latcn.archbase.example.application.query.OrderPageQuery;
import io.github.latcn.archbase.example.application.response.OrderIdResponse;
import io.github.latcn.archbase.example.application.response.OrderPageResponse;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import jakarta.validation.Valid;

@RestController
@RequestMapping("/api/order")
public class OrderController {

    @Autowired
    private ICommandBus commandBus;

    @Autowired
    private IQueryBus queryBus;

    @PostMapping("/create")
    public Result<OrderIdResponse> create(@Valid @RequestBody CreateOrderCommand command) {
        return Result.success(commandBus.dispatch(command));
    }

    @PostMapping("/pay")
    public Result<Void> pay(@Valid @RequestBody PayOrderCommand command) {
        commandBus.dispatch(command);
        return Result.success();
    }

    @GetMapping("/page")
    public Result<OrderPageResponse> page(OrderPageQuery query) {
        return Result.success(queryBus.dispatch(query));
    }
}

// ===== 配置文件 =====
// application.yml
archbase:
  enabled: true
  application-name: archbase-example
  
  cqrs:
    enabled: true
    mode: memory
  
  mybatis:
    enabled: true
    auto-fill: true
  
  web:
    enabled: true
    exception-handler: default
```

## 4. 完整模块依赖关系（Maven 坐标总览）

### 4.1 父工程 POM

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>io.github.latcn</groupId>
    <artifactId>archbase-parent</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <name>Arch-Base Parent</name>
    <description>可插拔的 Java 架构基础设施集合</description>
    <url>https://github.com/latcn/archbase</url>

    <properties>
        <java.version>17</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <spring-boot.version>3.1.5</spring-boot.version>
        <mybatis-plus.version>3.5.3.1</mybatis-plus.version>
        <mapstruct.version>1.5.5.Final</mapstruct.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <!-- Spring Boot BOM -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            
            <!-- Arch-Base 模块 -->
            <dependency>
                <groupId>io.github.latcn</groupId>
                <artifactId>archbase-core</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>io.github.latcn</groupId>
                <artifactId>archbase-foundation</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>io.github.latcn</groupId>
                <artifactId>archbase-cqrs</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>io.github.latcn</groupId>
                <artifactId>archbase-data-mybatis</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>io.github.latcn</groupId>
                <artifactId>archbase-web-spring</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>io.github.latcn</groupId>
                <artifactId>archbase-starter</artifactId>
                <version>${project.version}</version>
            </dependency>
            
            <!-- 第三方依赖版本管理 -->
            <dependency>
                <groupId>com.baomidou</groupId>
                <artifactId>mybatis-plus-boot-starter</artifactId>
                <version>${mybatis-plus.version}</version>
            </dependency>
            <dependency>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct</artifactId>
                <version>${mapstruct.version}</version>
            </dependency>
            <dependency>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct-processor</artifactId>
                <version>${mapstruct.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <modules>
        <module>archbase-core</module>
        <module>archbase-foundation</module>
        <module>archbase-cqrs</module>
        <module>archbase-data-mybatis</module>
        <module>archbase-web-spring</module>
        <module>archbase-starter</module>
        <module>archbase-example</module>
    </modules>
</project>
```

### 4.2 各模块坐标速查

| 模块             | ArtifactId              | 坐标                                                     |
| :------------- | :---------------------- | :----------------------------------------------------- |
| 核心契约层          | `archbase-core`         | `io.github.latcn.archbase:archbase-core:1.0.0`         |
| DDD 地基层        | `archbase-foundation`   | `io.github.latcn.archbase:archbase-foundation:1.0.0`   |
| CQRS 引擎层       | `archbase-cqrs`         | `io.github.latcn.archbase:archbase-cqrs:1.0.0`         |
| MyBatis 适配器    | `archbase-data-mybatis` | `io.github.latcn.archbase:archbase-data-mybatis:1.0.0` |
| Spring Web 适配器 | `archbase-web-spring`   | `io.github.latcn.archbase:archbase-web-spring:1.0.0`   |
| 自动配置入口         | `archbase-starter`      | `io.github.latcn.archbase:archbase-starter:1.0.0`      |
| 示例模块           | `archbase-example`      | `io.github.latcn.archbase:archbase-example:1.0.0`      |

## 5. 使用场景与配置示例

### 5.1 场景一：纯 CRUD + 统一异常处理（无 DDD）

**引入模块**：

```xml
<dependency>
    <groupId>io.github.latcn</groupId>
    <artifactId>archbase-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

**配置**：

```yaml
archbase:
  enabled: true
  web:
    enabled: true
```

### 5.2 场景二：经典 DDD + MyBatis（充血模型）

**引入模块**：

```xml
<dependency>
    <groupId>io.github.latcn</groupId>
    <artifactId>archbase-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

**配置**：

```yaml
archbase:
  enabled: true
  mybatis:
    enabled: true
  web:
    enabled: true
```

### 5.3 场景三：DDD + CQRS 严格读写分离

**引入模块**：

```xml
<dependency>
    <groupId>io.github.latcn</groupId>
    <artifactId>archbase-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

**配置**：

```yaml
archbase:
  enabled: true
  cqrs:
    enabled: true
  mybatis:
    enabled: true
  web:
    enabled: true
```

### 5.4 场景四：单元测试（内存模式，无数据库）

**引入模块**：

```xml
<dependency>
    <groupId>io.github.latcn</groupId>
    <artifactId>archbase-core</artifactId>
    <version>1.0.0</version>
</dependency>
<dependency>
    <groupId>io.github.latcn</groupId>
    <artifactId>archbase-foundation</artifactId>
    <version>1.0.0</version>
</dependency>
<dependency>
    <groupId>io.github.latcn</groupId>
    <artifactId>archbase-cqrs</artifactId>
    <version>1.0.0</version>
</dependency>
```

**配置**：

```yaml
archbase:
  enabled: true
  cqrs:
    enabled: true
```

## 6. 核心设计决策记录（ADR）

| 决策点           | 决策内容                                           | 理由                                                |
| :------------ | :--------------------------------------------- | :------------------------------------------------ |
| **接口命名规范**    | 所有接口统一加 `I` 前缀（`ExceptionHandler` 作为函数式接口特例不加） | 符合 Java 社区常见命名习惯，提高代码可读性                          |
| **Port 接口删除** | 不提供 `Port` 标记接口                                | 六边形架构的约束降级为"架构规范文档"和"推荐目录结构"，全面拥抱 Spring 原生 DI 机制 |
| **分页模型位置**    | 置于 `foundation` 而非 `core`                      | 分页是实现细节（技术栈相关），不应污染零依赖的核心契约                       |
| **条件装配策略**    | 默认 `enabled=false`，显式开启                        | 杜绝"引入即激活"的隐式副作用，严格遵循按需取用                          |
| **上下文传递**     | 舍弃 `ThreadLocal`，改用显式传参或 Reactor Context       | 适配现代响应式编程（WebFlux）与虚拟线程                           |
| **领域基类**      | 剥离审计与乐观锁字段                                     | 领域层应只关心业务规则，技术字段通过 AOP/MP 自动填充                    |
| **多 ORM 支持**  | 官方仅维护 MyBatis-Plus，暴露 `IRepositoryFactory` SPI | 避免框架过于臃肿，允许社区贡献 JPA/R2DBC 适配器                     |
| **CQRS 定位**   | 作为高级可选组件，不强制替代传统 AOP                           | 简单的 CRUD 应用不需要 CQRS，Controller 直接注入 Service 即可    |

## 7. 演进路线

| 版本         | 计划内容                                                                                 |
| :--------- | :----------------------------------------------------------------------------------- |
| **v1.0.0** | 发布 `core`、`foundation`、`data-mybatis`、`web-spring`、`starter`、`example`；提供内存版 CQRS 总线 |
| **v1.1.0** | 提供 `EventStore`（基于 JDBC）SPI 实现，支持事件溯源；完善测试模块 `archbase-test`                         |
| **v1.2.0** | 提供 `archbase-data-jpa` 适配器（JPA/Hibernate 支持）                                         |
| **v2.0.0** | 官方实现 `archbase-data-r2dbc` 适配器；支持 Spring Boot 3.x（Jakarta）全量迁移                       |

## 8. 附录：完整类名索引

| 类/接口                                     | 所属模块         | 类型         | 完整类名                                                                       |
| :--------------------------------------- | :----------- | :--------- | :------------------------------------------------------------------------- |
| `ICommand`                               | core         | 接口         | `io.github.latcn.archbase.core.api.ICommand`                               |
| `IQuery`                                 | core         | 接口         | `io.github.latcn.archbase.core.api.IQuery`                                 |
| `IResponse`                              | core         | 接口         | `io.github.latcn.archbase.core.api.IResponse`                              |
| `IErrorCode`                             | core         | 接口         | `io.github.latcn.archbase.core.exception.IErrorCode`                       |
| `BaseException`                          | core         | 类          | `io.github.latcn.archbase.core.exception.BaseException`                    |
| `ErrorCode`                              | core         | 枚举         | `io.github.latcn.archbase.core.exception.ErrorCode`                        |
| `ExceptionHandler`                       | core         | SPI（函数式接口） | `io.github.latcn.archbase.core.spi.ExceptionHandler`                       |
| `BaseEntity<ID>`                         | foundation   | 抽象类        | `io.github.latcn.archbase.foundation.entity.BaseEntity`                    |
| `IValueObject`                           | foundation   | 接口         | `io.github.latcn.archbase.foundation.valueobject.IValueObject`             |
| `BaseDomainEvent`                        | foundation   | 抽象类        | `io.github.latcn.archbase.foundation.event.BaseDomainEvent`                |
| `IRepository<Entity, ID>`                | foundation   | 接口         | `io.github.latcn.archbase.foundation.repository.IRepository`               |
| `IRepositoryFactory`                     | foundation   | SPI 接口     | `io.github.latcn.archbase.foundation.repository.IRepositoryFactory`        |
| `IAssembler<Entity, PO>`                 | foundation   | 接口         | `io.github.latcn.archbase.foundation.assembler.IAssembler`                 |
| `PageQuery`                              | foundation   | 类          | `io.github.latcn.archbase.foundation.pagination.PageQuery`                 |
| `PageResult<T>`                          | foundation   | 类          | `io.github.latcn.archbase.foundation.pagination.PageResult`                |
| `ICommandBus`                            | cqrs         | 接口         | `io.github.latcn.archbase.cqrs.bus.ICommandBus`                            |
| `IQueryBus`                              | cqrs         | 接口         | `io.github.latcn.archbase.cqrs.bus.IQueryBus`                              |
| `ICommandHandler<C, R>`                  | cqrs         | 接口         | `io.github.latcn.archbase.cqrs.bus.ICommandHandler`                        |
| `IQueryHandler<Q, R>`                    | cqrs         | 接口         | `io.github.latcn.archbase.cqrs.bus.IQueryHandler`                          |
| `IBusInterceptor`                        | cqrs         | SPI 接口     | `io.github.latcn.archbase.cqrs.interceptor.IBusInterceptor`                |
| `IBusInvocation<R>`                      | cqrs         | 接口         | `io.github.latcn.archbase.cqrs.interceptor.IBusInvocation`                 |
| `IBusConfigurer`                         | cqrs         | SPI 接口     | `io.github.latcn.archbase.cqrs.config.IBusConfigurer`                      |
| `InMemoryCommandBus`                     | cqrs         | 类          | `io.github.latcn.archbase.cqrs.impl.InMemoryCommandBus`                    |
| `InMemoryQueryBus`                       | cqrs         | 类          | `io.github.latcn.archbase.cqrs.impl.InMemoryQueryBus`                      |
| `HandlerRegistry`                        | cqrs         | 类          | `io.github.latcn.archbase.cqrs.registry.HandlerRegistry`                   |
| `AbstractBaseRepository<Entity, PO, ID>` | data-mybatis | 抽象类        | `io.github.latcn.archbase.data.mybatis.repository.AbstractBaseRepository`  |
| `MybatisRepositoryFactory`               | data-mybatis | 类          | `io.github.latcn.archbase.data.mybatis.factory.MybatisRepositoryFactory`   |
| `AutoFillMetaObjectHandler`              | data-mybatis | 类          | `io.github.latcn.archbase.data.mybatis.handler.AutoFillMetaObjectHandler`  |
| `MybatisPageHelper`                      | data-mybatis | 工具类        | `io.github.latcn.archbase.data.mybatis.page.MybatisPageHelper`             |
| `Result<T>`                              | web-spring   | 类          | `io.github.latcn.archbase.web.spring.result.Result`                        |
| `ResultCode`                             | web-spring   | 枚举         | `io.github.latcn.archbase.web.spring.result.ResultCode`                    |
| `GlobalExceptionHandler`                 | web-spring   | 抽象类        | `io.github.latcn.archbase.web.spring.exception.GlobalExceptionHandler`     |
| `DefaultExceptionHandler`                | web-spring   | 类          | `io.github.latcn.archbase.web.spring.exception.DefaultExceptionHandler`    |
| `RequestParamResolver`                   | web-spring   | 类          | `io.github.latcn.archbase.web.spring.resolver.RequestParamResolver`        |
| `RequestContext`                         | web-spring   | 类          | `io.github.latcn.archbase.web.spring.context.RequestContext`               |
| `ArchBaseProperties`                     | starter      | 配置类        | `io.github.latcn.archbase.starter.properties.ArchBaseProperties`           |
| `CqrsProperties`                         | starter      | 配置类        | `io.github.latcn.archbase.starter.properties.CqrsProperties`               |
| `MybatisProperties`                      | starter      | 配置类        | `io.github.latcn.archbase.starter.properties.MybatisProperties`            |
| `WebProperties`                          | starter      | 配置类        | `io.github.latcn.archbase.starter.properties.WebProperties`                |
| `ArchBaseAutoConfiguration`              | starter      | 配置类        | `io.github.latcn.archbase.starter.autoconfigure.ArchBaseAutoConfiguration` |
| `CqrsAutoConfiguration`                  | starter      | 配置类        | `io.github.latcn.archbase.starter.autoconfigure.CqrsAutoConfiguration`     |
| `MybatisAutoConfiguration`               | starter      | 配置类        | `io.github.latcn.archbase.starter.autoconfigure.MybatisAutoConfiguration`  |
| `WebAutoConfiguration`                   | starter      | 配置类        | `io.github.latcn.archbase.starter.autoconfigure.WebAutoConfiguration`      |

## 9. 总结

Arch-Base 是一套**防御性极强、演进性极佳**的架构规范实现。它通过 **Core（纯契约）— Foundation（轻基类）— Adapters（可插拔实现）** 的三层结构，确保业务代码不被技术框架绑架。

**核心理念回顾**：

- **按需取用**：所有功能默认关闭，显式开启（`archbase.enabled=true` + 各模块独立开关）
- **接口规范化**：统一 `I` 前缀（`ExceptionHandler` 特例），增强代码可读性
- **SPI 驱动**：核心层只定义接口，实现由适配器模块提供
- **保持克制**：能用 Spring 原生机制解决的问题绝不重复造轮子
- **示例驱动**：提供 `archbase-example` 模块，展示所有组件的组合使用方式

**Maven 坐标速记**：

```xml
<dependency>
    <groupId>io.github.latcn</groupId>
    <artifactId>archbase-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

它能从容应对从**单体架构**到**微服务/响应式架构**的未来迁移，真正实现对使用者 **"0 技术栈绑架，100% 按需取用"** 的承诺。
