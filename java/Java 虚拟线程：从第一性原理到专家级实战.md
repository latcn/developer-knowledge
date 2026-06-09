
## 前言

传统 Java 并发模型中，每个 `java.lang.Thread` 都直接映射到一个操作系统内核线程。当线程执行 I/O 操作时，内核线程进入阻塞状态，宝贵的操作系统资源被无效占用——这是一种极其低效的范式。

虚拟线程（Project Loom）从第一性原理重新思考：**线程的本质是一段可以挂起和恢复的执行上下文**。操作系统线程之所以“重”，是因为它绑定了内核级调度和固定大小的栈内存。如果将调度与栈管理移到用户态，就能实现极高并发。

基于此，Java 虚拟线程的设计目标清晰：**用极轻量的资源实现海量并发，同时保持同步阻塞编程模型**。JDK 21 中虚拟线程正式成为 LTS 的一部分。本文从架构设计、调度原理、核心功能、生产问题、监控调优、最佳实践六个维度全面解读。

---
## 第一章：架构设计

### 1.1 核心概念：四种“线程”的定义

| 概念 | 定义 |
|------|------|
| **操作系统线程** | 内核级线程，由 OS 调度，资源开销大（MB 级栈空间） |
| **平台线程（Platform Thread）** | 传统 `Thread`，一对一映射到 OS 线程 |
| **载体线程（Carrier Thread）** | 实际执行虚拟线程代码的平台线程，是虚拟线程的“肉身” |
| **虚拟线程（Virtual Thread）** | 由 JVM 调度的轻量线程，栈帧存放在堆上，挂载到载体线程上执行 |

### 1.2 M:N 调度模型

M 个虚拟线程映射到 N 个载体线程（通常 N = CPU 核心数），M 可达百万级。

```
┌─────────────────────────────────────────────┐
│          虚拟线程（用户态，海量）              │
│  VT1  VT2  VT3  VT4  VT5  VT6  VT7  VT8 ... │
└─────────────────────────────────────────────┘
                     │ 挂载/卸载
┌─────────────────────────────────────────────┐
│       载体线程（ForkJoinPool Worker）         │
│      Carrier1   Carrier2   Carrier3   ...   │
└─────────────────────────────────────────────┘
                     │ 1:1 映射
┌─────────────────────────────────────────────┐
│            操作系统内核线程                   │
│     OS Thread1  OS Thread2  OS Thread3 ...  │
└─────────────────────────────────────────────┘
```

### 1.3 挂载与卸载机制

- **挂载（Mount）**：将虚拟线程的栈状态恢复到载体线程上执行。
- **卸载（Unmount）**：将虚拟线程的栈状态保存到堆内存，释放载体线程。

触发时机：
- 虚拟线程执行阻塞 I/O（网络、文件）、`Thread.sleep()`、锁等待等操作时，JVM 自动卸载。
- 阻塞操作完成后，虚拟线程重新挂载到任意可用载体线程。

```java
var executor = Executors.newVirtualThreadPerTaskExecutor();
executor.submit(() -> {
    var response = httpClient.send(request);  // 阻塞 -> 自动卸载
    System.out.println(response);              // 恢复后执行
});
```

### 1.4 内存布局

- 栈帧存放在**堆**上，而非线程栈内存。
- 初始内存仅几百字节，动态增长。
- 卸载时栈数据保存到堆，挂载时恢复。

---
## 第二章：调度器原理

### 2.1 为什么选择 ForkJoinPool？

虚拟线程调度器使用**工作窃取（Work-Stealing）的 ForkJoinPool**，配置为**异步模式（FIFO）**。满足：
- 高效任务分发
- 负载均衡
- 低争用
- 与阻塞感知深度集成

### 2.2 FIFO 模式 vs LIFO 模式

- 普通 ForkJoinPool 默认 LIFO，适合分治任务。
- 虚拟线程调度器设置 `asyncMode=true`，本地任务队列采用 FIFO（公平性更好，减少窃取冲突）。

### 2.3 协作式调度

虚拟线程采用**协作式**而非抢占式调度：
- 只有主动让出 CPU（阻塞操作或 `Thread.yield()`）才会卸载。
- 纯 CPU 计算且永不 yield 的虚拟线程会长时间占用载体线程。

### 2.4 调度器并发控制

载体线程数默认 = `Runtime.getRuntime().availableProcessors()`。  
可通过系统参数调整（见第五章）。当所有载体线程被占满时，新任务进入全局队列等待。

---
## 第三章：核心功能

### 3.1 创建虚拟线程

```java
// 方式1：直接启动
Thread.startVirtualThread(() -> doWork());

// 方式2：Builder
Thread.ofVirtual()
      .name("vt-", 0)
      .uncaughtExceptionHandler((t, e) -> log.error("Error", e))
      .start(() -> doWork());

// 方式3：ExecutorService（推荐）
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> doWork());
}
```

### 3.2 结构化并发（预览）

`StructuredTaskScope` 将并发任务生命周期与代码块绑定。

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var userTask = scope.fork(() -> fetchUser());
    var orderTask = scope.fork(() -> fetchOrders());
    scope.join();
    scope.throwIfFailed();
    return new Response(userTask.get(), orderTask.get());
}
```

### 3.3 Scoped Values（预览）

替代 `ThreadLocal`，值不可变、作用域严格、自动清理。

```java
private static final ScopedValue<String> REQUEST_ID = ScopedValue.newInstance();

void handle() {
    ScopedValue.where(REQUEST_ID, "id-123").run(() -> {
        // 内层任意方法可调用 REQUEST_ID.get()
    });
}
```

### 3.4 阻塞操作透明

开发者继续使用同步阻塞风格，虚拟线程自动在阻塞时卸载载体线程。**以同步代码获得异步框架的吞吐能力**。

---
## 第四章：常见生产问题与解决方案

### 4.1 固定问题（Pinning）—— 第一杀手

**定义**：虚拟线程被绑定到载体线程，无法卸载或迁移。

**触发场景**：
- 进入 `synchronized` 块或方法
- 调用 `native` 方法
- 在 `synchronized` 块内执行阻塞操作（如 I/O、`Thread.sleep`）

**危害**：若所有载体线程都被 pinned 且阻塞，系统死锁。

```java
// 危险
public synchronized void dangerous() {
    httpClient.send(request);  // 阻塞 -> 固定载体线程
}

// 安全：使用 ReentrantLock
private final ReentrantLock lock = new ReentrantLock();
public void safe() {
    lock.lock();
    try {
        httpClient.send(request);
    } finally {
        lock.unlock();
    }
}
```

**解决方案**：
1. 立即行动：将 `synchronized` 替换为 `ReentrantLock`，或缩小同步块范围。
2. 短期缓解：增加载体线程数 `-Djdk.virtualThreadScheduler.parallelism=...`。
3. 长期方案：升级到 JDK 25+（JEP 491 实现 `synchronized` 无固定）。
4. 监控：`-Djdk.tracePinnedThreads=full` 定位固定位置。

### 4.2 内存占用过高

**场景一：ThreadLocal 滥用**  
每个虚拟线程有自己的 `ThreadLocalMap`。若同时存在海量虚拟线程，内存占用飙升。但**这不是泄漏**——当虚拟线程死亡，Map 随之回收。真正泄漏是虚拟线程被长期持有（如静态集合）。  
**解决方案**：使用 `ScopedValue` 或局部变量，避免海量线程同时存活。

**场景二：ExecutorService 未关闭**  
```java
// 危险
var executor = Executors.newVirtualThreadPerTaskExecutor();
executor.submit(() -> longTask());
// 忘记关闭 -> 线程持续存在

// 安全
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> longTask());
}
```

### 4.3 CPU 飙升

常见原因：
- **Pinning 导致载体线程饥饿**：pinned 线程长时间占用载体线程，其他虚拟线程无法调度。
- **忙等待**：某些库实现中使用自旋而非阻塞。
- **频繁挂载/卸载**：大量短时阻塞操作导致调度开销增加。

**诊断**：
- `-Djdk.tracePinnedThreads=full` 查看 pinning。
- JFR 监控 `jdk.VirtualThreadPinned` 事件。

### 4.4 死锁

**类型一：Pinning 饱和死锁**  
当所有载体线程都被 pinned 且阻塞等待对方释放锁，系统僵死。

**类型二：混合锁死锁**  
`synchronized` 与 `ReentrantLock` 混用且获取顺序不一致。

**避免策略**：
- 尽量统一使用 `ReentrantLock`。
- 避免在 `synchronized` 块内调用可能阻塞的方法。

### 4.5 阻塞调用不透明

某些阻塞操作会导致意外 pinning，例如：
- `synchronized` 内的 I/O
- 未适配的 JDBC 驱动
- JNI 调用

**解决方案**：升级依赖，使用异步驱动（如 R2DBC），或避免在同步块中阻塞。

### 4.6 虚拟线程池化（反模式）

虚拟线程设计为“每任务新建”，**不应池化**。池化会引入不必要的复杂性，并可能导致资源管理问题。

```java
// ❌ 反模式
ExecutorService pool = Executors.newFixedThreadPool(10);
pool.submit(() -> { ... }); // 平台线程池包装虚拟线程？错误

// ✅ 正确
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> { ... });
}
```

---
## 第五章：调试与监控

### 5.1 关键 JVM 参数

| 参数 | 说明 |
|------|------|
| `-Djdk.tracePinnedThreads=full` | 打印固定线程的完整堆栈 |
| `-Djdk.tracePinnedThreads=short` | 打印概要 |
| `-Djdk.virtualThreadScheduler.parallelism=N` | 载体线程数（默认 CPU 核心数） |
| `-Djdk.virtualThreadScheduler.maxPoolSize=N` | 最大载体线程数（默认 256） |

### 5.2 JFR 事件

- `jdk.VirtualThreadPinned`：固定事件
- `jdk.VirtualThreadStart` / `jdk.VirtualThreadEnd`
- `jdk.VirtualThreadMount` / `jdk.VirtualThreadUnmount`

启用：`-XX:StartFlightRecording:filename=recording.jfr`

### 5.3 线程 Dump

`jstack <pid>` 输出中包含 `<virtual>` 标记的线程。检查是否存在大量 `WAITING` 或 `PARKED` 虚拟线程，以及载体线程状态。

### 5.4 APM 集成

关注指标：虚拟线程总数（泄漏预警）、挂载/卸载速率、pinning 时长、载体线程利用率。

---
## 第六章：性能调优

### 6.1 吞吐量对比

| 场景 | 平台线程 | 虚拟线程 | 优势 |
|------|---------|---------|------|
| 高延迟 I/O（DB + HTTP） | 受限于线程数 | 百万级并发 | 数量级提升 |
| 短连接 HTTP | 上下文切换昂贵 | 无上下文切换 | 3-5 倍 |
| 纯 CPU 密集型 | 略优 | 略逊 | 平台线程更优 |

### 6.2 调优参数

```bash
# 载体线程数（I/O 密集可适度增加，如 CPU*4）
-Djdk.virtualThreadScheduler.parallelism=16
# 最大载体线程数
-Djdk.virtualThreadScheduler.maxPoolSize=32
```

**原则**：默认通常够用。通过 JFR 数据驱动调整。

### 6.3 避免的陷阱

| ❌ 反模式 | ✅ 推荐 |
|----------|--------|
| 池化虚拟线程 | 每任务新建 |
| `synchronized` 内含阻塞 | `ReentrantLock` 或缩小同步块 |
| `ThreadLocal` 无节制 | `ScopedValue` 或局部变量 |
| 忽略 Executor 关闭 | try-with-resources |

### 6.4 Web 容器集成

Spring Boot 3.2+：

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

Tomcat/Jetty 自动为每个请求创建虚拟线程。

---
## 第七章：最佳实践

### 7.1 设计模式变更

| 传统模式 | 虚拟线程模式 |
|---------|-------------|
| 线程池 + 队列 | 每任务新建虚拟线程 |
| 异步回调 / CompletableFuture | 同步阻塞代码 |
| 反应式框架 | 传统阻塞库 |
| 精细控制并发数 | 随意创建百万线程 |

### 7.2 代码审查清单

- [ ] `synchronized` 是否包含阻塞操作？ → 替换为 `ReentrantLock`
- [ ] `ThreadLocal` 是否必要？ → 改用 `ScopedValue` 或局部变量
- [ ] 第三方库是否兼容虚拟线程？
- [ ] `ExecutorService` 是否用 try-with-resources 管理？
- [ ] 是否存在 `native` 方法调用？

### 7.3 适用场景决策树

```
是否以 I/O 操作为主（DB、HTTP、文件）？
├── 是 → 虚拟线程 ✅ 强烈推荐
│      └── 是否含大量 CPU 计算？
│          ├── 少量 → 适合
│          └── 大量 → 混合使用平台线程池
└── 否（纯 CPU） → 平台线程 ✅
```

### 7.4 迁移策略

1. **识别**：扫描所有 `synchronized` 和阻塞点。
2. **替换锁**：`synchronized` → `ReentrantLock`。
3. **更换执行器**：`Executors.newFixedThreadPool(...)` → `Executors.newVirtualThreadPerTaskExecutor()`。
4. **替换 ThreadLocal**：逐步改为 `ScopedValue`。

### 7.5 生产部署清单

- [ ] JDK 21+（推荐 JDK 25+ 以获取无固定 `synchronized`）
- [ ] 启用 `-Djdk.tracePinnedThreads=full` 测试
- [ ] 检查所有第三方库兼容性
- [ ] 无虚拟线程池化
- [ ] JFR 监控配置
- [ ] Pinning 报警阈值
- [ ] 回滚方案

---
## 附录：速查手册

### A. 常用 API

| 功能 | 代码 |
|------|------|
| 创建虚拟线程 | `Thread.startVirtualThread(runnable)` |
| 虚拟线程执行器 | `Executors.newVirtualThreadPerTaskExecutor()` |
| 结构化并发 | `new StructuredTaskScope.ShutdownOnFailure()` |
| Scoped Value | `ScopedValue.where(key, value).run(action)` |
| 检测 pinning | `-Djdk.tracePinnedThreads=full` |
| 载体线程数 | `-Djdk.virtualThreadScheduler.parallelism=N` |

### B. 故障排查决策表

| 症状 | 可能原因 | 排查步骤 | 解决方案 |
|------|---------|---------|---------|
| 延迟飙升 | Pinning 饥饿 | `tracePinnedThreads` + JFR | 替换 `synchronized` |
| CPU 100% | 忙等待 / 大量 pinning | JFR 查看 `VirtualThreadPinned` | 优化阻塞点 |
| 内存持续增长 | 海量虚拟线程同时存活 | 堆 dump | 控制并发数，使用 `ScopedValue` |
| 应用卡死 | Pinning 饱和死锁 | 检查载体线程状态 | 增加 parallelism 或升级 JDK |

### C. 关键 JEP 参考

| JEP | 特性 | 状态 |
|-----|------|------|
| JEP 425 | Virtual Threads (Preview) | JDK 19 |
| JEP 436 | Virtual Threads (Second Preview) | JDK 20 |
| JEP 444 | Virtual Threads | JDK 21 LTS |
| JEP 491 | Synchronize without Pinning | JDK 25 |
| JEP 453 | Structured Concurrency (Preview) | JDK 21 |
| JEP 480 | Structured Concurrency (Third Preview) | JDK 23 |

---