
---
> **阅读建议**：本文适合从零系统学习 JVM，也可作为生产调优和问题排查的参考手册。建议按顺序阅读，也可直接跳转到感兴趣的章节。
---
## 目录

**第一部分：理解 JVM —— 从零构建你的虚拟机认知**

1. [引言：从源代码到可运行程序](#1)
2. [JVM 整体架构](#2)
3. [类加载子系统](#3)
4. [运行时数据区详解](#4)
5. [对象的内存布局与生命周期](#5)
6. [垃圾回收：核心原理与算法](#6)
7. [垃圾收集器详解与对比](#7)
   - 7.1 Serial / Serial Old
   - 7.2 ParNew
   - 7.3 Parallel Scavenge / Parallel Old
   - 7.4 CMS
   - 7.5 G1
   - 7.6 ZGC 与 Shenandoah
   - 7.7 各收集器 YGC / Old GC / Full GC 详细过程汇总
   - 7.8 收集器核心差异对比表
   - 7.9 选型速查
8. [执行引擎：从解释到极致编译优化](#8)
9. [Java 内存模型与并发](#9)

**第二部分：实战 —— JVM 调优方法、工具与参数**

10. [监控与诊断工具箱](#10)
11. [内存容量规划与调优](#11)
12. [垃圾收集器选型与参数实战](#12)
13. [紧急故障排查手册](#13)
14. [JIT 与容器环境专项优化](#14)
15. [总结与最佳实践](#15)

**第三部分：生产环境常见坑与问题分类解决指南**

16. [JVM 生产环境常见坑与问题分类解决指南](#16)
    - 16.1 问题分类总览
    - 16.2 内存溢出（OOM）的三种主要形态及解法
    - 16.3 GC 停顿过长与频繁 GC
    - 16.4 CPU 飙高的 GC 原因及排查
    - 16.5 线程阻塞与死锁
    - 16.6 元空间/类加载泄漏（大数据框架常见）
    - 16.7 堆外内存泄漏（Netty、NIO 典型问题）
    - 16.8 容器环境（K8s/Docker）特有坑
    - 16.9 大数据高并发场景下的 JVM 调优 checklist
    - 附：高频问题速查表

---

## 第一部分：理解 JVM

<a name="1"></a>
### 1. 引言：从源代码到可运行程序

#### CPU 眼中的世界
CPU 只认识特定平台的机器码。C/C++ 直接编译成平台相关机器码，丧失了跨平台性。

#### 从第一性原理出发：为什么需要虚拟机？
Java 的目标是 “Write Once, Run Anywhere”。解决办法是 **加一层抽象**：
- 先将 Java 代码编译成一个虚构的理想 CPU 的指令集——**字节码（Bytecode）**。
- 然后为每个真实平台（Windows、Linux 等）分别实现一个程序，这个程序模拟该理想 CPU，读取字节码并一条条让真实 CPU 执行。这个程序就是 **Java 虚拟机（JVM）**。

这一抽象层解决了三个最根本的问题：
1. **跨平台**：只要平台有 JVM 实现，同一份字节码到处运行。
2. **自动内存管理**：理想 CPU 规定程序员只管申请对象，JVM 后台自动回收垃圾。
3. **安全沙箱**：字节码经过验证，防止恶意破坏宿主机。

接下来，我们钻进这台“虚拟的计算机”，看看内部到底是怎么搭建的。

<a name="2"></a>
### 2. JVM 整体架构

JVM 作为一个“模拟计算机”，遵循真实计算机的基本结构：
- **类加载器（Class Loader）**：把 `.class` 文件读入内存并整理格式。
- **运行时数据区（Runtime Data Area）**：JVM 的内存，分多个功能区。
- **执行引擎（Execution Engine）**：JVM 的“CPU”，解释或编译执行字节码。

**一次 Java 程序运行的全景时序：**
`java HelloWorld` → 启动 JVM 进程 → 类加载器加载 `HelloWorld.class` → 类元信息放入方法区 → JVM 找到 `main` 方法，在虚拟机栈中创建栈帧 → 执行引擎解释/编译执行字节码 → 若需对象，则在堆中分配 → 内存不足时触发 GC → 程序结束。

<a name="3"></a>
### 3. 类加载子系统

#### 3.1 加载、链接与初始化
- **加载**：通过类的全限定名找到字节码，读入并转换成方法区中的内部结构，同时在堆中生成 `Class` 对象。
- **链接**：分为三步：
  - **验证**：字节码安全检查（魔数、版本、指令合法性）。
  - **准备**：为静态变量分配内存并赋**默认零值**（真正的赋值在初始化阶段）。
  - **解析**：将符号引用转换为直接引用（地址偏移）。
- **初始化**：执行静态变量赋值和静态代码块，对应 `<clinit>()` 方法。

#### 3.2 双亲委派模型
多个类加载器形成层级关系：  
**Bootstrap ClassLoader**（加载 `lib` 核心类）→ **Platform/Extension ClassLoader**（加载 `lib/ext` 或平台类）→ **Application ClassLoader**（加载 ClassPath 下的类）。  
一个类加载器收到请求时，优先委派给父加载器，父加载器无法完成时才自己加载。这样保证了核心类的安全与唯一性。

**为什么要双亲委派？**  
若无此机制，同一个类可能被多个加载器加载，导致类型混乱（例如用户自定义 `java.lang.Object` 会破坏基础类型体系）。

**破坏双亲委派的常见场景**：
- **SPI 机制**（如 JDBC）：通过线程上下文类加载器让父加载器调用子加载器的实现类。
- **Tomcat**：每个 Web 应用使用独立的 `WebappClassLoader`，优先加载自身 `WEB-INF/classes` 和 `lib`，打破双亲委派，实现应用隔离和 JSP 热部署。
- **OSGi**：采用网状类加载器，实现模块化热替换。

<a name="4"></a>
### 4. 运行时数据区详解

| 区域 | 线程私有？ | 存放内容 |
|------|------------|----------|
| 程序计数器 | 是 | 当前线程执行的字节码行号 |
| Java 虚拟机栈 | 是 | 栈帧（局部变量表、操作数栈、动态链接、返回地址） |
| 本地方法栈 | 是 | 服务于 `native` 方法 |
| 堆（Heap） | 否 | 对象实例和数组，GC 主战场 |
| 方法区 / 元空间 | 否 | 类的结构信息、常量池、静态变量、编译后代码 |

- **Java 虚拟机栈**：每个方法一个栈帧。局部变量表存参数和局部变量；操作数栈是基于栈的指令集的工作台；动态链接解析符号引用；返回地址指出口。**GC 不会回收栈帧**，栈帧随着方法结束自动弹出。
- **方法区演进**：JDK 7 及以前使用**永久代（PermGen）**；JDK 8 开始使用**元空间（Metaspace）**，直接使用本地内存，不再受堆大小限制。

<a name="5"></a>
### 5. 对象的内存布局与生命周期

#### 5.1 对象创建过程
`new` 指令 → 类加载检查 → 分配内存（指针碰撞或空闲列表，并发控制用 CAS 或 TLAB）→ 初始化零值 → 设置对象头 → 执行 `<init>()` 构造函数。

#### 5.2 对象内存布局
- **对象头**：
  - **Mark Word**：存储哈希码、GC 年龄、锁状态等（64 位虚拟机下 8 字节）。
  - **Klass Pointer**：指向类元数据。在 64 位 JVM 中，默认开启指针压缩（`-XX:+UseCompressedOops`），Klass Pointer 为 4 字节；若关闭压缩则为 8 字节。
  - **因此，默认情况下普通对象头为 12 字节**（8 + 4），而非固定 16 字节。
- **实例数据**：父类与自身字段。
- **对齐填充**：保证对象起始地址是 8 字节的倍数。

#### 5.3 对象访问定位
- **句柄访问**：引用指向句柄池，句柄再指向实例数据和类型数据，对象移动时只需改句柄。
- **直接指针（HotSpot 默认）**：引用直接指向对象实例，访问速度快。

<a name="6"></a>
### 6. 垃圾回收：核心原理与算法

#### 6.1 第一性需求：为什么不能手动管理内存？
程序员无法可靠地处理复杂引用关系，导致内存泄漏或悬挂指针。JVM 接管内存回收，让开发者专注业务。

#### 6.2 判断对象生死
- **引用计数法**：简单但无法解决循环引用。
- **可达性分析**：从 **GC Roots**（栈中引用、静态变量、JNI 引用等）出发，顺着引用链可达的对象为存活，其余为垃圾。

#### 6.3 分代收集假说
大多数对象朝生夕死，多次存活的对象更难回收。因此堆分为**新生代**（Eden + Survivor*2）和**老年代**，不同区域用不同算法。

#### 6.4 基础算法
- **标记-清除**：标记存活，线性清除。会形成内存碎片。
- **标记-复制**：两块内存，回收时将存活对象复制到另一块，无碎片但浪费空间。适合新生代（对象死亡率高，浪费比例低）。
- **标记-整理**：存活对象向一端移动，清理边界外内存。适合老年代，无碎片但移动成本高。

#### 6.5 并发回收的挑战
并发标记时，用户线程可能改变引用关系，导致存活对象被漏标。解决方案：
- **增量更新**（CMS 使用）：黑色对象新增对白色对象引用时，将黑色变回灰色重新扫描。
- **SATB（原始快照）**（G1、Shenandoah 使用）：标记开始时拍快照，引用被删除时将被删引用对象标记为活对象，宁可多留不漏删。
两种方案均依赖**写屏障**来截获引用变动。

<a name="7"></a>
### 7. 垃圾收集器详解与对比

垃圾收集器没有“最好”，只有“最适合”。下面逐个剖析设计权衡，并**详细展开每个收集器的 YGC、Old GC、Mixed GC（G1）、Full GC 的过程**。

---

#### 7.1 Serial / Serial Old

- **设计**：单线程，新生代标记-复制，老年代标记-整理，全程 STW。
- **优点**：单核下没有线程交互开销，简单高效。
- **缺点**：多核下停顿过长。
- **适用**：客户端应用、微服务小内存（<100MB）。

##### YGC（Serial，单线程复制）
- **触发**：Eden 区满。
- **过程**：STW → 复制存活对象到 Survivor To → 清空 Eden/From → 交换 From/To → 恢复应用。
- **年龄处理**：每次 YGC 存活对象年龄 +1，达到 `-XX:MaxTenuringThreshold`（默认 15）则晋升到老年代；另根据**动态年龄判断**（Survivor 同年龄对象总和超 50%，则大于等于该年龄的晋升）。

##### Old GC（Serial Old，单线程标记-整理）
- **触发**：老年代空间不足，或 YGC 后晋升对象放不进老年代。
- **过程**：STW → 标记存活对象 → **整理**（滑动到一端）→ 清理边界外内存。

##### Full GC
- **触发**：晋升失败、老年代空间不足、`System.gc()`。
- **过程**：同 Old GC，但范围覆盖新生代 + 老年代 + 元空间。

---

#### 7.2 ParNew

- **设计**：Serial 的多线程新生代版本，标记-复制并行，仍 STW。
- **用途**：历史上与 CMS 搭配，现已被 Parallel Scavenge 与 G1 取代。

##### YGC（ParNew，并行多线程复制）
- **触发**：Eden 区满。
- **过程**：STW，多线程并行执行复制（逻辑同 Serial YGC）。

---

#### 7.3 Parallel Scavenge / Parallel Old

- **设计**：新生代与老年代均为并行 STW 收集，追求**可控的高吞吐量**。提供 `MaxGCPauseMillis` 和 `GCTimeRatio` 精确调节。
- **优点**：吞吐量最高，适合计算密集型任务。
- **缺点**：停顿时间可能较长，不适合低延迟应用。
- **注意**：**这是 JDK 8 的默认垃圾收集器组合**（`-XX:+UseParallelGC`）。

##### YGC（Parallel Scavenge，并行多线程复制）
- **触发**：Eden 区满。
- **核心特点**：支持**自适应调节**（`-XX:+UseAdaptiveSizePolicy`），JVM 动态调整 Eden、Survivor 大小以匹配停顿/吞吐目标。
- **跨代引用**：通过卡表（Card Table）和写屏障维护老年代→新生代引用，YGC 时只扫描脏卡（Dirty Card）对应的老年代区域，而非全老年代。

##### Old GC（Parallel Old，并行多线程标记-整理）
- **触发**：老年代空间不足，或 YGC 后晋升失败。
- **过程**：STW，多线程并行执行标记 → 整理（滑动）→ 清理。

##### Full GC
- **触发**：老年代 GC 后仍不足，或元空间不足。
- **过程**：同 Parallel Old，但覆盖全堆（含元空间）。

---

#### 7.4 CMS (Concurrent Mark Sweep)

- **设计**：老年代收集，以最短停顿为目标，用 **标记-清除** 实现大部分阶段并发。
- **流程**（6 个阶段）：
  1. **初始标记**（STW）：标记 GC Roots 直接可达的老年代对象，以及新生代直接引用的老年代对象。时间很短。JDK 8 后可通过 `-XX:+CMSParallelInitialMarkEnabled` 开启并行执行。
  2. **并发标记**（并发）：GC 线程与应用线程同时运行，从已标记对象开始遍历老年代对象图。期间引用可能变化，发生变化的对象所在 Card 被标记为 Dirty。
  3. **并发预清理**（并发）：扫描 Dirty Card 中对象，将其标记为存活，并清除 Dirty 标识。为重新标记做准备。
  4. **可被终止的预清理**（并发）：循环处理 From/To 区引用的老年代对象及 Dirty Card，满足阈值时终止。可用 `-XX:+CMSScavengeBeforeRemark` 在此阶段之前强制执行一次 YGC，减少后续重新标记时的新生代扫描量。
  5. **重新标记**（STW）：扫描“新生代 + GC Roots + Dirty Card 对应的老年代对象”，完成最终标记。通常比初始标记更长，可多线程并行（`-XX:+CMSParallelRemarkEnabled`）。
  6. **并发清除**（并发）：清除未被标记的对象，产生内存碎片。
- **致命缺点**：内存碎片导致并发失败或晋升失败，退化为 Serial Old 造成长停顿；浮动垃圾需要预留空间。
- **现状**：**JDK 9 废弃，JDK 14 移除**。我们学习它是为了理解并发回收的经典模式。

##### Full GC（CMS 失败时的降级）
- **触发**：
  - **Concurrent Mode Failure（并发模式失败）**：CMS 并发回收期间，老年代空间不足（晋升对象或大对象分配无法容纳）→ 退化为 **Serial Old** 进行单线程 STW 的 Full GC。
  - **Promotion Failed（晋升失败）**：YGC 时 Survivor 放不下，老年代也放不下 → 触发 Full GC。
- **碎片整理**：CMS 基于标记-清除，产生碎片。可通过 `-XX:CMSFullGCsBeforeCompaction` 设置执行多少次 CMS GC 后进行一次 STW 的碎片整理（默认 0，即每次 Full GC 都整理），整理时同样退化为 Serial Old。

---

#### 7.5 G1 (Garbage First)

- **设计**：化整为零，将堆划分为大小相等的 Region，动态扮演 Eden/Survivor/Old 角色。追求 **可预测的停顿**。
- **核心流程**：Young GC (STW 复制) → 并发标记周期 (SATB) → Mixed GC (回收部分高价值老年代 Region)。
- **关键技术**：Remembered Set (RSet) 解决跨 Region 引用，写屏障维护。
- **优点**：无内存碎片，停顿可预测，内存自动伸缩。
- **代价**：RSet 通常占堆内存 5% 以上，CPU 写屏障开销。
- **适用**：4GB～32GB 堆的服务器应用，**JDK 9 及之后默认收集器**。

##### YGC（Young GC）
- **触发**：G1 动态调整 Eden Region 数量，预测 YGC 时间接近 `MaxGCPauseMillis` 时触发。
- **过程**（STW，多线程并行复制）：
  1. **根扫描**：扫描 GC Roots（栈帧引用、JNI 引用等）。
  2. **更新 RSet**：处理跨 Region 引用信息。
  3. **对象复制**：将 Eden + Survivor 存活对象复制到新的 Region（存活对象年龄 +1），达到阈值则晋升到 Old Region。
  4. **回收原 Region**：清空，变为空闲状态。

##### Mixed GC（G1 特有）
- **触发**：老年代占用率达到 **`-XX:InitiatingHeapOccupancyPercent`**（默认 **45%**）时，触发**并发标记周期**，之后执行多次 Mixed GC。
- **完整流程**：
  1. **初始标记**（STW）：标记 GC Roots 直接可达的对象（附带在 YGC 中完成，无额外停顿）。
  2. **并发标记**（并发）：从初始标记对象出发遍历对象图，使用 **SATB** 记录并发期间引用变化。
  3. **最终标记**（STW）：处理 SATB 缓冲区的引用变化，完成标记。
  4. **筛选回收 / Mixed GC**（STW，多线程并行复制）：计算每个 Region 的回收价值（垃圾占比），按**收益最高、耗时最短**原则排序，将选中的 Old Region 与所有年轻代 Region 组成 **CSet（Collection Set）** 回收。**关键约束**：Selected Old Region 活跃度需低于 `-XX:G1MixedGCLiveThresholdPercent`（默认 85%）；混合回收可能分多次执行（默认 `-XX:G1MixedGCCountTarget=8`），每次只回收部分 Region，将单次 STW 控制在 `MaxGCPauseMillis` 内。

##### Full GC（G1 降级，严重异常）
- **触发**：Mixed GC 复制时 **To-space exhausted**（无足够空闲 Region），或并发标记因对象分配过快而失败（Concurrent Mark Abort），或 RSet 更新溢出。
- **过程**：放弃 G1 的并发/并行机制，退化为**单线程 Serial Old** 收集器进行 **STW 标记-整理**，回收整个堆。停顿时间数秒级，是系统健康度严重告警信号。

---

#### 7.6 ZGC 与 Shenandoah

两者目标一致：**全并发**，停顿低至亚毫秒级，且不随堆大小或对象数量增加。

##### ZGC
- **原理**：基于**染色指针（Colored Pointer）**和读屏障（Load Barrier）。利用 64 位指针的高 4 位存储对象状态（M0/M1/Remapped/Finalizable），避免额外内存开销。读屏障在读取对象引用时触发，若对象已被转移则自动重定向，实现并发转移过程中应用线程可继续运行。
- **关键特性**：
  - 分代：JDK 21 引入**分代 ZGC**（JEP 439），需通过 `-XX:+ZGenerational` 显式启用，JDK 24 才默认分代。
  - 内存开销极小。
- **适用**：大堆（几十 GB 到 TB 级），对延迟极为敏感的核心系统。

##### Shenandoah
- **原理**：基于 **Brooks Pointer（转发指针）** 和读屏障。对象头前增加一个转发指针，访问时读屏障检查并修正，实现并发复制。
- **与 ZGC 的区别**：
  - Shenandoah 使用额外转发指针，每个对象多一个指针的内存开销；ZGC 使用染色指针，内存开销更小。
  - ZGC 仅支持 x86_64、AArch64；Shenandoah 支持全平台。
  - OracleJDK 不包含 Shenandoah（仅 OpenJDK 有）。
- **适用**：同 ZGC。

##### ZGC / Shenandoah 的回收过程（以 ZGC 为例，分代模式）
- **YGC**：回收年轻代 Region，采用**并发整理**，**无 STW 停顿**。
- **并发回收周期**（4 阶段，仅 2 次极短 STW）：
  1. **初始标记**（STW <1ms）：标记 GC Roots 直接可达的对象。
  2. **并发标记**（并发）：遍历对象图，标记存活对象。
  3. **最终标记**（STW <1ms）：处理并发标记阶段的引用变化。
  4. **并发转移**（并发）：将存活对象复制到新 Region，读屏障自动修正引用。
  5. **并发重映射**（并发）：修正剩余引用（可与下次标记合并）。
- **Full GC**：仅在并发失败（如标记阶段对象分配过快）时触发，退化为 STW 单线程整理，极为罕见。

Shenandoah 过程类似，只是使用 Brooks Pointer 实现。

---

#### 7.7 各收集器 YGC / Old GC / Full GC 详细过程汇总表

| 收集器 | YGC 名称 | YGC 算法 | YGC 是否并发 | Old/Mixed GC 名称 | Old GC 算法 | Old GC 是否并发 | Full GC 降级路径 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Serial** | YGC | 标记-复制 | 否 | Old GC | 标记-整理 | 否 | — |
| **Parallel** | YGC | 标记-复制 | 否 | Old GC | 标记-整理 | 否 | — |
| **CMS+ParNew** | YGC | 标记-复制 | 否 | Old GC (CMS) | 标记-清除 | 大部分并发 | **Serial Old** |
| **G1** | YGC | 标记-复制 | 否 | **Mixed GC** | 标记-复制 | 否（但只 STW 部分 Region） | **Serial Old** |
| **ZGC** | 分代 YGC（JDK21+） | 并发整理 | **并发** | 老年代 GC | 并发整理 | **全并发** | 退化为 STW 整理 |
| **Shenandoah** | 分代 YGC（JDK21+） | 并发整理 | **并发** | 老年代 GC | 并发整理 | **全并发** | 退化为 STW 整理 |

---

#### 7.8 收集器核心差异对比表

| 收集器组合 | 目标 | 老年代算法 | 并发/并行 | 停顿时间 | 吞吐量 | 内存开销 | 核心弱点 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Serial / Serial Old** | 简单高效 | 标记-整理 | 全串行 STW | 长 | 中等 | 极小 | 多核停顿过长 |
| **Parallel / Parallel Old** | 极致吞吐量 | 并行标记-整理 | 全并行 STW | 长（可控） | **最高** | 较小 | 大堆停顿依然长 |
| **CMS + ParNew** | 低停顿 | 标记-清除 | 部分并发 | 较短 | 较低 | 较小 | 内存碎片，并发失败（已废弃） |
| **G1** | 可预测停顿 | 分区标记-复制 | 大部分并发 | **中低** | 中高 | **较大** (RSet) | 参数调优较复杂 |
| **ZGC / Shenandoah** | 亚毫秒级停顿 | 全并发整理 | **全并发** | **极低** (<1ms) | 中低 | ZGC小，Shen中等 | 吞吐量一般，需大堆 |

---

#### 7.9 选型速查（基于 JDK 版本）
- **JDK 8**：默认 Parallel。若需要低延迟且无法升级 JDK，可尝试 G1（`-XX:+UseG1GC`），但建议直接升级。
- **JDK 11+**：默认 G1，绝大多数场景适用。
- **JDK 17+**：ZGC 已成熟，大堆低延迟首选。
- **JDK 21+**：可使用分代 ZGC（`-XX:+UseZGC -XX:+ZGenerational`）获得更佳性能。

<a name="8"></a>
### 8. 执行引擎：从解释到极致编译优化

#### 8.1 解释器与 JIT 编译器共存
- **解释执行**：启动快但运行慢。
- **即时编译 (JIT)**：将热点代码编译为本地机器码，运行时快，但有编译开销。
- HotSpot 默认混合模式，先解释后编译。

#### 8.2 热点探测
基于方法调用计数器和回边计数器。达到阈值触发 JIT 编译。  
> 注：当分层编译（默认开启）时，`-XX:CompileThreshold` 参数失效，JVM 动态调整阈值。

#### 8.3 编译器演进
- **C1（Client）**：编译快，优化浅，适合启动快场景。
- **C2（Server）**：编译慢但深度优化，运行时极致性能。
- **分层编译**：结合两者，先用 C1 编译并收集统计信息，再交由 C2 深度优化。

#### 8.4 核心编译优化技术
- **方法内联**：消除方法调用开销，是其它优化的基石。
- **逃逸分析**：对象未逃逸出方法或线程时，进行栈上分配、标量替换、锁消除。
- **去虚拟化**：虚方法调用如只有一种实现则转为直接调用并内联。

<a name="9"></a>
### 9. Java 内存模型与并发

#### 9.1 为什么需要内存模型？
CPU 和编译器会指令重排序，多级缓存导致数据不一致。JMM 定义了多线程环境下的可见性、有序性和原子性规则。

#### 9.2 主内存与工作内存抽象
- **主内存**：所有线程共享的变量存储区。
- **工作内存**：线程私有缓存，保存主内存变量的副本。

#### 9.3 happens-before 规则
- 程序次序、锁规则、volatile 规则、传递性、start/join 等。
- `volatile` 保证可见性和有序性（禁止重排序），不保证原子性。
- `synchronized` 保证原子性、可见性、有序性。
- `final` 能防止对象逸出导致的未初始化可见问题（前提是对象没有 this 引用逸出）。

---

## 第二部分：实战 —— JVM 调优方法、工具与参数

<a name="10"></a>
### 10. 监控与诊断工具箱

#### 10.1 命令行工具
- `jps`：列出 Java 进程。
- `jstat -gcutil <pid> 1000`：查看各区使用百分比与 GC 次数。  
  **关键字段**：  
  `S0U`/`S1U` – Survivor 区使用量；`EU` – Eden 使用量；`OU` – 老年代使用量；`YGC`/`YGCT` – Young GC 次数和耗时；`FGC`/`FGCT` – Full GC 次数和耗时。
- `jmap -dump:live,format=b,file=heap.hprof <pid>`：堆转储快照。  
  > **注意**：`jmap -heap <pid>` 在某些 JDK 8+ 版本/平台上受限或已移除，生产环境推荐使用 `jcmd <pid> GC.heap_info` 或 `jhsdb jmap --heap --pid <pid>`。
- `jstack <pid>`：线程快照，检测死锁与长时间阻塞。
- `jcmd <pid> help`：多功能诊断命令。

#### 10.2 GC 日志（JDK 9+ 统一格式）
```
-Xlog:gc*:file=gc.log:time,level,tags
```
关注：停顿时间、回收量、各阶段耗时。可使用 GCViewer 或 GCEasy 可视化。

<a name="11"></a>
### 11. 内存容量规划与调优

#### 11.1 堆大小
- `-Xms` 和 `-Xmx` 设置相同，避免动态伸缩。
- 堆大小 ≤ 物理内存 50~80%（容器内须结合容器支持）。
- 容器环境推荐：`-XX:MaxRAMPercentage=75.0`，同时确保开启 `-XX:+UseContainerSupport`（**JDK 8u191 之前需显式添加，JDK 10+ 默认开启**）。

#### 11.2 新生代与老年代比例
- `-XX:NewRatio`（默认 2，新生代 1/3）。
- 响应式系统可增大新生代（如 NewRatio=1），减少对象过早晋升。
- Survivor 区可调 `-XX:SurvivorRatio` 和目标使用率 `TargetSurvivorRatio`。

#### 11.3 元空间与直接内存
- **元空间默认大小**：`-XX:MetaspaceSize` 默认约 **21MB**（触发 GC 的阈值），**不是初始容量**。`-XX:MaxMetaspaceSize` 默认**无上限**（受本地内存限制），建议显式设置 `-XX:MaxMetaspaceSize=256m` 防止动态类加载无限膨胀。
- 直接内存限制：`-XX:MaxDirectMemorySize`，防止 NIO 使用过多堆外内存导致 OOM。

<a name="12"></a>
### 12. 垃圾收集器选型与参数实战

#### 12.1 通用调优步骤
1. 明确目标（延迟/吞吐量/内存）。
2. 建立基线（采集 GC 日志、CPU、内存数据）。
3. 分析瓶颈（GC 停顿过长？吞吐量不足？）。
4. 一次只改一个参数，压测验证。

#### 12.2 各收集器核心参数

**G1**
```
-XX:+UseG1GC
-XX:MaxGCPauseMillis=100        # 停顿目标，默认 200ms
-XX:G1HeapRegionSize=8m         # Region 大小，值为 2 的幂
-XX:InitiatingHeapOccupancyPercent=35  # 触发并发标记的堆占用阈值，默认 45
-XX:G1NewSizePercent=5          # 新生代最小占比
-XX:G1MaxNewSizePercent=60      # 新生代最大占比
```

**Parallel**
```
-XX:+UseParallelGC
-XX:MaxGCPauseMillis=<N>
-XX:GCTimeRatio=<N>
-XX:+UseAdaptiveSizePolicy      # 自动调整新生代比例
```

**ZGC**
```
-XX:+UseZGC
-XX:ConcGCThreads=4             # 并发线程数，通常为核数 1/4
-XX:+ZGenerational              # JDK 21+ 启用分代模式（JDK 24 前需显式）
```

#### 12.3 参数模板参考

**通用 Web 服务（G1，堆 4G，JDK 11+）**
```
-Xms4g -Xmx4g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=100
-XX:+ParallelRefProcEnabled
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/logs/oom.hprof
-Xlog:gc*:file=/logs/gc.log:time,level,tags
-XX:+UseContainerSupport
-XX:MaxRAMPercentage=75.0
```

**高吞吐批处理（Parallel GC，JDK 8+）**
```
-Xms8g -Xmx8g
-XX:+UseParallelGC
-XX:ParallelGCThreads=4
-XX:+HeapDumpOnOutOfMemoryError
```

**低延迟大堆（ZGC，32G，JDK 17+）**
```
-Xms32g -Xmx32g
-XX:+UseZGC
-XX:ConcGCThreads=8
-XX:+HeapDumpOnOutOfMemoryError
-Xlog:gc*:file=/logs/gc.log:time,level,tags
```

<a name="13"></a>
### 13. 紧急故障排查手册

#### 13.1 内存溢出 (OOM)
- 添加参数 `-XX:+HeapDumpOnOutOfMemoryError`，指定 `-XX:HeapDumpPath=/path`。
- 使用 MAT 或 JProfiler 分析堆转储，查看 Dominator Tree 找最大对象及其 GC Roots。
- **常见原因**：集合无限增长、ThreadLocal 未清理、查询未分页、监听器未注销、动态生成类过多（元空间溢出）。
- **临时解决**：增大堆/元空间并重启；根治需修改代码。

#### 13.2 CPU 飙高
- `top -Hp <pid>` 找到高 CPU 线程 ID（十进制）。
- 转换为十六进制，用 `printf "%x" <tid>`。
- `jstack <pid> | grep -A 20 <十六进制tid>` 查看该线程堆栈。
- **常见原因**：死循环、频繁 Full GC、正则表达式灾难性回溯、无 sleep 的密集循环。

#### 13.3 死锁
- `jstack -l <pid>` 会自动检测并输出 `Found one Java-level deadlock`。
- 修复方法：按顺序获取锁，或使用 `tryLock` 定时等待。

<a name="14"></a>
### 14. JIT 与容器环境专项优化

#### 14.1 JIT 优化
- 冷启动慢可降低编译阈值（仅当禁用分层编译时有效，不推荐）。
- 观察编译日志：`-XX:+PrintCompilation`。
- CodeCache 满了会导致 JIT 停止，设置 `-XX:ReservedCodeCacheSize=256m`（默认 240MB）。

#### 14.2 容器环境（Docker/K8s）
- **JDK 8u191 及更早**：必须添加 `-XX:+UseCGroupMemoryLimitForHeap -XX:+UseContainerSupport` 才能感知容器内存限制。
- **JDK 10+**：`UseContainerSupport` 默认开启，推荐使用 `-XX:MaxRAMPercentage=75.0` 动态调整堆大小，避免写死 `-Xmx`，以便容器规格变更时自动适应。
- 注意：CPU 限制同样会被 JVM 感知（`ActiveProcessorCount` 默认为容器 CPU 数）。

<a name="15"></a>
### 15. 总结与最佳实践

现在，让我们用三句话概括 JVM 的核心知识体系：

1. **JVM 是一台模拟计算机**：类加载、内存管理、执行引擎三大部件配合，实现对字节码的跨平台执行。
2. **自动内存管理是最大财富**：通过可达性分析、分代收集和各种垃圾收集器的权衡，让你免于手动内存管理的噩梦。
3. **并发正确性靠 JMM 保证**：理解 happens-before 规则和 volatile、synchronized、final 的语义，才能编写正确的并发代码。

**开发中的最佳实践提醒：**
- 尽量用 try-with-resources 释放系统资源。
- 缩小变量作用域，让对象尽快不可达，利于 GC。
- 谨慎使用缓存，为长生命周期集合设定容量上限，并考虑弱/软引用。
- 高并发状态变量优先用 `volatile` 或原子类，避免 synchronized 滥用。
- 性能优化前务必采集 GC 日志和线程 dump，不要凭感觉调参。

**调优者的工作流：** 观察指标 → 定位瓶颈 → 形成假设 → 修改单一参数 → 压测验证 → 回滚或保留。JVM 没有银弹，但你手中的这些原理、工具和收集器对比，就是你做出正确决策的全部武器。

---

## 第三部分：生产环境常见坑与问题分类解决指南

<a name="16"></a>
### 16. JVM 生产环境常见坑与问题分类解决指南

在大数据、高并发场景下，JVM 面临的内存分配速度、对象存活率、GC 频率、元空间压力等挑战远高于普通应用。本章将常见问题按**分类**整理，并给出可直接落地的排查思路和解决方案。

---

#### 16.1 问题分类总览

| 分类 | 典型现象 | 高频场景 |
| :--- | :--- | :--- |
| **内存溢出类** | OOM: Java heap space, Metaspace, Direct buffer, unable to create native thread | 大对象未释放、类加载泄漏、堆外内存泄漏、线程创建过多 |
| **GC 停顿过长** | 接口超时、TP99 飙升、监控看到 GC pause > 1s | Full GC 频繁、Mixed GC 拷贝量大、CMS 并发模式失败 |
| **CPU 飙高** | 容器 CPU throttle，top -Hp 看到 GC 线程或 compiler 线程占用高 | 频繁 GC、JIT 编译过热、死循环、正则回溯 |
| **线程阻塞/死锁** | 应用无响应、jstack 看到大量 BLOCKED / WAITING | 连接池满、同步块竞争、分布式锁超时 |
| **元空间/类加载泄漏** | Metaspace 持续增长，Full GC 无法回收 | Groovy 动态编译、热部署、大量代理类生成 |
| **堆外内存泄漏** | 进程 RSS 远大于 Xmx，直接内存 OOM | NIO ByteBuffer 未释放、Netty 使用不当、JNI 泄漏 |
| **容器环境特有** | 容器内 JVM 识别错误内存/CPU，频繁 OOM Kill | JDK 版本过低、未设置 UseContainerSupport |
| **高并发分配压力** | Young GC 过于频繁（每秒多次），晋升年龄不足 | TLAB 过小、Eden 区过小、大对象过多 |

---

#### 16.2 内存溢出（OOM）的三种主要形态及解法

##### 16.2.1 Java Heap Space OOM
- **典型堆栈**：`java.lang.OutOfMemoryError: Java heap space`
- **常见原因**：
  - 缓存数据无限增长（如 `HashMap` 作为本地缓存未设上限）
  - 批量查询数据量过大（如不分页查询数据库）
  - 线程局部变量（ThreadLocal）未清理，导致对象长期持有
  - 监听器/回调未注销，集合对象持续引用
  - 内存泄漏：对象被 GC Roots 意外引用（如静态集合、类加载器）

- **解决方案**：
  1. **分析堆转储**：`jmap -dump:live,format=b,file=heap.hprof <pid>`，用 MAT 找出 Dominator Tree 中最大的对象及其 GC Roots 路径。
  2. **代码修复**：
     - 为缓存添加容量上限（如 Guava Cache、Caffeine 的 maximumSize）
     - 分页查询数据库（`LIMIT offset, size`）
     - 使用 `try-with-resources` 或显式 `remove()` 清理 ThreadLocal
     - 使用弱引用/软引用缓存（`WeakHashMap`、`SoftReference`）
  3. **临时止血**：增大 `-Xmx` 并重启，但需尽快定位泄漏点。

##### 16.2.2 Metaspace OOM
- **典型堆栈**：`java.lang.OutOfMemoryError: Metaspace`
- **常见原因**：
  - 动态生成类（如 Groovy、JSP、CGLIB 代理、MyBatis 的 `MapperProxy`）数量过多
  - 热部署应用时，旧的类加载器未被回收（如 Tomcat 的 `WebappClassLoader` 泄漏）
  - `-XX:MaxMetaspaceSize` 设置过小

- **解决方案**：
  1. **监控类加载数量**：`jstat -gc <pid>` 查看 `MC` 和 `MU`；`jcmd <pid> GC.class_stats` 输出类统计。
  2. **代码修复**：
     - 避免在循环中动态生成类（如 `GroovyClassLoader.parseClass`）
     - 确保自定义类加载器在不用时被置为 `null` 并触发 GC
     - Tomcat 中配置 `renewSessionOnAuthentication="false"` 等防止 Listener 泄漏
  3. **参数调整**：设置合理的 `-XX:MaxMetaspaceSize=256m`，并设置 `-XX:MinMetaspaceFreeRatio=40` 让 JVM 更积极回收。

##### 16.2.3 Direct Buffer OOM（堆外内存）
- **典型堆栈**：`java.lang.OutOfMemoryError: Direct buffer memory` 或进程被 OOM Killer 杀死
- **常见原因**：
  - NIO `ByteBuffer.allocateDirect()` 未及时释放（`DirectByteBuffer` 由 `Cleaner` 回收，但 GC 不可控）
  - Netty 中 `PooledByteBufAllocator` 未正确配置或内存泄漏
  - `-XX:MaxDirectMemorySize` 限制过小

- **解决方案**：
  1. **监控堆外内存**：`jcmd <pid> VM.native_memory summary`（需启用 `-XX:NativeMemoryTracking=summary`），或使用 `pmap -x <pid>` 观察 RSS。
  2. **代码修复**：
     - 显式调用 `((DirectBuffer) buffer).cleaner().clean()`（不推荐，依赖 unsafe）
     - 使用 Netty 的 `Unpooled.unreleasableBuffer` 并确保 `ReferenceCountUtil.release`
     - 避免频繁分配大块直接内存，改用内存池
  3. **参数调整**：适当增大 `-XX:MaxDirectMemorySize`，但需结合物理内存总量。同时考虑启用 `-XX:+DisableExplicitGC` 会阻止 `System.gc()` 对 DirectByteBuffer 的回收，需谨慎。

##### 16.2.4 Unable to create new native thread
- **典型堆栈**：`java.lang.OutOfMemoryError: unable to create new native thread`
- **常见原因**：
  - 线程数超过操作系统限制（`ulimit -u`）
  - 每个线程栈占用内存（`-Xss`）过大，导致地址空间不足（32 位 JVM 常见）
  - 线程创建过快，未设置最大线程池大小

- **解决方案**：
  1. **检查线程数**：`jstack <pid> | grep "^\"" | wc -l`，结合 `top -Hp <pid>`。
  2. **代码修复**：
     - 使用有界线程池（`ThreadPoolExecutor` 必须设置 `maximumPoolSize`）
     - 避免使用 `Executors.newCachedThreadPool()`（无界）
     - 异步任务使用 `ForkJoinPool` 或 `CompletableFuture` 共用线程池
  3. **系统调整**：增大 `ulimit -u`，降低 `-Xss`（如从 1M 降到 512k），确保 64 位 JVM。

---

#### 16.3 GC 停顿过长与频繁 GC

##### 16.3.1 高并发下 Young GC 过于频繁
- **现象**：`jstat -gcutil` 显示 YGC 每秒几十甚至上百次，YGCT 增长很快。
- **根本原因**：Eden 区过小，导致对象快速填满 Eden，频繁触发 YGC。
- **大数据/高并发场景特点**：短时间内创建大量临时对象，若 Eden 不足以容纳一次请求周期内的临时对象，这些对象会被提升到 Survivor 甚至老年代，导致 YGC 频繁且晋升压力大。

- **解决方案**：
  1. **增大新生代**：将 `-Xmn` 或 `-XX:NewRatio` 调小（如 `-XX:NewRatio=1` 让新生代占堆一半）。对于 G1，可调高 `-XX:G1NewSizePercent` 和 `-XX:G1MaxNewSizePercent`。
  2. **调整 TLAB（线程本地分配缓冲区）**：增大 `-XX:TLABSize`（默认 Eden 的 1%）或使用 `-XX:+ResizeTLAB` 自适应。TLAB 过小会导致频繁 TLAB 重填和锁竞争。
  3. **分析对象分配**：使用 `-XX:+PrintTLAB` 或 async-profiler 的 `alloc` 事件，找出哪个组件分配了大量临时对象，进行对象复用（如池化、StringBuilder 重用）。

##### 16.3.2 高并发下 Old GC / Full GC 频繁
- **现象**：FGC 每分钟多次，单次停顿超过 1 秒，导致服务雪崩。
- **常见原因**：
  - **晋升失败**：Survivor 区过小，YGC 后存活对象直接进老年代，老年代快速填满。
  - **大对象直接进入老年代**：`-XX:PretenureSizeThreshold` 设置过小，导致短生命周期大对象（如 byte[] 序列化结果）进入老年代。
  - **内存泄漏**：老年代持续增长无法回收。
  - **元空间填满**（对于 Full GC 包含元空间的收集器）。

- **解决方案**：
  1. **增大 Survivor 区**：`-XX:SurvivorRatio=6` 或 4（降低 Eden 占比），让更多存活对象留在新生代。
  2. **调整晋升阈值**：`-XX:MaxTenuringThreshold=15`（默认 15），配合动态年龄判断（Survivor 区同年龄对象总和 > 50% 则晋升）。可适当提高目标使用率 `-XX:TargetSurvivorRatio=90`。
  3. **大对象阈值调大**：`-XX:PretenureSizeThreshold=2m`（需注意单位是字节，且仅对 Serial/ParNew 有效；G1 使用 Region 大小的一半作为大对象阈值）。
  4. **切换到 G1 或 ZGC**：对于大于 4GB 的堆，G1 的 Mixed GC 可控性更好；对于 > 16GB 且延迟敏感，使用 ZGC。

##### 16.3.3 CMS 并发模式失败（Concurrent Mode Failure）
- **现象**：CMS GC 日志中出现 `concurrent mode failure`，随后退化为 Serial Old，停顿长达数秒。
- **原因**：CMS 并发回收期间，老年代增速太快，剩余空间不足以容纳晋升对象或大对象分配。
- **大数据/高并发场景**：短时流量尖刺导致对象分配速度远超 CMS 回收速度。

- **解决方案**（仅限 JDK 8 且无法升级的遗留系统）：
  1. **提前触发 CMS**：降低 `-XX:CMSInitiatingOccupancyFraction` 到 70 或更低，同时设置 `-XX:+UseCMSInitiatingOccupancyOnly`。
  2. **增大老年代**：增加堆大小，或降低 `-XX:NewRatio` 让老年代更大。
  3. **减少碎片**：开启 `-XX:+UseCMSCompactAtFullCollection` 和 `-XX:CMSFullGCsBeforeCompaction=1`，但会增加 Full GC 停顿。
  4. **终极方案**：升级到 JDK 11+ 并切换至 G1/ZGC。

##### 16.3.4 G1 Mixed GC 停顿超预期
- **现象**：G1 的 Mixed GC 停顿时间远超 `MaxGCPauseMillis`（默认 200ms）。
- **原因**：
  - Region 过大（`-XX:G1HeapRegionSize` 超过默认值），导致复制对象耗时增加。
  - 并发标记周期未完成，Mixed GC 需要回退标记。
  - RSet 占用内存过高，扫描耗时增加。
  - 大对象 Region（Humongous）碎片化，导致复制困难。

- **解决方案**：
  1. **调整 Region 大小**：堆内存 / 2048 自动计算，可手动设为 2MB、4MB、8MB、16MB 等 2 的幂。避免超过 32MB。
  2. **控制并发标记阈值**：降低 `-XX:InitiatingHeapOccupancyPercent`（如 35），提前标记，避免突然涌入大量对象。
  3. **增加 Mixed GC 次数**：`-XX:G1MixedGCCountTarget=16`（默认 8），让每次回收更少 Region，降低单次停顿。
  4. **启用并行引用处理**：`-XX:+ParallelRefProcEnabled`。

---

#### 16.4 CPU 飙高的 GC 原因及排查

##### 16.4.1 频繁 GC 导致 CPU 飙升
- **现象**：`top` 看到 `%sys` 或 `%user` 高，`jstat -gcutil` 显示 GC 活动频繁。
- **原因**：GC 线程（如 Parallel GC 的多个线程）执行标记、复制、整理，大量消耗 CPU。
- **排查**：用 `jstat -gc <pid> 1000` 观察 YGC/FGC 频率。若每秒 GC 次数 > 5，则 CPU 高很可能是 GC 造成。
- **解决**：见 16.3 节，核心是减少 GC 频率。

##### 16.4.2 JIT 编译线程占满 CPU
- **现象**：`top -Hp` 中看到 `C2 CompilerThread` 或 `C1 CompilerThread` 占用高 CPU。
- **原因**：
  - 系统刚启动，大量方法正在编译。
  - CodeCache 满，导致编译线程停止并不断重试（日志出现 `CodeCache is full`）。
  - 分层编译默认开启，C1 和 C2 同时工作。

- **解决方案**：
  1. 检查 CodeCache 使用率：`jstat -codecache <pid>`。若已满，增大 `-XX:ReservedCodeCacheSize=256m`（默认 240MB）。
  2. 监控编译日志：`-XX:+PrintCompilation`，观察是否有反复编译（如方法过大无法内联）。
  3. 关闭分层编译（不推荐）：`-XX:-TieredCompilation`。

##### 16.4.3 正则表达式或字符串操作导致回溯 CPU 飙升
- **现象**：CPU 飙高但 GC 正常，`jstack` 看到线程在 `Pattern.matcher` 或 `String.split`。
- **原因**：Java 正则使用回溯算法，输入数据长或模式复杂时导致指数级计算。
- **解决方案**：
  - 使用 `String.indexOf` 或 `String.replace` 等简单方法替代正则。
  - 限制输入字符串长度。
  - 使用 `java.util.regex.Pattern` 预编译并复用。

---

#### 16.5 线程阻塞与死锁

##### 16.5.1 死锁
- **现象**：应用完全无响应，`jstack -l <pid>` 输出 `Found one Java-level deadlock`。
- **常见原因**：多线程以不同顺序获取同一组锁。
- **解决方案**：
  - 代码修复：统一锁顺序，或使用 `tryLock` 定时加锁。
  - 避免使用多个 `synchronized` 方法嵌套调用。

##### 16.5.2 线程池任务堆积
- **现象**：请求响应越来越慢，`jstack` 中看到大量线程处于 `WAITING`（`park`）或 `TIMED_WAITING`，但队列长度很大。
- **原因**：线程池核心线程数过小，阻塞队列无界，导致任务积压，最终 OOM 或超时。
- **解决方案**：
  - 使用有界队列（`ArrayBlockingQueue`、`LinkedBlockingQueue` 带 capacity）。
  - 设置 `RejectedExecutionHandler`（如 `CallerRunsPolicy` 做背压）。
  - 监控线程池指标（`ThreadPoolExecutor` 的 `getQueue().size()`）。

##### 16.5.3 死锁与锁竞争的高并发优化
- **大数据高并发场景**：使用 `ConcurrentHashMap` 代替 `Hashtable`；使用 `LongAdder` 代替 `AtomicLong`；使用 `ReadWriteLock` 或 `StampedLock` 优化读多写少。
- **避免使用 `synchronized` 方法**，缩小锁粒度。
- **利用 `CompletableFuture` 异步编程**减少阻塞。

---

#### 16.6 元空间/类加载泄漏（大数据框架常见）

##### 场景一：Groovy 动态脚本
- **问题**：使用 `GroovyClassLoader.parseClass` 每次都会生成新类，且类加载器未释放，导致 Metaspace 泄漏。
- **解决**：使用 `GroovyShell` 配合缓存编译后的 `Class`，或使用 `@CompileStatic` 减少动态类生成。

##### 场景二：热部署（如 Tomcat、Spring DevTools）
- **问题**：每次重新部署时，旧的 `WebappClassLoader` 无法回收，因为一些线程或静态引用仍持有它。
- **解决**：
  - 使用 `-XX:+TraceClassUnloading` 查看类卸载情况。
  - 确保所有线程（如 `TimerThread`、`ScheduledExecutorService`）在卸载前关闭。
  - 在 Tomcat 配置中设置 `clearReferencesStatic` 和 `clearReferencesThreads`。

##### 场景三：CGLIB 动态代理
- **问题**：Spring AOP 或 Mock 框架生成大量代理类，每个代理类都占用 Metaspace。
- **解决**：
  - 限制 `-XX:MaxMetaspaceSize` 并监控。
  - 使用 `java.lang.reflect.Proxy` 替代 CGLIB（基于接口）。
  - 升级到 Java 11+，利用动态类归档（CDS）减少内存占用。

---

#### 16.7 堆外内存泄漏（Netty、NIO 典型问题）

##### 现象
- 堆内存正常（MAT 分析无泄漏），但 `top` 看到 RES 远超 Xmx，系统 OOM Killer 杀死进程。

##### 常见原因
- **Netty 中的 `ByteBuf` 未释放**（未调用 `ReferenceCountUtil.release`）。
- **NIO 的 `DirectByteBuffer` 分配后，`Cleaner` 未及时运行**（GC 不可控）。
- **JNI 本地代码分配的内存未释放**。

##### 解决方案
1. **启用 NMT**：`-XX:NativeMemoryTracking=detail`，然后 `jcmd <pid> VM.native_memory detail` 查看分类。
2. **Netty 调优**：
   - 使用 `-Dio.netty.noPreferDirect=false` 避免过度使用直接内存。
   - 设置 `-Dio.netty.maxDirectMemory=0` 让 Netty 使用堆内存。
   - 定期调用 `System.gc()`（需评估对 GC 的影响）。
3. **代码最佳实践**：
   - 使用 `try-finally` 释放 `ByteBuf`：`try { ... } finally { buf.release(); }`
   - 在池化分配器（`PooledByteBufAllocator`）中，确保每个 `ByteBuf` 被释放。

---

#### 16.8 容器环境（K8s/Docker）特有坑

##### 16.8.1 JVM 不感知容器内存限制
- **现象**：容器内存 limit 设为 4G，但 JVM 的 `-Xmx` 自动探测到宿主机内存（如 128G），导致容器 OOM Kill。
- **原因**：JDK 8u191 之前，JVM 未开启 `UseContainerSupport`。
- **解决**：
  - JDK 8u191+：添加 `-XX:+UseCGroupMemoryLimitForHeap -XX:+UseContainerSupport`
  - JDK 10+：默认开启 `UseContainerSupport`，直接使用 `-XX:MaxRAMPercentage=75.0`。
  - 避免写死 `-Xmx`，以免容器规格变更时失效。

##### 16.8.2 CPU 限流导致 GC 停顿增加
- **现象**：容器设置了 CPU limit（如 2 cores），但 JVM 的 GC 线程数仍为物理 CPU 数（如 32），导致线程上下文切换严重，GC 停顿增加。
- **解决**：JVM 9+ 默认 `ActiveProcessorCount` 会感知容器 CPU 限制。若版本较低，手动设置 `-XX:ParallelGCThreads=2`（与容器 CPU 一致）。

##### 16.8.3 容器内存太小导致频繁 GC
- **现象**：容器内存 limit 4G，堆分配 3G，元空间+直接内存+线程栈等容易超过 4G，引起 OOM Kill。
- **解决**：合理规划 `-Xmx` 应小于容器内存的 80%，并预留直接内存和元空间。推荐使用 `-XX:MaxRAMPercentage=75.0` 自动计算。

---

#### 16.9 大数据高并发场景下的 JVM 调优 checklist

以下清单专为大数据流处理（如 Kafka Streams、Flink、Spark Streaming）和高并发微服务设计：

1. **合理评估对象大小**：利用 `jol`（Java Object Layout）估算典型业务对象的大小，确定 Eden 区是否能容纳一次请求周期内的临时对象。
2. **避免大对象**：将大集合拆分成小批次，避免超过 `PretenureSizeThreshold` 或 G1 Region 一半的对象。
3. **使用对象池**：对于 byte[]、ByteBuffer、ProtoBuf 等频繁分配的大对象，使用 `commons-pool2` 或 Netty 的 `Recycler`。
4. **选择正确的收集器**：
   - 堆 < 8GB，延迟要求一般：Parallel GC。
   - 堆 8~32GB，延迟要求中等：G1。
   - 堆 > 32GB，延迟敏感：ZGC（JDK 17+）。
5. **调整 G1 的 Mixed GC 次数**：`-XX:G1MixedGCCountTarget=16`，降低单次停顿。
6. **对于 Flink / Spark**：设置 `-XX:+UseG1GC -XX:MaxGCPauseMillis=100`，并开启 `-XX:+ParallelRefProcEnabled`。
7. **监控和告警**：接入 Prometheus + JMX Exporter，采集 `java_lang_GarbageCollector_CollectionTime` 和 `java_lang_MemoryPool_UsageUsed`。设置 GC 停顿 > 1s 告警。
8. **压测验证**：生产流量回放或使用 JMH 模拟高并发分配，调整 TLAB 和 Survivor 比例。

---

### 附：高频问题速查表

| 问题 | 典型日志/现象 | 第一步排查命令 | 常用解决方案 |
| :--- | :--- | :--- | :--- |
| **Java heap space OOM** | `OutOfMemoryError: Java heap space` | `jmap -dump:live,format=b,file=heap.hprof <pid>` | MAT 分析大对象，修复内存泄漏，增大堆 |
| **Metaspace OOM** | `OutOfMemoryError: Metaspace` | `jstat -gc <pid>` 看 MU 接近 MC | 增大 MaxMetaspaceSize，检查类加载泄漏 |
| **Direct buffer OOM** | `OutOfMemoryError: Direct buffer memory` | `jcmd <pid> VM.native_memory summary` | 检查 Netty/NIO 内存泄漏，增大 MaxDirectMemorySize |
| **Unable to create native thread** | `unable to create new native thread` | `ulimit -u`，`ps -eLf | wc -l` | 减小 -Xss，使用有界线程池 |
| **Full GC 频繁** | GC 日志显示 Full GC 每分钟多次 | `jstat -gcutil <pid> 1000` 观察 FGC | 增大 Survivor 区，调整晋升阈值，切 G1/ZGC |
| **CMS concurrent mode failure** | GC 日志出现此短语并退化为 Serial Old | 查看 CMS 触发时老年代占用 | 降低 CMSInitiatingOccupancyFraction，升级 G1 |
| **G1 Mixed GC 停顿过长** | `MaxGCPauseMillis` 不生效 | `-XX:+PrintGCDetails` 查看 Mixed GC 耗时 | 减小 Region 大小，增加 MixedGC 次数 |
| **CPU 飙高且 GC 线程忙** | `top -Hp` 看到 GC 线程 | `jstat -gcutil` 高 YGC/FGC | 优化分配速率，增大 Eden |
| **CPU 飙高且编译器线程忙** | `C2 CompilerThread` 高 CPU | `jstat -codecache` 看已满 | 增大 ReservedCodeCacheSize |
| **线程死锁** | 应用卡死，`jstack` 输出 deadlock | `jstack -l <pid>` | 代码统一锁顺序或使用 tryLock |
| **容器 OOM Kill** | `dmesg` 显示 `Out of memory: Kill process` | `kubectl top pod` 看内存 | 开启 UseContainerSupport，使用 MaxRAMPercentage |

---
*本文档基于 OpenJDK 官方文档、HotSpot 源码分析以及生产环境经验整理，是系统学习 JVM 和日常调优的可靠参考。*