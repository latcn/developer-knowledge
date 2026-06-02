
> **适用版本**：Redis 7.0+（同步标注 6.x 差异）
> **学习路径说明**：本文档每节按“核心思想 → 设计推导 → 关键实现 → 实战要点 → 进阶思考题”五段式组织，建议按顺序阅读，每节掌握后再进入下一节。
---
## 第一章 架构设计剖析

### 1.1 单线程事件循环模型

#### 核心思想
Redis 的核心操作（读写命令执行）在很长一段时间内是单线程的。这不是缺陷，而是一种极度聚焦的设计选择——用简单换取极致的可预测性和低延迟。

**费曼类比**：想象一个超级高效的咖啡师。他一次只做一杯咖啡，但动作极快，从不浪费时间在切换任务上。顾客排队点单，他依次处理，每杯咖啡的制作时间完全可预测。这就是 Redis 单线程的本质。

#### 设计推导（第一性原理）

我们从三个基础事实出发：

1. **基础事实一**：Redis 是内存数据库。内存读写延迟约 100ns，加上内存分配（jemalloc）开销约 50-200ns，而网络 I/O 延迟约 100μs-1ms。
2. **基础事实二**：CPU 通常是 Redis 的瓶颈（内存分配、数据结构操作、协议解析），而非磁盘 I/O。
3. **基础事实三**：多线程引入锁竞争、上下文切换开销（约 1-10μs）、并发 bug 风险。

**推导逻辑**：
- 既然操作瓶颈在 CPU 和内存访问而非磁盘 I/O，那么多线程并不能加速单个命令的执行（CPU 核心只有一个能真正执行该命令）
- 单线程避免了所有锁开销，每个命令的执行时间完全可预测
- 真正耗时的是网络读写，可以通过 I/O 多路复用（epoll/kqueue）来解决

**为什么 6.0 又引入多线程？**
随着万兆网络普及，网络包的解析（读取请求、解析协议）和响应写回成为了新瓶颈——一个单线程处理 10 万 QPS 时，约 30-40% 的 CPU 时间花在网络协议处理上。Redis 6.0 引入的多线程**仅用于网络 I/O**（读请求解析、写响应发送），命令执行依然是单线程，保持了核心的简洁性。这个特性默认关闭，需配置：

```bash
io-threads 4                    # I/O 线程数（建议 ≤ CPU 核数）
io-threads-do-reads yes         # 开启读多线程（默认只写多线程）
```

**单线程的性能极限推算**：
- 假设每个命令平均耗时 1μs（纯内存操作），单线程理论上限 = 100 万 QPS
- 实际中考虑内存分配、协议解析、网络 I/O，单线程约 10-20 万 QPS（GET/SET 等简单命令）

#### 关键实现

**事件循环四阶段**：
```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  beforesleep │    │  aeApiPoll   │ → │  readQuery   │ → │  processCmd  │ → │  sendReply   │
│  定时任务    │    │  等待事件    │    │  读取+解析    │    │  (纯单线程)  │    │  写回客户端   │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
     ↑                                                          │
     └──────────────── 回到循环 ───────────────────────────────┘
```

**核心代码路径**：
```
aeMain (src/ae.c)
  → while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);   # beforesleep 在事件处理之前执行
        aeProcessEvents(eventLoop, ...);
    }
```
`beforesleep` 回调中注册的任务包括：
- `activeExpireCycle()`       # 定期过期键删除
- `flushAppendOnlyFile()`     # AOF 刷盘
- `replicationCron()`        # 复制相关定时任务
- `clientsCron()`            # 客户端超时处理

`aeProcessEvents` 内部处理可读/可写事件：
- 可读事件 → `readQueryFromClient()`
- 可写事件 → `sendReplyToClient()`

#### 实战要点

- **危险操作（O(n) 阻塞单线程）**：
  - `KEYS *`：遍历所有键，百万键耗时秒级
  - `FLUSHALL`/`FLUSHDB`：同步模式下 O(n)
  - `SORT` 大列表：O(n log n)
  - `SSCAN`/`HSCAN`/`ZSCAN` 虽然分页，但单次 `COUNT` 过大会阻塞

- **安全的替代**：
  - `KEYS` → `SCAN`（渐进式迭代，每批 10-100 个）
  - `FLUSHALL ASYNC` / `FLUSHDB ASYNC`（Redis 4.0+，后台线程异步清空）
  - `DEL` 大对象 → `UNLINK`（Redis 4.0+，异步释放内存）

- **慢查询监控配置**：
  ```bash
  slowlog-log-slower-than 10000   # 超过 10ms 记录（生产建议值）
  slowlog-max-len 128             # 保留最近 128 条
  ```

#### 进阶思考题
> 如果让你设计一个多线程版本的 Redis 命令执行引擎，你会如何划分数据分区以避免锁竞争？这种设计在 Redis Cluster 模式下是否已经部分实现？（提示：Cluster 每个分片依然是单线程，多个分片间天然并行）

---

### 1.2 数据结构设计哲学：SDS、Quicklist、Skiplist

#### 核心思想
Redis 不直接使用 C 语言原生的字符串、数组、链表，而是为每个数据结构重新造了轮子。每个定制结构都有明确的“要解决的问题 → 设计选择 → 付出的代价”三部曲。

**费曼类比**：C 语言的字符串就像一张白纸，写多长就是多长，想要改更长的内容就得换一张新纸。Redis 的 SDS（简单动态字符串）则像一块带刻度的写字板，你随时知道已经写了多少、还剩多少空间，要追加内容不用每次都换板子。

---

#### SDS（Simple Dynamic String）

**解决的问题**：
- C 字符串获取长度是 O(n)（需要遍历到 `\0`），而 Redis 频繁需要获取键值长度
- C 字符串拼接容易缓冲区溢出，追加操作效率低
- C 字符串不能存储二进制数据（`\0` 会被截断）

**设计选择（Redis 3.2+ 多头 SDS）**：

Redis 3.2 起，SDS 引入了多种头部类型以优化不同长度字符串的内存占用：

```c
// Redis 7.0 实际结构（五种头部类型）
// sdshdr5：用于极短字符串（< 32字节），不单独存储 len/alloc（节省空间）
// sdshdr8/sdshdr16/sdshdr32/sdshdr64：用于不同长度范围的字符串

struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len;         // 已使用长度（1 字节）
    uint8_t alloc;       // 分配的总长度，不含头部和 \0（1 字节）
    unsigned char flags; // 头部类型标识，低 3 位表示类型（1 字节）
    char buf[];          // 灵活数组，实际数据
};

struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len;        // 2 字节
    uint16_t alloc;      // 2 字节
    unsigned char flags; // 1 字节
    char buf[];
};
// sdshdr32 使用 uint32_t（4字节），sdshdr64 使用 uint64_t（8字节）
```

**与旧版 SDS 的关键变化**：

| 维度 | 旧版（Redis < 3.2） | 新版（Redis ≥ 3.2） |
|------|-------------------|-------------------|
| 字段 | `int len; int free` | `uint8/16/32/64_t len/alloc` |
| 语义 | `free` 表示剩余空间 | `alloc` 表示总分配空间（不含头部） |
| 头部大小 | 固定 8 字节 | 1-17 字节（按需选择） |
| 内存优化 | 无 | 短字符串头部仅 1-3 字节 |

**预分配策略**（`sdsMakeRoomFor()` 函数）：
```c
// 新长度 < 1MB：翻倍分配
if (newlen < SDS_MAX_PREALLOC)  // SDS_MAX_PREALLOC = 1MB (1024*1024)
    newlen *= 2;
// 新长度 ≥ 1MB：增加 1MB
else
    newlen += SDS_MAX_PREALLOC;
```

**embstr 阈值推导**（Redis 7.0 验证）：
```c
// embstr 阈值计算（为什么是 44 字节）：
// jemalloc 分配 bin 大小：8, 16, 32, 48, 64, 80, 96, 112, 128...
// embstr 使用 64 字节 bin（能容纳的最合适大小）
// redisObject: 16 字节（type + encoding + lru + refcount + ptr）
// sdshdr8: 3 字节（len + alloc + flags）
// 空终止符: 1 字节（\0）
// 可用 payload: 64 - 16 - 3 - 1 = 44 字节
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
```

**代价分析**：
- 内存开销：短字符串 1-3 字节头部，中等字符串 5 字节，长字符串 9-17 字节
- 预分配策略：用空间换时间，减少内存重分配次数。但可能导致 50% 的内存浪费（极端情况下）

---

#### Quicklist（快速列表）

**解决的问题**：
- 普通链表：每个节点独立内存分配，指针开销大（每个元素至少 2 个指针 = 16 字节），且内存不连续，缓存不友好
- 压缩列表（ziplist）：内存紧凑，但连锁更新风险，插入/删除可能 O(n²)

**Redis 的演进路径**：
```
Redis < 3.2:  List = linkedlist（小数据）或 ziplist（大数据）
Redis 3.2:    List = quicklist（统一实现，节点内 ziplist）
Redis 7.0:    quicklist 节点内替换为 listpack
```

**Quicklist 结构**：
```
Quicklist 结构示意：
┌─────────────────────────────────────────────────┐
│ quicklist                                        │
│  ├─ head → quicklistNode                         │
│  │         ├─ prev                              │
│  │         ├─ listpack (最大 8KB，可配置)       │
│  │         │   ├─ entry1, entry2, entry3        │
│  │         ├─ next → quicklistNode               │
│  │                    ├─ prev                    │
│  │                    ├─ listpack                │
│  │                    │   ├─ entry4, entry5      │
│  │                    ├─ next → ...              │
│  ├─ tail → 最后一个节点                           │
│  ├─ count: 5（总元素数）                         │
│  ├─ len: 2（节点数）                             │
└─────────────────────────────────────────────────┘
```

**Quicklist 设计哲学——组合优于继承**：
- 宏观上是一个双向链表，支持 O(1) 的头尾操作（LPUSH/RPOP 等）
- 每个节点内部是一个紧凑的 listpack，内存连续，缓存友好
- 将 O(n) 的链表随机访问优化为 O(n/k)，k 为每个节点的平均元素数

**关键配置**：
```bash
# Redis 7.0 的配置（旧版 list-max-ziplist-size 自动映射）
list-max-listpack-size -2
# 取值含义：
#  -5: 最大 64KB  （不推荐）
#  -4: 最大 32KB
#  -3: 最大 16KB
#  -2: 最大 8KB   （默认，性能与内存的平衡点）
#  -1: 最大 4KB
#   正数: 精确的元素数量（如 5 = 每个节点最多 5 个元素）

list-compress-depth 0  # 两端不压缩的节点数（0 = 不压缩，默认）
# 1: 头尾各 1 个节点不压缩，其余用 LZF 压缩
# 注意：压缩节点在被访问时需要解压，增加 CPU 开销
# 适用场景：大列表，且大部分访问集中在头尾（如时间线）
# 不适用场景：需要频繁随机访问列表中间元素
```

---

#### Skiplist（跳跃表）

**解决的问题**：
- 有序集合（ZSet）需要按分数范围查询，同时也需要快速按成员查找
- 平衡树（如 AVL、红黑树）实现复杂，且范围查询不如链表直观

**设计选择**：
```
跳表结构示意（索引层级，每个节点随机分配层数）：
Level 2:  Head ──────────────────────→ [55] ──→ NULL
Level 1:  Head ──────→ [23] ────────→ [55] ──→ [78] → NULL
Level 0:  Head → [10] → [23] → [37] → [55] → [78] → [90] → NULL
```
Redis 的跳表最多 32 层，每个节点的层数通过随机算法生成。

**关键修正：层数随机概率是 25%，不是 50%**：
```c
// src/t_zset.c 中的实际实现
#define ZSKIPLIST_P 0.25      // 上升概率 25%
#define ZSKIPLIST_MAXLEVEL 32 // 最大 32 层

int zslRandomLevel(void) {
    int level = 1;
    // 每次有 25% 概率上升一层
    while ((random() & 0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level < ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```

**为什么选择 25%？**
- 50% 概率：平均每个节点 2 层，即每个节点平均 2 个前进指针 → 内存开销大
- 25% 概率：平均每个节点 1/(1-0.25) = 1.33 个前进指针 → 内存效率更高
- 平均查找复杂度依然是 O(log n)（底数变大，常数因子增加，但仍在可接受范围）

**空间开销补充说明**：每个跳表节点还包含一个**后退指针**（backward），用于反向遍历。因此每个节点实际平均指针数约为 1.33 (前进) + 1 (后退) = **2.33 个**。红黑树通常有 3 个指针（左、右、父节点），跳表在内存效率上仍略有优势。

**与平衡树的对比优势**：

| 维度 | 跳表（Redis 实现） | 红黑树 |
|------|-------------------|--------|
| 实现复杂度 | 简单，约 200 行 C 代码 | 复杂，约 500+ 行 |
| 范围查询 | O(log n + m)，直接遍历第 0 层 | 需要中序遍历，实现复杂 |
| 平均指针数 | 约 2.33 个/节点（前进+后退） | 3 个/节点（左+右+父） |
| 并发友好 | 可分段加锁 | 全局锁代价高 |
| 层数上限 | 32 | N/A |

**跳表与字典的组合（ZSet 的完整结构）**：
```
ZSet 内部结构：
┌────────────────────────────────┐
│  zset                          │
│  ├─ dict (哈希表)              │
│  │   member → score           │  ← 支持 O(1) 按成员查找分数
│  │                             │
│  └─ zskiplist (跳表)          │
│      Node: member, score      │  ← 支持按分数范围查询
│      Node: member, score      │     支持 O(log n) 排名查询
│      ...                      │
└────────────────────────────────┘
```
两个结构共享 member 和 score 对象，通过指针复用避免数据冗余。

#### 实战要点

- **SDS 优化**：短键（< 44 字节）使用 embstr，内存利用率最高。键名设计应尽量短，但可读性优先
- **Quicklist 调优**：
  - 队列场景（FIFO）：`list-max-listpack-size -2`（默认），`list-compress-depth 0`
  - 大列表 + 少访问中间：增大 `list-compress-depth`（如 2-3），利用 LZF 压缩节省内存
- **跳表随机性**：跳表的层数随机性在极端情况下可能导致 O(n) 退化（概率极低，约 1/4^31 ≈ 1/2^62）
- **内存优化公式**：
  ```
  纯整数列表（10 万元素）：
    纯 listpack（紧凑编码）：  约 800KB
    quicklist（默认 8KB 节点）：约 900KB（增加节点指针）
    纯链表：                  约 1.6MB（每个元素 16 字节指针）
  ```

#### 进阶思考题
> Quicklist 是“listpack 的链表”，这种混合结构将查找时间从 O(n) 优化到了 O(n/k)（k 是 listpack 分段数）。如果让你设计一个“哈希表的 listpack 变体”，你会如何平衡查找速度和内存紧凑性？（提示：小型 Hash 直接用 listpack，O(n) 顺序查找在小数据量下反而比 O(1) 哈希更快，因为缓存友好、无哈希计算开销）

---

### 1.3 持久化机制：RDB vs AOF 的第一性推导

#### 核心思想
持久化本质上是在“数据安全性”、“恢复速度”、“运行时性能”三个维度之间做权衡。Redis 提供了两种策略及其混合模式，允许你根据业务特性自由选择或组合使用。

#### 设计推导（第一性原理）

**从三个维度反向推导**：

| 维度 | RDB（快照） | AOF（追加日志） |
|------|------------|----------------|
| **数据安全性** | 两次快照间的数据会丢失（通常秒级到分钟级） | 可配置为每条命令 fsync，最多丢失 1 秒数据 |
| **恢复速度** | 极快，直接加载二进制快照到内存 | 慢，需逐条重放命令（通常比 RDB 大数倍） |
| **运行时性能** | fork 子进程写入，父进程几乎无影响，但 fork 瞬间可能阻塞 | 持续写入磁盘，fsync 策略影响吞吐 |

**推导过程**：

1. **为什么需要 RDB？** 假设你每天备份一次，恢复时只需加载最新快照。这就像数据库的全量备份——恢复最快，但增量数据会丢失。

2. **为什么需要 AOF？** 假设你想最多丢失 1 秒数据，那你需要记录每一秒内的所有写操作。这就像数据库的 binlog——数据更安全，但恢复时需要重放所有日志。

3. **为什么两者结合？** Redis 4.0 引入混合持久化：RDB 做快照基础 + AOF 记录增量。Redis 7.0 的 Multi-Part AOF 进一步优化了这一机制。

**fork 子进程的关键设计（Copy-on-Write）**：
- `fork()` 时父进程和子进程共享同一块物理内存，内核将页表标记为只读
- 当父进程要修改某页时，内核触发页错误，复制该页给父进程（COW），子进程继续使用原页写入 RDB 文件
- **推导出瓶颈**：如果写操作频繁，COW 复制大量内存页，导致内存翻倍的风险

#### 关键实现

**RDB 触发条件**（满足任一即触发）：
```bash
save 3600 1    # 3600秒（1小时）内至少 1 次修改
save 300 100   # 300秒（5分钟）内至少 100 次修改
save 60 10000  # 60秒（1分钟）内至少 10000 次修改
```
两个核心函数：`rdbSaveBackground()` 负责 fork 子进程，`rdbSave()` 在子进程中执行实际写入。RDB 文件默认使用 LZF 压缩（`rdbcompression yes`）。

**AOF 三种刷盘策略及性能影响**：
```bash
appendfsync always    # 每条命令 fsync
# 数据最安全，但性能最差
# SATA SSD：约 1000-5000 QPS
# NVMe SSD：约 10000-50000 QPS
# 性能下降 10-100 倍（相比不 fsync）

appendfsync everysec  # 每秒 fsync（默认推荐）
# 最多丢失 1 秒数据
# 性能损失约 5-15%（相比不 fsync）
# 极少数情况：磁盘写入超过 1 秒时，Redis 会阻塞等待

appendfsync no        # 由操作系统决定何时刷盘
# 性能最高，但可能丢失 30 秒数据
# 仅适用于可以承受大量数据丢失的场景（如纯缓存）
```

**Multi-Part AOF（Redis 7.0 重大改进）**：

Redis 7.0 之前，AOF 是一个不断增长的单文件，重写时通过 fork 子进程生成新的单文件 AOF。7.0 引入了 Multi-Part AOF：

```
AOF 目录结构（7.0+）：
┌────────────────────────────────────────────┐
│ appendonlydir/                              │
│  ├─ appendonly.aof.manifest  ← 清单文件    │
│  │   file appendonly.aof.1.base.rdb seq 1 type b
│  │   file appendonly.aof.1.incr.aof seq 1 type i
│  │   file appendonly.aof.1.incr.aof seq 2 type i
│  ├─ appendonly.aof.1.base.rdb ← 基础文件   │
│  ├─ appendonly.aof.1.incr.aof ← 增量文件1  │
│  └─ appendonly.aof.1.incr.aof ← 增量文件2  │
└────────────────────────────────────────────┘

工作原理：
1. 第一次启动或重写后：生成新的 base 文件（RDB 格式）和 manifest 清单
2. 后续写入：追加到 incr 增量文件（AOF 格式）
3. 下次重写：生成新的 base + manifest，旧文件被清理
4. 恢复：按 manifest 顺序加载 base 和各 incr 文件
5. 崩溃恢复：如果重写中途崩溃，旧的 manifest 仍然有效，系统可恢复
```

**优势**：
- 重写只需要生成新的 base 文件，历史增量文件保持不变
- 重写失败不影响现有数据（新文件原子替换清单）
- base 使用 RDB 格式，恢复速度更快

**AOF 重写触发条件**：
```bash
auto-aof-rewrite-percentage 100  # 当前 AOF 总大小是上次重写后的 2 倍
auto-aof-rewrite-min-size 64mb  # 且 AOF 总大小至少 64MB
```
重写过程 fork 子进程，利用 COW 机制生成新的 base 文件。

#### 实战要点

**生产环境推荐配置**：
```bash
# Redis 7.0+ 配置（混合持久化通过 aof-use-rdb-preamble 启用 RDB 格式 base 文件）
aof-use-rdb-preamble yes         # base 文件使用 RDB 格式（恢复更快）
appendfsync everysec             # 每秒刷盘
save 3600 1                      # RDB 兜底（即使有 AOF 也建议保留）
save 300 100
save 60 10000
```

**容量规划公式**：
- RDB 文件大小 ≈ 内存占用 × 0.5-0.7（LZF 压缩后）
- AOF 文件大小（重写前）≈ 写入量 × 时间（可能达到内存的 2-3 倍）
- Multi-Part AOF 总大小 ≈ base 文件（≈ RDB 大小）+ 增量文件大小
- 磁盘预留 = MAX(RDB 最大文件 × 保留份数, AOF 总大小) × 1.3（30% 余量）

**常见陷阱与排查**：

1. **fork 阻塞**：内存越大，fork 时间越长
   ```
   10GB 内存：fork 约 100-200ms
   50GB 内存：fork 约 500-1000ms
   监控指标：latest_fork_usec（INFO Stats）
   解决方案：使用物理机、关闭 THP（Transparent Huge Pages）
   ```

2. **COW 内存爆炸**：写密集型场景，`used_memory` 可能翻倍
   ```
   原因：父进程大量写操作，触发 COW 复制内存页
   监控：used_memory_rss / used_memory 的比值
   预防：预留至少 50% 物理内存余量
   ```

3. **AOF 刷盘抖动**：
   ```
   现象：everysec 模式下出现毫秒级延迟毛刺
   原因：磁盘写入超过 1 秒，Redis 阻塞等待
   解决：使用 NVMe SSD、fsync 改为 no（如果接受丢数据）
   ```

#### 进阶思考题
> 如果让你设计一个“永不丢失数据”的 Redis 持久化方案，除了将 `appendfsync` 设为 `always`，还有哪些架构层面的方案？（提示：多副本同步写入、共享存储、WAL 日志的变体、Raft 共识的持久化层）

---

## 第二章 核心原理与算法深度解析

### 2.1 过期键淘汰策略：惰性删除 + 定期删除

#### 核心思想
Redis 的键过期处理采用**惰性删除**（访问时检查）和**定期删除**（后台主动扫描）的双重机制。两者结合，既避免了定时器开销（每个键设一个定时器太昂贵），又防止了过期键长期占用内存。

**费曼类比**：想象一个冰箱管理过期食物的两种方式。惰性删除是“打开冰箱看到坏掉的才扔”；定期删除是“每隔一段时间抽查几个格子，坏掉的全清掉”。前者保证不浪费精力，后者保证不会堆积太多。

#### 设计推导

**如果只用惰性删除**：一个键过期后再无访问，它将永远占用内存（内存泄漏）

**如果只用定时器删除（每个键一个 timer）**：百万级过期键需要百万个定时器，消耗巨大 CPU 和内存

**Redis 的折中方案**：
1. **惰性删除**：每次访问键时检查 `expireIfNeeded()`，过期则删除。零额外开销。
2. **定期删除**：每秒执行 `server.hz` 次（默认 10），每次随机抽取 20 个带过期时间的键，删除其中过期的；如果过期比例超过 25%，继续重复此过程，但限制总执行时间防止卡顿。

**Redis 7.0 的动态 hz 特性**：
```bash
hz 10                # 基础频率（默认 10，范围 1-500）
dynamic-hz yes       # 7.0 起默认开启，自动根据客户端连接数调整 hz
# 大量连接 → hz 自动升高（更快处理过期键和客户端超时）
# 空闲时 → hz 自动降低（节省 CPU）
```

#### 关键实现

**定期删除核心逻辑**（`activeExpireCycle` 函数简化）：
```c
// src/expire.c 中实现
void activeExpireCycle(int type) {
    // 计算单次执行的时间预算（微秒）
    // 公式：timelimit = 1000000 * ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC / server.hz / 100
    // ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC 默认为 25，表示最多占用 25% 的 CPU 时间
    // 当 server.hz = 10 时，timelimit = 1000000 * 25 / 10 / 100 = 25000 微秒 = 25 毫秒
    timelimit = ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC * 1000000 / server.hz / 100;

    for each db {
        do {
            // 随机取 20 个带过期时间的键
            expired = 0;
            sampled = 0;
            while (sampled < 20 && !timelimit_exceeded) {
                key = randomKey(db->expires);
                if (isExpired(key)) {
                    deleteKey(key);
                    expired++;
                }
                sampled++;
            }
            // 过期比例超过 25% 且未超时，继续循环
        } while (expired > 20/4 && !timelimit_exceeded);
    }
}
```

**关于 `active-expire-effort` 配置**（Redis 4.0+）：
此配置可间接调整过期清理的“力度”，默认值为 1，范围 1-10。它会动态调整 `timelimit`、每次循环扫描的键数等参数，但并非简单的线性叠加。调高该值会占用更多 CPU，适合过期键密集的场景。

**惰性删除触发点**：
- 所有读写命令执行前（`lookupKeyRead`、`lookupKeyWrite` 内部调用 `expireIfNeeded`）
- `EXPIRE`、`SETEX`、`EXPIREAT` 等命令设置新过期时间时

#### 最大内存淘汰策略

当 `maxmemory` 达到上限，Redis 需要额外策略决定淘汰哪些键：

| 策略 | 行为 | 适用场景 |
|------|------|---------|
| `noeviction` | 拒绝写入，返回错误 | 缓存不允许丢失 |
| `allkeys-lru` | 在所有键中淘汰最近最少使用的 | 通用缓存（推荐） |
| `volatile-lru` | 只在有过期时间的键中淘汰 LRU | 持久数据 + 临时缓存混合 |
| `allkeys-lfu` | 在所有键中淘汰最不经常使用的 | 热点数据保护 |
| `volatile-lfu` | 只在有过期时间的键中淘汰 LFU | 同上，保护持久数据 |
| `allkeys-random` | 随机淘汰 | 数据访问模式均匀 |
| `volatile-random` | 随机淘汰有过期时间的键 | 同上 |
| `volatile-ttl` | 淘汰最接近过期的键 | 期望优先保留长 TTL |

**LRU vs LFU 的 Redis 特化实现**：

```c
// Redis LFU 使用 redisObject 中的 24 位 lru 字段
// 高 16 位：上次衰减时间（分钟级时间戳）
// 低 8 位：对数计数器（访问频率，范围 0-255，默认从 5 开始）
//
// 对数增长：防止高频键过快达到上限
// counter = 1 时，每访问 1 次 +1
// counter = 255 时，需要约 1 百万次访问才能 +1
// 衰减：随时间流逝，counter 定期减少
```

#### 实战要点

- **避免大量键同时过期**：如果大量键在同一时刻过期，定期删除会反复循环，导致 CPU 尖峰。解决方案：在 EXPIRE 时间上加随机偏移
  ```python
  # 错误做法
  redis.expire(key, 86400)  # 所有键同时过期

  # 正确做法
  import random
  redis.expire(key, 86400 + random.randint(0, 3600))  # 24h + 0-1h 随机偏移
  ```

- **LRU vs LFU 选择指南**：
  - LRU：适合“最近访问的会再次访问”的场景（如用户会话、活动中的临时缓存）
  - LFU：适合“频繁访问的才是热数据”的场景（如热门文章、商品详情），但需要额外内存记录频率

- **监控指标**：
  ```bash
  INFO Stats | grep evicted
  # evicted_keys：被淘汰的键数量
  # 持续增长 → 内存不足 → 扩容或调整策略

  INFO Keyspace | grep expires
  # keyspace_hits / keyspace_misses：缓存命中率
  ```

#### 进阶思考题
> 为什么 Redis 不采用标准的双向链表实现 LRU（像传统操作系统那样），而是使用近似 LRU（随机采样）？这种“近似而高效”的设计哲学在 Redis 的哪些其他地方也有体现？（提示：采样数量由 `maxmemory-samples` 控制，默认 5，值越大越精确但 CPU 开销越大）

---

### 2.2 主从复制、哨兵、集群的分布式演进

#### 核心思想
Redis 的高可用路线图是一个不断演进的过程：单机 → 主从复制（数据冗余） → 哨兵（自动故障转移） → 集群（水平分片）。每一步都解决前一步的核心痛点。

**费曼类比**：这就像一个公司的扩张过程——开始是一个人单干（单机），忙不过来就招个助手复制工作（主从），自己生病时让助手自动顶上（哨兵），最后业务太大需要分部门各管一摊（集群）。

#### 设计推导（如果让你设计，会怎么做？）

**问题一：单机故障怎么办？**
→ 答案：加一个备份机，复制数据（主从复制）

**问题二：主库故障，谁来做切换？**
→ 答案：加一个“监工”监控主库，故障时自动提拔从库（哨兵）

**问题三：数据量或写入量超过单机容量？**
→ 答案：把数据按规则分到多个主库（集群分片）

**为什么主从复制是异步的？**
- 同步复制：每条命令必须等从库确认 → 性能暴跌到网络延迟级别（RTT）
- 半同步复制：等至少一个从库确认 → 性能略有下降，但主从都故障时丢数据
- 异步复制（Redis 默认）：主库不等待从库确认 → 性能最佳，但主库宕机时可能丢数据
- 这是 **CAP 定理** 在 Redis 中的体现：选择了 AP（可用性 + 分区容错），牺牲了一致性（C）

#### 主从复制深度解析

**全量复制阶段**（首次连接或重连太晚）：
```
Slave: PSYNC ? -1
Master: +FULLRESYNC <replid> <offset>
Master: fork子进程生成RDB → 发送给Slave
Master: 同时将新写入命令缓冲到：
        ├─ replication buffer (该从库专用，用于发送)
        └─ replication backlog (所有从库共享的环形缓冲区)
Slave: 加载RDB → 执行缓冲的命令
```

**增量复制阶段**（短暂断连后重连）：
```
Slave: PSYNC <replid> <offset>
Master: 检查 repl_backlog 环形缓冲区
        if offset 还在缓冲区内:
            返回 +CONTINUE，发送 backlog 中 offset 后的增量命令
        else:
            触发全量复制
```

**三个关键缓冲区**：

| 缓冲区 | 用途 | 配置参数 |
|--------|------|---------|
| replication backlog | 所有从库共享，支持增量复制 | `repl-backlog-size`（默认 1MB） |
| replication buffer | 每个从库独立，用于全量/增量发送 | `client-output-buffer-limit slave` |
| 客户端输出缓冲区 | 普通客户端的响应缓冲 | `client-output-buffer-limit normal` |

**⚠️ 生产环境警告**：`repl-backlog-size` 默认值仅为 1MB，对于写入量稍大的场景（如 5MB/s），0.2 秒即会填满。短暂网络抖动就可能导致全量复制。**务必根据写入速率调大该值**，推荐 256MB~1GB，计算公式：`写入速率(MB/s) × 最大断线重连时间(秒) × 2`。

**三个致命问题与对策**：
1. **复制风暴**：多从库同时全量同步，主库 fork 多次
   - 对策：从库错峰启动、使用树状复制结构（从库的从库）
2. **repl_backlog 溢出**：写量大 + 断线时间长，导致反复全量复制
   - 对策：如上调大 `repl-backlog-size`
3. **数据不一致**：主库宕机时，从库可能没收到最后几条命令
   - 对策：业务接受少量数据丢失 + 幂等设计，或使用 `WAIT` 命令做半同步

**`WAIT` 命令**（Redis 3.0+）：
```bash
SET key value
WAIT numreplicas timeout  # 阻塞直到至少 numreplicas 个从库确认
# 实现半同步复制，但性能下降明显
```

#### 哨兵（Sentinel）

**哨兵的三项任务**：
```
1. 监控：定期 PING 主从库，标记主观下线（SDOWN）和客观下线（ODOWN）
2. 通知：通过 Pub/Sub 通知客户端主库变更
3. 自动故障转移：选主、晋升、重配置
```

**故障转移完整流程**：
```
1. 哨兵 A 发现主库无响应 → 标记 SDOWN
2. 哨兵 A 询问其他哨兵 → 达到 quorum 数量 → 标记 ODOWN
3. 哨兵集群执行 Leader 选举（算法与 Raft 类似，需 majority 票数）
4. Leader 从所有从库中选择最优者：
   a. 优先级高（slave-priority 配置，越小越高）
   b. 复制偏移量大（数据最新）
   c. runid 字典序小（进一步决策）
5. Leader 执行 SLAVEOF NO ONE → 晋升为新主库
6. 其他从库指向新主库 → SLAVEOF <新主IP> <端口>
7. 旧主库恢复 → 自动成为新主库的从库
```

**推荐部署架构**：
```
最少 3 个哨兵节点（奇数个，分别在不同物理机/机架）
哨兵通信端口：26379
quorum = 2  # 至少 2 个哨兵同意才能故障转移
# 注意：故障转移执行还需要 majority（3 个哨兵中 > 1.5 = 2 票）
# 所以 3 哨兵 + quorum=2 是生产最小配置
```

**哨兵连接方式**：
```python
# 客户端连接哨兵获取主库地址
from redis.sentinel import Sentinel
sentinel = Sentinel([('sentinel1', 26379), ('sentinel2', 26379), ('sentinel3', 26379)])
master = sentinel.master_for('mymaster', socket_timeout=0.1)
slave = sentinel.slave_for('mymaster', socket_timeout=0.1)
```

#### Redis Cluster（集群分片）

**16384 个哈希槽设计推导**：
- 为什么是 16384？因为心跳消息中携带槽位信息用位图表示，16384 位 = 16384/8 = 2048 字节 = **2KB**
- 在 1Gbps 网络下，2KB 的心跳包开销可接受
- 如果是 65536 槽则位图 8KB，心跳包太大，节点数多时网络开销显著
- CRC16 算法计算：`slot = crc16(key) & 16383`（等价于 % 16384）

**客户端重定向流程**：
```bash
# 客户端向错误的节点发送命令
Client → Node A: GET user:1001
Node A: -MOVED 3999 10.0.0.3:6379  # 永久重定向
# 智能客户端（如 JedisCluster）会缓存槽位映射，减少重定向

# 槽迁移中的重定向
Client → Node A: GET user:1001
Node A: -ASK 3999 10.0.0.3:6379   # 临时重定向（槽正在迁移）
Client → Node B: ASKING            # 确认接受 ASK 重定向
Client → Node B: GET user:1001
```

**集群模式下的限制**：
- 不支持多键跨槽操作（除非用 hash tag `{...}` 强制同槽）
  ```bash
  # 错误：两个键可能在不同槽
  MSET user:1:name "Alice" user:2:name "Bob"

  # 正确：使用 hash tag 强制同槽
  MSET {user}:1:name "Alice" {user}:2:name "Bob"
  # hash tag 规则：对 {} 之间的部分计算 CRC16
  ```
- 不支持事务跨槽
- 批量操作（如 SINTER 两个集合）需要两个集合在同一槽
- **🔴 发布订阅不可用**：在 Redis Cluster 模式下，`PUBLISH`/`SUBSCRIBE` 命令会返回 `CROSSSLOT` 错误。如需类似功能，请使用 Redis Stream 或专业消息队列。

**集群 Gossip 协议**：
- 每个节点维护集群中所有节点的信息
- 通过 PING/PONG 消息定期交换，传播节点信息和槽配置
- 去中心化设计，无单点故障
- **节点规模建议**：官方建议集群节点数不超过 1000。生产环境中超过 200 节点时，Gossip 消息开销会明显增加，需谨慎规划。

#### 实战要点

**复制参数优化公式**：
```bash
# repl_backlog 大小（生产务必调大）
repl-backlog-size = 主库每秒写入量(MB) × 最长断线恢复时间(秒) × 2
# 示例：5MB/s × 120s × 2 = 1200MB

# 从库输出缓冲区
client-output-buffer-limit slave 256mb 64mb 60
# 硬限制 256MB，软限制 64MB，持续超过软限制 60 秒后断开
```

**哨兵部署清单**：
```
✅ 至少 3 个哨兵，奇数个
✅ 哨兵不要和 Redis 节点在同一物理机
✅ 所有哨兵配置一致（特别是 quorum 和 mymaster 名称）
✅ 哨兵之间的防火墙规则放通 26379
✅ 配置 sentinel down-after-milliseconds（建议 5000-15000ms）
```

**集群容量规划**：
```
总主节点数 = ceil(总数据量(GB) / 单节点建议内存(15-20GB))
总节点数 = 总主节点数 × (1 + 从库副本数)

# 示例：100GB 数据，每节点 20GB，1 主 2 从
# 主节点 = ceil(100/20) = 5
# 总节点 = 5 × 3 = 15 个节点
# 加上 3 个哨兵 = 18 个进程
```

**集群配置 `cluster-require-full-coverage`**：
- 该配置控制当集群中任意哈希槽不可用时是否拒绝写入。
- 默认值在 Redis 5.0 中由 `yes` 改为 `no`。
- **`yes`（强一致）**：任何槽不可用则整个集群拒绝所有查询，适合金融等严苛场景。
- **`no`（高可用）**：允许部分槽故障时其他槽继续服务，推荐用于缓存或可接受部分数据丢失的场景。

#### 进阶思考题
> Redis Cluster 使用“去中心化”的 Gossip 协议同步集群状态，而非依赖中心协调器。这种设计在节点数量扩展到 1000+ 时，Gossip 消息的开销会如何增长？有哪些可能的优化方向？（提示：Redis 7.0 引入了 `cluster-announce-hostname` 等新特性改善大规模集群管理）

---

### 2.3 事务与 Lua 脚本的原子性保证

#### 核心思想
Redis 的事务（`MULTI/EXEC`）与关系型数据库的 ACID 事务有本质区别。它保证的是**隔离性**（命令串行执行，不被其他客户端打断）而非原子性（失败不自动回滚）。

**费曼类比**：Redis 事务就像一个没有“撤销”按钮的命令队列。你说“开始排队”，然后把要做的事依次写下来，最后说“执行”。执行时别人不能插队，但如果中间某条指令失败了，前面的不会撤销。

#### 设计推导

**为什么 Redis 事务不做回滚？**
- Redis 语法错误在入队时就会被发现（`MULTI` 阶段检查命令语法），直接拒绝执行
- 运行时错误（如对字符串执行 `INCR`）是编程错误，应该在开发阶段发现
- 回滚机制增加复杂度（需要 undo log），违背了 Redis 追求简单的设计哲学
- 在内存数据库中，回滚的代价可能很高（复杂数据结构的逆操作）

**Lua 脚本的出现**：
事务的局限性在于“不能做判断”——你无法在事务中说“如果 GET 的值是 10，就 DEL 这个键”。Lua 脚本解决了这个问题，它在 Redis 内部执行，天然具有**隔离性**（整个脚本执行期间 Redis 不处理其他命令）。

> **技术澄清**：日常讨论中常将 Lua 脚本称为“原子性”，实际是指 **隔离性（Isolation）**。脚本执行期间虽不会被其他命令打断，但如果脚本运行中出现错误（如对字符串执行 `INCR`），已执行的写操作**不会自动回滚**。这与 `MULTI/EXEC` 事务的行为一致。在官方语境下，描述为“原子方式执行”时特指隔离性。

#### 关键实现

**事务执行流程**：
```
Client: MULTI
Redis: +OK
Client: SET key1 "value1"
Redis: +QUEUED
Client: INCR key2
Redis: +QUEUED
Client: EXEC
Redis: 1) OK
       2) (integer) 1

# 放弃事务
Client: MULTI
Client: SET key1 "value1"
Client: DISCARD  # 清空队列
Redis: +OK
```

**WATCH 实现乐观锁（Optimistic Locking）**：
```
Client A: WATCH balance          # 监视 balance 键
Client A: GET balance → "100"
Client A: MULTI
Client A: DECRBY balance 50      # 入队：减 50
# ———— 此时 Client B 修改了 balance ————
Client B: SET balance 200
# ———— Client A 的事务会失败 ————
Client A: EXEC
Redis: (nil)                     # 执行失败，需要重试
Client A: UNWATCH
Client A: WATCH balance          # 重新监视并重试整个流程
```

> **术语说明**：WATCH + MULTI/EXEC 实现的是**乐观并发控制（Optimistic Concurrency Control, OCC）**，其原理类似于 CAS 的思想——比较预期值，不一致则重试。

**Lua 脚本的隔离性**：
```lua
-- 原子性地限流（滑动窗口）
local current = redis.call('INCR', KEYS[1])
if current == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[1])
end
if current > tonumber(ARGV[2]) then
    return 0  -- 超过限制
end
return 1  -- 允许通过
```
脚本执行期间，Redis 进入“脚本模式”，所有其他命令被阻塞。

#### 实战要点

**事务 vs Lua 脚本选择指南**：

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| 简单的批量无依赖写操作 | MULTI/EXEC | 简单，无需 Lua 解析开销 |
| 需要读后判断再写入（如库存扣减） | Lua 脚本 | 隔离性 + 逻辑判断 |
| 需要隔离性 + 回滚能力 | Lua + 手动补偿 | Redis 不支持自动回滚 |
| 分布式事务 | 不推荐 Redis | 考虑 TCC 模式或关系型数据库 |
| 复杂计算 + 写入 | Lua 脚本 | 减少网络往返 |

**Lua 脚本重要注意事项**：
```
1. 🔴 执行时间限制：lua-time-limit 5000（单位：毫秒 = 5秒）
   超过 5 秒后，Redis 开始接受 SCRIPT KILL 命令
   如果脚本已经执行过写操作，SCRIPT KILL 无效 → 只能 SHUTDOWN NOSAVE

2. 谨慎使用随机操作（SPOP、RANDOMKEY 等）
   在 Redis 3.2+ 中，脚本效果复制机制可处理随机操作，但仍可能导致性能问题或主从数据不一致，建议尽量避免。

3. 脚本缓存优化：
   SCRIPT LOAD "return redis.call('GET', KEYS[1])"  # 返回 SHA1
   EVALSHA <sha1> 1 mykey                           # 用 SHA1 调用
   避免每次发送完整脚本，减少网络带宽

4. 集群模式下：脚本操作的所有键必须在同一槽
   解决方案：使用 hash tag，如 {user:1001}:name 和 {user:1001}:age
```

**SCRIPT KILL vs SHUTDOWN NOSAVE**：
```bash
# 脚本只读（未修改数据），可以杀死
> SCRIPT KILL
OK

# 脚本已修改数据，SCRIPT KILL 报错
> SCRIPT KILL
(error) UNKILLABLE Sorry the script already executed write commands...
# 只能关停服务器（不保存数据）
> SHUTDOWN NOSAVE
```

#### 进阶思考题
> 如果要实现一个“抢红包”功能，使用 Redis 事务还是 Lua 脚本？请写出核心逻辑。如果金额需要加密或复杂算法计算，Redis 还能胜任吗？边界在哪里？（提示：Redis Lua 脚本适合轻量级逻辑，复杂业务逻辑应该放在应用层，Redis 只做数据存储）

---

## 第三章 主要功能模块详解

### 3.1 8 种数据类型与编码转换规则

#### 核心思想
Redis 的每种数据类型都有多种底层编码实现，系统会根据元素数量和大小自动切换，在“内存效率”和“访问速度”之间动态平衡。

**重要版本变更**：Redis 7.0 起，ziplist 已被 **listpack** 全面替代。listpack 通过记录当前 entry 长度（而非前一个 entry 长度），从根本上解决了连锁更新问题。

#### listpack 替代 ziplist 的技术原因

**ziplist 连锁更新问题**：
```
ziplist 每个 entry 记录前一个 entry 的长度（prevlen）
prevlen 字段可变长：前一个 entry < 254 字节 → 1 字节，否则 → 5 字节

连锁更新触发：
  修改 entry(N) 使其变大 → entry(N+1) 的 prevlen 需要膨胀（1B→5B）
  → entry(N+1) 变大 → entry(N+2) 的 prevlen 也需要膨胀
  → ...最坏情况 O(n²)

listpack 的解决方案：
  每个 entry 记录自己的长度（在末尾），而非前一个 entry 的长度
  修改 entry(N) 只影响自己，不影响相邻 entry 的元数据
  从后向前遍历：读末尾的 len 字段 → 跳转到前一个 entry
```

**配置项名称变更**（Redis 7.0+）：

| 旧名称（≤6.x） | 新名称（7.0+） | 用途 |
|----------------|----------------|------|
| `hash-max-ziplist-entries` | `hash-max-listpack-entries` | Hash 使用 listpack 的最大字段数 |
| `hash-max-ziplist-value` | `hash-max-listpack-value` | Hash 字段值的最大字节数 |
| `zset-max-ziplist-entries` | `zset-max-listpack-entries` | ZSet 使用 listpack 的最大元素数 |
| `zset-max-ziplist-value` | `zset-max-listpack-value` | ZSet 成员的最大字节数（仅限 member，score 始终为 8 字节 double） |
| `list-max-ziplist-size` | `list-max-listpack-size` | List (quicklist) 每节点大小 |

> 注意：从旧版本升级时，旧配置项会被自动映射到新配置项，但建议手动更新配置文件。

#### 编码转换规则矩阵（Redis 7.0+）

| 类型 | 编码 1（小数据） | 编码 2（大数据） | 转换阈值（Redis 7.0+） |
|------|----------------|---------------|----------------------|
| **String** | `embstr`（≤44字节） | `raw`（>44字节） | 自动转换 |
| **String** | `int`（纯整数） | `raw` | 值变为非整数时转换 |
| **Hash** | `listpack` | `hashtable` | `hash-max-listpack-entries: 512`<br>`hash-max-listpack-value: 64` |
| **List** | `quicklist`（内部节点 listpack） | - | `list-max-listpack-size: -2`（8KB） |
| **Set** | `intset`（有序整数数组） | `hashtable` | `set-max-intset-entries: 512` |
| **ZSet** | `listpack` | `skiplist` | `zset-max-listpack-entries: 128`<br>`zset-max-listpack-value: 64` |
| **Stream** | `rax`（基数树）+ 节点内 `listpack` | - | `stream-node-max-bytes: 4096`<br>`stream-node-max-entries: 100` |
| **Bitmap** | 基于 String 的位操作 | - | 最大 2^32 位（512MB） |
| **HyperLogLog** | 稀疏编码 → 稠密编码 | - | 自动转换（稠密固定 12KB） |
| **GEO** | ZSet（52位 GeoHash 整数为 score，成员名为 member） | - | 同 ZSet，支持 listpack/skiplist 编码转换 |

> **ZSet 编码说明**：`skiplist` 编码内部包含 `dict`（member→score 映射，O(1) 查找）+ `zskiplist`（按 score 排序，范围查询），两个结构共享 member 和 score 对象。
>
> **Stream 编码说明**：Stream 的宏观结构是 Radix Tree（`rax`），以消息 ID 的时间戳部分为键组织节点。每个 `rax` 节点内部使用 `listpack` 存储该时间段内的多条消息。所以 Stream = `rax`（宏观索引）+ `listpack`（微观存储）。
>
> **intset 编码说明**：`intset` 内部有三种编码（int16/int32/int64），根据元素最大值自动选择。编码只升级不降级：即使删除大整数，编码也不会回退为更小的整数类型。

#### 编码转换示例与权衡

**Hash 的 listpack → hashtable 转换**：
```bash
# 场景：存储用户属性（小数据，使用 listpack）
HSET user:1001 name "张三" age 25 city "北京"  # 使用 listpack，极省内存

# 当字段超过 512 个或某个 value 超过 64 字节：
# 自动转换为 hashtable，内存翻 2-3 倍，但 O(1) 查询更快
```

**内存对比实测数据**（100 个 field 的 Hash）：
| 编码 | 内存占用（约） | GET 单个 field | GET 所有 field |
|------|-------------|--------------|---------------|
| listpack | ~800 字节 | O(n) 顺序扫描 | O(n) |
| hashtable | ~2500 字节 | O(1) 哈希查找 | O(n) |

**小数据量时 listpack 的 O(n) 反而更快**：
- 内存连续 → CPU 缓存友好
- 无哈希计算开销
- 临界点通常在 50-100 个元素，超过后哈希表优势明显

**Set 的 intset 编码**：
```bash
# 纯整数的 set 使用 intset（有序数组 + 二分查找）
SADD numbers 1 2 3 4 5
OBJECT ENCODING numbers  # "intset"（默认 int16 编码）

# intset 会根据整数范围自动升级
SADD numbers 100000      # 升级为 int32 编码
SADD numbers 3000000000  # 升级为 int64 编码

# 编码只升级不降级：即使 DEL 大整数，编码也不回退
# 一旦加入非整数元素，转换为 hashtable（不可逆）：
SADD numbers "hello"
OBJECT ENCODING numbers  # "hashtable"
```

#### 实战要点

**优化选择指南**：
- **社交粉丝列表**（纯整数用户 ID）：Set + intset，内存占用极低
- **商品属性**（字段少且值短）：Hash + listpack，比较 hashtable 节省约 60% 内存
- **排行榜**：ZSet + skiplist，支持分数范围查询和排名（skiplist 内部含 dict + zskiplist）
- **消息队列（7.0+ 推荐）**：Stream + 消费者组，相比 List 增加了确认机制、回溯消费、消费者组负载均衡
- **用户签到**：Bitmap，1 亿用户 365 天仅需约 4.6GB

**注意陷阱**：
```bash
# 危险：listpack 下的 O(n) 全量操作
HGETALL large_hash  # 如果还是 listpack 但元素很多，返回慢
LLEN long_list      # quicklist 的 LLEN 是 O(1)（头部记录了总长度）

# intset 的不可逆转换
SADD myset 1 2 3    # intset（int16 编码）
SADD myset "hello"  # 转为 hashtable，无法回到 intset（即使 DEL "hello"）
```

#### 进阶思考题
> 如果让你设计一个“哈希表的 listpack 变体”，你会如何平衡查找速度和内存紧凑性？（提示：可以借鉴 quicklist 的思路——多个小型 listpack + 索引结构）

---

### 3.2 发布订阅、Stream、Pipeline 对比

#### 核心思想
这三个功能都是解决“多消息传递”问题，但可靠性、持久化、性能特征完全不同，需要根据场景精确选择。

#### 三维对比矩阵

| 维度 | Pub/Sub | Stream | Pipeline |
|------|---------|--------|----------|
| **消息持久化** | ❌ 不持久，离线丢失 | ✅ 持久化到磁盘（AOF/RDB） | N/A（命令批处理） |
| **消息确认** | ❌ 无确认 | ✅ 消费者组 + ACK | ❌ 无 |
| **投递语义** | 最多一次（fire-and-forget） | **最多一次**（常规）<br>通过 `XCLAIM` 可支持故障恢复场景的至少一次 | N/A |
| **回溯消费** | ❌ 不支持 | ✅ 支持（按 ID 或时间范围） | ❌ 不支持 |
| **消费者组** | ❌ 无 | ✅ 支持（竞争/广播模式） | N/A |
| **典型吞吐量** | ~10万 消息/秒 | ~8万 消息/秒 | 命令打包，减少 RTT×N |
| **典型延迟** | < 1ms（同机房） | < 2ms（同机房） | 取决于命令执行时间 |
| **集群行为** | **不可用（返回 CROSSSLOT 错误）** | 按槽路由（无广播） | 同普通命令 |
| **适用场景** | 实时通知、聊天消息（仅限单机/主从） | 可靠消息队列、事件溯源 | 批量写入/读取 |

#### Pub/Sub 核心机制

```bash
# 发布者
PUBLISH channel:news "突发新闻: Redis 7.4 发布!"

# 订阅者（订阅后阻塞等待消息）
SUBSCRIBE channel:news
# 或模式匹配
PSUBSCRIBE channel:*
```

**消息流图**：
```
Publisher → [Redis 内部 pubsub_channels / pubsub_patterns 字典]
              ├→ Subscriber1（订阅 channel:news）
              ├→ Subscriber2（订阅 channel:news）
              └→ Subscriber3（PSUBSCRIBE channel:*）
```
消息不存储，直接推送到所有匹配的订阅者。订阅者不在线则消息消失（fire-and-forget）。

**⚠️ 集群中的 Pub/Sub 不可用**：
在 Redis Cluster 模式下，`PUBLISH` 和 `SUBSCRIBE` 命令会返回 `CROSSSLOT` 错误，因为频道名无法映射到哈希槽。如需在集群环境中使用发布订阅，请考虑：
- 使用 Stream 替代（推荐）
- 在客户端层面自行实现基于 topic 的路由
- 使用外部消息队列（如 Kafka、RabbitMQ）

自 **Redis 7.0** 起，正式引入了 **分片 Pub/Sub**（Sharded Pub/Sub），通过以下命令在集群中实现类似功能：

| 命令             | 作用       | 说明                     |
| -------------- | -------- | ---------------------- |
| `SSUBSCRIBE`   | 订阅分片频道   | 频道名会根据哈希槽分配到特定节点       |
| `SPUBLISH`     | 发布到分片频道  | 消息仅发送到负责该槽的节点，不再广播到全集群 |
| `SUNSUBSCRIBE` | 取消订阅     | 对应取消                   |
| `SPUBSUB`      | 查看分片订阅状态 | 类似 `PUBSUB` 但用于分片      |

**工作原理**：
- 分片频道的处理与普通键一致：`slot = crc16(channel_name) & 16383`。
- 发布者只需将消息发送到对应槽的节点，订阅者也只监听该节点。
- 消息不会跨节点广播，因此性能远优于模拟广播。

**适用场景**：
- 需要在 Redis Cluster 中实现轻量级实时通知（如在线状态、配置变更）。
- 对消息丢失不敏感（仍为 fire-and-forget，无持久化）。

**示例**：
```bash
# 客户端 A（连接至槽位对应的节点）
> SSUBSCRIBE notifications
Reading messages... (press Ctrl-C to quit)

# 客户端 B（连接至同一槽位的节点，可任意节点）
> SPUBLISH notifications "Hello cluster"
(integer) 1

# 客户端 A 收到消息
1) "ssubscribe"
2) "notifications"
3) (integer) 1
4) "Hello cluster"
```

> **重要提示**：分片 Pub/Sub 不会与普通 Pub/Sub 混合。如果集群中既使用 `PUBLISH` 又使用 `SPUBLISH`，两者相互独立。原有业务如需迁移到集群，应改用 `SSUBSCRIBE` / `SPUBLISH`。

#### Stream 核心机制

Stream 的底层使用 `rax`（基数树）+ 节点内 `listpack`：

```
Stream 数据结构示意：
┌─────────────────────────────────────────────────┐
│ rax (基数树，以消息 ID 为键)                      │
│  ├─ [timestamp1-*] → listpack                    │
│  │   ├─ msg1: {id: "t1-0", name: "Alice", ...}  │
│  │   └─ msg2: {id: "t1-1", name: "Bob", ...}    │
│  ├─ [timestamp2-*] → listpack                    │
│  │   ├─ msg3: {id: "t2-0", name: "Charlie", ...}│
│  │   └─ msg4: {id: "t2-1", name: "Diana", ...}  │
│  └─ ...                                          │
└─────────────────────────────────────────────────┘

消息 ID 格式：<毫秒时间戳>-<序列号>
例如：1623456789000-0
```

**消费者组模型**：
```
消费者组 "order-processors"
├── Consumer A (处理中: msg1, msg4)     ← Pending Entries List (PEL)
├── Consumer B (空闲)
└── Consumer C (处理中: msg3)

工作流程：
1. XREADGROUP GROUP group1 consumer1 STREAMS mystream >
   → 自动分配未投递的消息给消费者
2. 处理完成后 XACK mystream group1 <msg_id>
   → 从 PEL 中移除，标记为已确认
3. 如果消费者崩溃未 ACK：
   → 消息留在 PEL 中
   → 其他消费者通过 XCLAIM 认领超时消息（用于故障恢复，非常规语义）
```

**关键命令**：
```bash
# 添加消息（* 表示自动生成 ID）
XADD mystream * field1 value1 field2 value2

# 限制 Stream 长度（~ 表示近似，性能更好）
XADD mystream MAXLEN ~ 10000 * field value
XTRIM mystream MAXLEN ~ 10000

# 创建消费者组（$ 从最新开始，0 从头开始）
XGROUP CREATE mystream group1 $ MKSTREAM

# 消费者组读取（> 表示只读未投递的消息）
XREADGROUP GROUP group1 consumer1 COUNT 10 STREAMS mystream >

# 确认处理
XACK mystream group1 1623456789000-0

# 查看待处理消息
XPENDING mystream group1 - + 100  # 前 100 条
# 输出: 1) 总未确认数 2) 最早未确认ID 3) 最晚未确认ID 4) 每个消费者的未确认数

# 认领超时消息（用于故障恢复，3600000ms = 1小时未确认）
XCLAIM mystream group1 consumer2 3600000 1623456789000-0
```

**投递语义：最多一次（常规） + 故障恢复的至少一次**：
- **正常流程**：消息被消费者读取并 `XACK` 后，从 PEL 中移除，确保**不会重复**。
- **故障场景**：消费者崩溃未发送 `XACK`，消息留在 PEL。其他消费者可通过 `XCLAIM` 认领并重新处理，此时消息可能被处理**多次**。
- **结论**：Stream 的设计目标是**最多一次**（防止常规重复），`XCLAIM` 是为**故障恢复**设计的例外机制，不应依赖其实现常规的“至少一次”。业务层仍建议做幂等处理。

#### Pipeline

Pipeline 纯粹是网络优化——将多个命令打包发送，减少 RTT（往返时间）：
```
无 Pipeline:
  Client → [CMD1] → Redis → [R1] → Client  } RTT1
  Client → [CMD2] → Redis → [R2] → Client  } RTT2
  Client → [CMD3] → Redis → [R3] → Client  } RTT3
  总耗时：3 × RTT + 3 × 命令执行时间

有 Pipeline:
  Client → [CMD1, CMD2, CMD3] → Redis → [R1, R2, R3] → Client
  总耗时：1 × RTT + 3 × 命令执行时间

示例计算（RTT = 1ms，忽略命令执行时间）：
  无 Pipeline：1000 个 GET → 1000 × 1ms = 1 秒
  有 Pipeline（批大小 100）：10 × 1ms = 10ms
```

**吞吐量数据（本地回环）**：
- 单命令串行：约 2-5 万 QPS
- Pipeline（100 条/批）：约 50-100 万+ QPS
- 提升倍数 ≈ 批大小（直到 Redis 服务端 CPU 瓶颈）

**注意事项**：
- Pipeline 不保证隔离性，命令间可能被其他客户端的命令插入
- 单次 Pipeline 命令不宜太多（建议 100-1000 条），否则消耗过多内存和 CPU
- 集群模式需按槽分组后再 Pipeline（同槽的命令才能 Pipeline 到同一节点）

#### 实战要点

**选型决策树**：
```
需要可靠消息，支持回溯和消费者组？
    YES → Stream
    NO → 仅是客户端批量操作优化？
        YES → Pipeline
        NO → 实时推送，接受丢失，且不在集群环境？
            YES → Pub/Sub（仅限单机/主从）
            NO → 考虑专业消息队列（Kafka/RabbitMQ/Pulsar）
```

**Stream 生产实践**：
```bash
# 1. 消息长度限制（防止无限增长）
XADD mystream MAXLEN ~ 100000 * data "..."

# 2. 死信处理（长期 PEL 中的消息）
# 定时任务：监控 XPENDING 长度，超时自动 XCLAIM 或转移到死信 Stream
XPENDING mystream group1 - + 100
XCLAIM mystream group1 dead_letter_consumer 3600000 <msg_ids>

# 3. 消费者组扩展
# 同一消费者组内增加消费者自动负载均衡（XREADGROUP）
# 消费者总数不应超过 Stream 分区数（默认 1 个分区）
# 可通过多个 Stream 实现分区扩展
```

#### 进阶思考题
> Stream 的消费者组看起来很像 Kafka 的 Consumer Group，但 Redis Stream 是单分区的（所有消息在一个 Stream 中），而 Kafka 是多分区并行的。如果要实现“高并发的严格全局有序消息”，Redis Stream 有什么限制？如何突破？（提示：多个 Stream + 业务层路由）

---

### 3.3 可观测性：慢查询、监视器、键空间通知

#### 核心思想
生产环境的 Redis 需要全面的可观测性。Redis 提供了三种不同层次的“内窥”工具：慢查询日志（性能诊断）、监视器（实时调试）、键空间通知（事件驱动）。

#### 慢查询日志（Slow Log）

**记录机制**：
1. 每个命令执行前记录开始时间（微秒精度）
2. 执行完成后计算耗时（不含 I/O 时间，仅命令执行时间）
3. 如果超过 `slowlog-log-slower-than`，记录到固定大小的队列
4. 队列最多保存 `slowlog-max-len` 条记录（FIFO，超出后淘汰最旧的）

**核心配置**：
```bash
slowlog-log-slower-than 10000   # 超过 10 毫秒记录（生产建议值）
slowlog-max-len 128             # 保留 128 条慢查询（可按需调大）
```

**读取与分析**：
```bash
# 查看最近 10 条慢查询
SLOWLOG GET 10
# 返回：
# 1) 1) (integer) 12345          # 唯一 ID
#    2) (integer) 1623456789     # 发生时间戳
#    3) (integer) 15234          # 执行时间（微秒）
#    4) 1) "KEYS"                # 命令及参数
#       2) "*"
#    5) "127.0.0.1:52341"       # 客户端地址
#    6) "client-name"            # 客户端名称（如有）

# 获取慢查询数量
SLOWLOG LEN

# 清空慢查询记录
SLOWLOG RESET
```

**常见慢查询根因及优化**：

| 慢查询原因 | 示例 | 解决方案 |
|-----------|------|---------|
| 遍历所有键 | `KEYS *` | 改用 `SCAN` |
| 大范围排序 | `SORT biglist` | 限制排序范围或预排序存储到 ZSet |
| 批量删除 | `DEL bigkey` | 改用 `UNLINK`（异步删除） |
| 大范围集合运算 | `SINTER set1 set2 set3` | 限制集合大小，或用 `SINTERSTORE` + 分批 |
| 内存不足导致 swap | N/A | 增加内存或调整 `maxmemory` |
| 过期键集中清理 | 大量键同时过期 | 随机化过期时间 |
| fork 阻塞 | RDB/AOF rewrite | 减少单实例内存，使用多实例 |

#### 监视器（MONITOR）

```bash
MONITOR  # 实时打印服务器收到的每条命令
# 输出：
# 1623456789.123456 [0 127.0.0.1:52341] "SET" "key1" "value1"
# 1623456789.234567 [0 127.0.0.1:52342] "GET" "key2"
# ...
```

**注意事项**：
- ⚠️ **性能杀手**：启用 MONITOR 后，Redis 吞吐量下降约 50%（每条命令需额外输出到监视器客户端）
- ⚠️ **输出缓冲区风险**：如果监视器客户端读取速度跟不上，输出缓冲区会膨胀，可能导致 OOM
- 仅用于临时调试，生产环境严禁长期开启
- 可替代方案：`redis-cli --stat`、`redis-cli --latency`、慢查询日志

#### 键空间通知（Keyspace Notifications）

键空间通知通过 Pub/Sub 发送键空间变化事件，允许应用程序监听键的变化。

**配置**：
```bash
notify-keyspace-events ""        # 默认关闭
# 配置字符串格式：
# K：键空间通知（__keyspace@<db>__:<key>）
# E：键事件通知（__keyevent@<db>__:<event>）
# g：通用命令（DEL, EXPIRE, RENAME...）
# $：字符串命令
# l：列表命令
# s：集合命令
# h：哈希命令
# z：有序集合命令
# x：过期事件
# e：淘汰事件
# A：g$lshzxe 的别名（"All"）

# 常用配置示例：
notify-keyspace-events "Ex"      # 仅监听过期事件
notify-keyspace-events "KExe"    # 键空间+键事件：过期和淘汰
notify-keyspace-events "AKE"     # 所有事件
```

**两种通知格式**：
```bash
# 配置 "KExe" 后，当键过期时：
# 频道 1：__keyspace@0__:mykey     消息：expire
# 频道 2：__keyevent@0__:expired   消息：mykey

# 应用层订阅示例（Python）：
pubsub = redis.pubsub()
pubsub.psubscribe('__keyevent@0__:expired')
for message in pubsub.listen():
    if message['type'] == 'pmessage':
        expired_key = message['data']
        # 处理过期键
```

**重要限制与常见陷阱**：
- 通知是“尽力而为”的（fire-and-forget），不保证送达
- 如果订阅者离线，期间的事件会丢失
- 键过期后，通知发出时间和实际过期时间可能有延迟（取决于惰性删除+定期删除策略）
- 通知不会随 RDB/AOF 持久化，重启后不会重放
- 大量通知可能导致 CPU 和网络开销
- **集群模式下 Pub/Sub 不可用**，因此键空间通知同样无法在集群中使用

#### 实战要点

**生产环境可观测性清单**：
```bash
# 1. 性能监控
redis-cli --stat               # 实时统计
redis-cli --latency            # 延迟监控
redis-cli --latency-dist       # 延迟分布（需提前采集）
redis-cli --bigkeys            # 大键扫描（建议在从库执行）

# 2. INFO 关键指标
# INFO Server    # redis_version, uptime_in_seconds
# INFO Clients   # connected_clients, blocked_clients
# INFO Memory    # used_memory, used_memory_rss, mem_fragmentation_ratio
# INFO Stats     # instantaneous_ops_per_sec, evicted_keys, keyspace_hits/misses
# INFO Replication  # role, connected_slaves, master_repl_offset
# INFO Persistence  # rdb_last_save_time, aof_last_write_status
INFO Stats | grep -E "instantaneous_ops_per_sec|evicted_keys|keyspace_hits|keyspace_misses"
INFO Memory | grep -E "used_memory_rss|used_memory|mem_fragmentation_ratio"
INFO Replication | grep -E "master_repl_offset|slave_repl_offset"
INFO Persistence | grep -E "rdb_last_save_time|aof_last_write_status"

# 3. 告警阈值建议
instantaneous_ops_per_sec 骤降 > 50%    → 可能有问题
mem_fragmentation_ratio > 1.5          → 内存碎片高
evicted_keys 持续增长                   → 内存不足
master_repl_offset - slave_repl_offset > 10MB → 复制延迟大
slowlog 中 O(n) 命令频繁出现            → 设计问题
```

**键空间通知的典型应用**：
- **缓存失效联动**：缓存过期时自动更新（如 DB 数据变更后过期缓存）
- **分布式锁续约**：监听锁键的过期事件，自动续约或通知业务
- **延迟任务**：利用过期通知实现简单的延迟队列（精度有限，不保证可靠送达）

#### 进阶思考题
> 键空间通知是“尽力而为”的，可能有事件丢失。如果业务要求“绝不丢失过期事件”，你会如何设计替代方案？（提示：Stream + 定时轮询、Sorted Set 实现延迟队列、专业消息队列）

---

## 第四章 专家级实践洞察

### 4.1 常见性能陷阱与规避方案

#### 陷阱 1：BigKey

**定义**：
- 字符串 > 10KB
- 集合元素 > 10000 个
- 集合占用内存 > 10MB

**危害**：
- 读取：单次操作耗时长，阻塞其他请求
- 删除：`DEL` 是同步操作，大键删除会阻塞（Redis 4.0+ 用 `UNLINK` 解决）
- 网络：大值传输占用带宽，增加延迟
- 迁移：集群数据迁移时耗时过长

**检测**：
```bash
redis-cli --bigkeys           # 扫描大键（建议在从库执行）
MEMORY USAGE keyname          # 精确查询某个键的内存占用
redis-cli --memkeys           # 按内存占用排序的键扫描（Redis 7.0+）
```

**规避**：
```bash
# 1. 拆分大键
# 错误：单个哈希存储所有用户属性
HSET user:1 name "张三" age ... city ... (100+ fields)

# 正确：按业务维度拆分
HSET user:1:basic name "张三" age ...
HSET user:1:address city "北京" street ...

# 2. 分批删除
# 错误：
DEL bigset                  # 可能阻塞数百毫秒

# 正确：
UNLINK bigset               # Redis 4.0+：异步删除
# 或分批删除（Redis < 4.0）：
SSCAN bigset 0 COUNT 100 | while read element; do
    SREM bigset $element
done

# 3. 限制集合大小
# 使用 XADD MAXLEN、LTRIM 等命令定期裁剪
XADD mystream MAXLEN ~ 10000 * data "..."
LTRIM mylist 0 9999
```

#### 陷阱 2：HotKey

**定义**：某键的访问量远超其他键（如热门商品、明星微博）。

**危害**：
- 单节点 CPU 被打满（集群中热点所在节点）
- 网络带宽瓶颈
- 客户端连接堆积

**检测**：
```bash
# 方法1：MONITOR（临时，有性能损耗）
redis-cli MONITOR | grep "GET hotkey" | wc -l

# 方法2：客户端统计（推荐）
# 在应用层统计每个键的访问频率

# 方法3：Redis 7.0 的 HOTKEY 统计（需配合 LFU 淘汰策略）
redis-cli --hotkeys           # 扫描热键
# 注意：需要设置 maxmemory-policy 为 allkeys-lfu 或 volatile-lfu 以获得准确数据
```

**规避**：
```bash
# 1. 本地缓存（多级缓存）
# 在应用服务器使用 Caffeine/Guava 等本地缓存
# 热点数据缓存在 JVM 堆内，减少 Redis 访问

# 2. 读写分离
# 大量读热键 → 增加从库副本，客户端随机选择从库读取

# 3. 键分片（Key Sharding）
# hotkey → hotkey:1, hotkey:2, ..., hotkey:N
# 随机后缀：hotkey:random(1,10)
# 读取时也随机选择分片

# 4. 集群多副本
# Redis 7.0+ 支持集群从库故障转移
# 热点数据可通过多副本分散读流量
```

#### 陷阱 3：大量键同时过期

**现象**：CPU 尖峰、延迟毛刺。

**原因**：定期删除循环扫描过期键，高过期比例导致循环无法退出。

**规避**：
```python
# 错误：所有键同时过期
import redis
r = redis.Redis()
for user_id in range(1000000):
    r.setex(f"session:{user_id}", 86400, token)  # 都设 86400 秒

# 正确：过期时间加随机偏移
import random
for user_id in range(1000000):
    ttl = 86400 + random.randint(0, 7200)  # 24h + 0-2h 随机偏移
    r.setex(f"session:{user_id}", ttl, token)
```

#### 陷阱 4：内存碎片

**检测**：
```bash
INFO Memory | grep mem_fragmentation_ratio
# mem_fragmentation_ratio = used_memory_rss / used_memory
# > 1.5：存在碎片
# < 1：发生 swap（危险）
```

**规避**：
```bash
# 1. jemalloc 优化
# Redis 默认使用 jemalloc，碎片比 glibc malloc 少

# 2. 启用碎片自动整理（Redis 4.0+）
activedefrag yes
active-defrag-ignore-bytes 100mb   # 碎片超过 100MB 开始整理
active-defrag-threshold-lower 10   # 碎片率 > 1.1 时开始
active-defrag-threshold-upper 100  # 碎片率 > 2 时全力整理

# 3. 避免碎片产生的设计
# 避免频繁创建/删除大小差异大的键
# 尽量使用 listpack/intset 等紧凑编码
```

#### 陷阱 5：Swap（内存交换）

**危害**：Redis 数据被换出到磁盘，每次访问需要从磁盘读回 → 延迟从微秒级暴增到毫秒级。

**检测**：
```bash
# 方法1：查看碎片率
INFO Memory | grep mem_fragmentation_ratio
# < 1 说明部分内存被 swap 了

# 方法2：系统命令
cat /proc/$(pidof redis-server)/status | grep -E "VmSwap|VmRSS"
```

**规避**：
```bash
# 1. 系统层面禁用 swap
echo "vm.swappiness = 0" >> /etc/sysctl.conf
sysctl -p
# 或
swapoff -a  # 前提：物理内存足够

# 2. Redis 层面设置 maxmemory
maxmemory 70%_of_physical_memory  # 只使用 70% 物理内存
maxmemory-policy allkeys-lru      # 达到上限时淘汰

# 3. 容器环境
# Docker/K8s 设置 memory limit，确保不触发 OOM
```

---

### 4.2 容量规划与内存优化公式

#### 基础公式

**Redis 实际内存消耗**：
```
总内存 = 数据内存 + 内部结构开销 + 复制缓冲区 + 持久化 COW 预留 + 碎片

数据内存：
- 字符串：embstr ~21字节，raw ~34字节（不含实际数据）
- List/Hash/ZSet (listpack): 极紧凑，约 1.1-1.3× 原始数据
- Set (intset): 极紧凑，约 1.1-1.2× 原始数据
- Hash (hashtable): 约 2-3× 原始数据（指针开销大）
- ZSet (skiplist): 约 2-3× 原始数据（dict + zskiplist 双结构）

内部结构开销：
- redisObject: 16 字节
- dictEntry: 24 字节
- 指针: 8 字节（64位系统）

复制缓冲区：
- repl-backlog-size: 生产建议 256MB-1GB
- client-output-buffer-limit slave: 建议 256MB

持久化 COW 预留：
- 写密集型场景：预留 50-100% 数据内存
- 读多写少场景：预留 20-30% 数据内存

碎片：
- jemalloc: 通常 1.0-1.3（可能达到 1.5-2.0）
```

#### 内存优化公式与示例

**场景 1：纯缓存系统（LRU 淘汰）**
```
配置：
maxmemory 16GB                    # 单实例
maxmemory-policy allkeys-lru
hz 10

预估：
有效容量 = 16GB × 0.8（预留开销） = 12.8GB
实际可缓存数据（序列化后）≈ 10-12GB
```

**场景 2：持久化存储系统**
```
配置：
假设 10GB 数据，1主2从，NVMe SSD

主机内存：
  数据：10GB
  COW 预留：10GB × 0.5 = 5GB（中等写入量）
  replication backlog：1GB
  repl buffer：0.5GB × 2 = 1GB
  碎片：约 2GB
  ────────────────────
  总计：约 19GB → 建议 24GB 物理内存

从机内存（每个）：
  数据：10GB
  COW 预留：10GB × 0.1 = 1GB（从机无写，只有 BGSAVE 时才 COW）
  碎片：约 1GB
  ────────────────────
  总计：约 12GB → 建议 16GB 物理内存

总物理内存需求：
  主机 24GB + 从机 16GB × 2 + 哨兵（2GB × 3）= 62GB
```

**场景 3：大键优化**
```
场景：100 万用户，每个用户 100 个属性（平均 20 字节/key, 50 字节/value）

方案 A：单个 Hash（HMSET user:{id} field1 v1 ... field100 v100）
  编码：listpack（100 fields < 512，使用 listpack）
  每用户内存：约 100 × (20+50) × 1.2 = 8.4KB
  总内存：8.4KB × 100万 = 8.4GB

方案 B：拆分为 4 个 Hash
  编码：listpack（每个 25 fields，远小于 512）
  每用户内存：100 × (20+50) × 1.1 = 7.7KB（listpack 更紧凑）
  总内存：7.7GB（节省约 8%）

方案 C：纯 String（SET user:{id}:field1 v1 ...）
  编码：embstr 或 raw
  每用户内存：100 × (34 + 20 + 50) = 10.4KB（34=头部，20=key后缀，50=value）
  总内存：10.4GB（比 Hash 方案多约 24%）
```

#### 容量规划检查清单

```
□ 1. 数据量预估
  □ 键数量、平均大小、数据类型
  □ 读写比例、峰值 QPS
  □ 数据增长速率

□ 2. 内存预算
  □ 数据内存 + 30% 开销
  □ COW 预留（根据写量：20%-100%）
  □ 复制缓冲区（建议 256MB-1GB）
  □ 碎片预留（10%-30%）

□ 3. 持久化开销
  □ RDB/AOF 磁盘空间
  □ AOF 重写临时空间（与 AOF 文件同等大小）
  □ 备份存储空间

□ 4. 网络预算
  □ 带宽 = QPS × 平均命令大小
  □ 复制带宽 = 写入量 × (1 + 从库数量)
  □ 集群 Gossip 开销（节点数 × 心跳包大小 × 频率）

□ 5. 高可用冗余
  □ 从库数量（建议 ≥ 2）
  □ 哨兵数量（≥ 3，奇数）
  □ 跨机架/可用区部署
```

---

### 4.3 生产环境高可用架构决策树

#### 架构选型决策树

```
┌─ 数据量 < 20GB 且 QPS < 5万？
│   ├─ 是 → 单实例是否可接受？
│   │   ├─ 可接受故障停服 → 单实例 + RDB/AOF 备份
│   │   └─ 不能停服 → 主从 + 哨兵（3节点）
│   └─ 否 → 继续判断
│
├─ 数据量 20-200GB 或 QPS 5-50万？
│   ├─ 写多读少 → Redis Cluster
│   │   └─ 1主2从 × N分片 + 3哨兵（哨兵仅用于监控，集群自带故障转移）
│   └─ 读多写少 → 主从 + 哨兵 + 多从库读写分离
│       └─ 1主N从（N≥3） + 3哨兵
│
└─ 数据量 > 200GB 或 QPS > 50万？
    ├─ Redis Cluster（唯一选择）
    │   └─ 1主2从 × M分片（M = 数据量/20GB）
    └─ 考虑混合架构：Redis Cluster + 本地缓存
```

#### 生产环境配置模板

**主从 + 哨兵模式**：
```bash
# Redis 主从配置（所有节点相同部分）
port 6379
daemonize yes
pidfile /var/run/redis_6379.pid
logfile /var/log/redis_6379.log

# 持久化
save 3600 1
save 300 100
save 60 10000
rdbcompression yes
dbfilename dump.rdb
dir /data/redis

# AOF
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-use-rdb-preamble yes

# 内存
maxmemory 16GB
maxmemory-policy allkeys-lru

# 慢查询
slowlog-log-slower-than 10000
slowlog-max-len 128

# 网络
timeout 300
tcp-keepalive 300

# 复制（仅从库配置）
# replicaof <master_ip> 6379
# masterauth <password>

# 安全
requirepass <strong_password>
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command KEYS ""
```

**哨兵配置**（所有哨兵相同）：
```bash
# sentinel.conf
port 26379
daemonize yes
pidfile /var/run/redis-sentinel_26379.pid
logfile /var/log/redis-sentinel_26379.log

# 监控主库
sentinel monitor mymaster <master_ip> 6379 2
sentinel auth-pass mymaster <password>

# 故障转移配置
sentinel down-after-milliseconds mymaster 10000  # 10 秒无响应判定下线
sentinel failover-timeout mymaster 180000        # 3 分钟故障转移超时
sentinel parallel-syncs mymaster 1               # 一次只同步一个从库
```

**Redis Cluster 配置**：
```bash
# 集群专用配置
port 6379
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 15000     # 15 秒节点超时
# cluster-require-full-coverage 默认为 no（推荐）
cluster-require-full-coverage no   # 允许部分槽不可用时继续服务

# 集群总线端口 = port + 10000（自动）
# 需要开放 6379 和 16379 端口

# 迁移配置
cluster-migration-barrier 1    # 从库迁移阈值
```

---

### 4.4 专家级诊断流程

#### 延迟排查五步法

```
步骤 1：确认延迟来源
  redis-cli --latency           # 客户端到服务端延迟
  redis-cli --latency-history   # 延迟历史记录
  redis-cli --latency-dist      # 延迟分布图

步骤 2：检查服务端负载
  INFO CPU                      # CPU 使用率
  SLOWLOG GET 100               # 最近慢查询
  INFO Stats                    # instantaneous_ops_per_sec

步骤 3：检查持久化
  INFO Persistence              # rdb_last_save_time 是否正常
  INFO Stats | grep latest_fork_usec  # fork 耗时

步骤 4：检查内存
  INFO Memory                   # used_memory_rss, mem_fragmentation_ratio
  redis-cli --bigkeys           # 大键扫描

步骤 5：检查复制/集群
  INFO Replication              # master_repl_offset 差距
  CLUSTER INFO                  # 集群状态
```

#### 急救命令速查

```bash
# 客户端相关
CLIENT LIST                    # 查看所有连接
CLIENT KILL ADDR ip:port       # 杀死指定连接
CLIENT PAUSE 10000             # 暂停所有写入（主从切换时使用）

# 内存相关
MEMORY DOCTOR                  # 内存诊断报告
MEMORY MALLOC-STATS            # jemalloc 统计
MEMORY PURGE                   # 请求释放脏页

# 调试相关
DEBUG OBJECT key               # 查看键的内部编码
DEBUG SLEEP 0.1                # 模拟延迟（秒）
MONITOR                        # 实时命令监控（⚠️ 性能杀手）

# 配置相关
CONFIG SET maxmemory 20GB      # 在线调整配置
CONFIG REWRITE                 # 持久化配置到文件
```

> **学习建议**：按章节顺序学习，每节掌握后可对照“进阶思考题”检验理解深度。建议配合 Redis 源码阅读（GitHub: redis/redis），从 `src/server.c` 的 `main()` 函数开始追踪，加深对架构的理解。
---