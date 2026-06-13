
## 一、 核心机制与数据可靠性

### 1.1 消息存储与刷盘策略

RocketMQ 采用 CommitLog + ConsumeQueue + IndexFile 三层存储架构。CommitLog 顺序写保证高吞吐，ConsumeQueue 作为消费索引支持高效拉取。

- **异步刷盘丢消息风险**：CommitLog 异步刷盘（`flushDiskType=ASYNC_FLUSH`）在 Broker 宕机时存在未刷盘消息丢失风险。**【解决方案】**：金融/订单等零丢失场景必须配置 `flushDiskType=SYNC_FLUSH`；允许微量丢失的高吞吐场景保留异步刷盘但需在 SLA 中明确标注“极端宕机可能丢失最近数百毫秒消息”，并配合主从同步降低综合丢失概率。（对应问题 #1）
- **主从同步数据回退**：主从同步模式下 Slave 仅支持读不支持写，故障切换后新 Master 可能因同步延迟导致已确认消息回退。**【解决方案】**：启用 Dledger/Raft 模式（Controller 模式）替代传统主从复制；若使用传统模式，切换后必须执行 `mqadmin getBrokerClusterInfo` 比对新旧 Master 的 `commitLogMaxOffset`，对差异区间消息进行人工补偿或标记为“待确认”。（对应问题 #2）
- **广播消费进度不持久化**：广播消费模式不维护服务端消费进度，重启后默认从最新位点开始消费。**【解决方案】**：业务侧自行实现基于 Redis/DB 的消费位点持久化；或在启动时通过 `consumer.setPullTimeDelayMillsWhenException()` 配合本地缓存位点手动 seek 到指定 offset，文档与监控面板需显著标注“广播模式重启不续消”。（对应问题 #3）
- **顺序消息阻塞粒度**：顺序消息消费失败时本地无限重试会阻塞后续消息，该阻塞是进程级而非队列级。**【解决方案】**：将顺序消息消费逻辑拆分为“轻量校验+异步处理”两阶段，仅在内存队列锁持有期间完成快速校验；或将单 Topic 拆分为多个细粒度顺序 Topic 降低单进程锁竞争；设置最大本地重试次数后转入异常处理流程而非无限阻塞。（对应问题 #4）
- **Broker 注册时序竞争**：Broker 启动后向 NameServer 注册路由与开始接收生产请求之间存在短暂窗口，此时生产者可能获取到旧路由或连接被拒绝。**【解决方案】**：Broker 启动脚本中先完成本地存储恢复与预热，再执行 NameServer 注册；生产者端配置 `sendLatencyFaultEnable=true` 启用故障延迟机制，自动规避刚注册的 Broker；K8s 环境 Readiness Probe 检查 Broker 注册成功标志后再放行流量；监控 Broker 启动后首分钟 `sendFailTps` 确认窗口期影响。（对应问题 #12）

### 1.2 消息大小与格式约束

- **4MB 上限含 Header 开销**：默认 `maxMessageSize=4194304` 包含协议 Header、属性区等开销，实际 Body 可用空间约 4MB - 2KB。**【解决方案】**：业务发送前按 `body.length + properties.getBytes().length + 100 < maxMessageSize` 预校验；大消息改用对象存储 + 引用消息模式；Broker 端调整 `maxMessageSize` 时需同步调整客户端 SDK 对应参数避免客户端提前拒绝。（对应问题 #5）
- **压缩阈值判断错误**：消息体压缩阈值基于原始大小判断，压缩后仍可能超 Broker `maxMessageSize` 限制导致 `MESSAGE_ILLEGAL` 且无法自动降级。**【解决方案】**：发送前按压缩后预估大小（原始大小 × 压缩率 + Header）预校验；大消息先压缩再判断是否超限；Broker 端 `maxMessageSize` 预留 20% 压缩膨胀缓冲；监控 `MESSAGE_ILLEGAL` 错误码中 `compressOverflow` 子类频次。（对应问题 #49）

## 二、 生产发送与事务语义

### 2.1 发送行为与重试

- **消费者组变更不继承堆积**：消费者组名变更被视为全新订阅，历史堆积不会被继承或迁移。**【解决方案】**：禁止通过改名方式重置消费；需重置位点时使用 `mqadmin resetOffsetByTimestamp`；迁移消费者组时先创建新组并手动同步 offset 映射表后再切换流量。（对应问题 #6）
- **send() 重试次数偏差**：`DefaultMQProducer.send()` 内部重试次数计算逻辑与文档描述存在偏差，实际重试次数 = `retryTimesWhenSendFailed + 1`（首次发送计入总尝试次数）。**【解决方案】**：代码注释与内部文档统一修正为“总尝试次数 = retryTimesWhenSendFailed + 1”；超时敏感业务将该值设为 0 并在外层封装自定义重试。（对应问题 #7）
- **批量发送部分失败不可知**：批量发送消息时若部分失败，客户端无法获知具体哪些消息成功/失败。**【解决方案】**：关键业务禁止使用批量发送；非关键日志类批量发送改为“全批重试或全批丢弃”语义；需精确感知单条状态时拆分为单条异步发送 + CompletableFuture 聚合结果。（对应问题 #15）
- **sendMsgTimeout 含排队时间**：生产者 `sendMsgTimeout` 包含 Broker 端排队等待时间，Broker 过载时易产生“假超时”（消息实际已写入但客户端判定失败）。**【解决方案】**：Broker 端启用 `rejectRequestWhenSlow=true` 快速失败而非排队；客户端区分 `RemotingTimeoutException` 与 `MQBrokerException(SYSTEM_BUSY)`，后者说明消息可能已落盘，应查询确认而非直接重试；超时时间设置为 P99 延迟的 2~3 倍而非平均值。（对应问题 #35）
- **固定间隔重试加剧过载**：客户端发送失败重试采用固定间隔退避（默认 100ms × 重试次数），Broker 过载时形成周期性流量脉冲。**【解决方案】**：业务层封装指数退避 + 随机抖动重试逻辑（如 `baseDelay * 2^n + random(0, baseDelay)`）；或使用 Sentinel/Resilience4j 等框架接管重试策略；Broker 端配合 `brokerRebalanceWaitTimeMillis` 与流控规则平滑流量。（对应问题 #64）

### 2.2 事务消息边界

- **本地事务超时状态不一致**：生产者本地事务执行超时但 Broker 半消息已提交，导致状态不一致。**【解决方案】**：本地事务执行时间必须远小于 `transactionCheckInterval`（默认 60s）；设置合理的 `transactionTimeout` 使 Broker 端超时早于本地超时；回查接口实现幂等且能快速返回 UNKNOW 触发下次回查而非阻塞。（对应问题 #26）
- **回查接口缺幂等保护**：事务消息回查接口被并发调用时缺乏幂等保护。**【解决方案】**：回查接口以 `msgId + transactionId` 为键加分布式锁或本地 ConcurrentHashMap 防重入；数据库查询加唯一索引兜底；回查结果缓存 TTL ≥ `transactionCheckInterval`。（对应问题 #29）
- **回查挤占消费线程池**：事务消息回查任务与正常消费线程共享同一线程池，高频回查导致消费延迟飙升。**【解决方案】**：升级至 4.9.4+ / 5.x 版本，回查使用独立线程池 `transactionCheckThreadPoolNums`；低版本通过减少半消息发送速率、优化本地事务执行时间间接降低回查频率；监控 `checkTransactionMessageTps` 指标设置告警阈值。（对应问题 #45）
- **批量事务语义互斥**：批量发送的消息若作为事务消息使用，整批共享同一事务 ID，无法实现批次内部分消息的事务语义。**【解决方案】**：事务消息严格禁止使用 `send(Collection<Message>)` 批量接口；代码层面增加拦截器/注解校验，检测到事务消息 + 批量组合时直接抛异常；已误用的历史批次需拆分为单条重新投递。（对应问题 #62）

## 三、 消费模型与负载均衡

### 3.1 消费模式与进度管理

- **过滤表达式语法静默失败**：消息过滤表达式语法错误不会在订阅时报错，而是在运行时静默丢弃所有消息。**【解决方案】**：订阅后立即发送测试消息验证消费是否正常；使用 `mqadmin consumerProgress` 对比过滤前后 TPS 是否合理；CI/CD 流水线集成 SQL92/Tag 表达式语法校验工具；开启 `filterServerEnable=true` 时额外验证 Filter Server 编译日志。（对应问题 #18）
- **订阅关系内存态重启丢失**：消费者组订阅关系仅在内存中维护，Broker 重启后依赖客户端首次心跳重新上报，重启窗口内新到达的消息因无订阅关系被跳过或投递至错误位点。**【解决方案】**：Broker 滚动重启而非全量重启；重启前执行 `mqadmin updateSubGroup` 强制持久化订阅关系；升级至 5.x 版本（订阅关系已持久化）；监控 Broker 重启后首分钟消费 TPS 骤降告警。（对应问题 #48）
- **pullBatchSize 字节溢出断连**：消费者拉取消息时 `pullBatchSize` 仅控制条数不控制总字节数，大消息场景下单次响应体可达 128MB 超出 Netty 帧长度限制导致解码异常断连。**【解决方案】**：大消息（>512KB）场景将 `pullBatchSize` 调小至 4~8；升级 SDK 至 4.9.4+ 启用动态批次大小适配；Broker 端设置 `maxTransferBytesOnMessageInDisk` 限制单次传输字节上限；监控消费者端 `decodeException` 日志频次。（对应问题 #65）

### 3.2 Rebalance 与分配算法

- **Rebalance 重复消费窗口**：消费者 Rebalance 期间可能出现短暂的重复消费窗口。**【解决方案】**：消费端必须实现幂等（数据库唯一键/Redis SETNX/业务状态机）；使用 `MessageExt.getReconsumeTimes()` 识别重试消息；Rebalance 期间暂停拉取而非立即释放队列（5.x Pop 模式天然解决此问题）。（对应问题 #11）
- **AVG 分配永久倾斜**：AVG 分配算法在队列数不能被消费者数整除时产生永久性负载倾斜。**【解决方案】**：部署时确保队列数为消费者数的整数倍；或使用一致性哈希分配策略（`AllocateStrategy.CONSISTENT_HASH`）；5.x 版本使用 Pop 消费模式由 Broker 端动态分配消除客户端分配不均。（对应问题 #40）
- **全局顺序 Rebalance 有序性失效**：全局顺序消息在 Broker 扩缩容 rebalance 窗口内有序性暂时失效。**【解决方案】**：扩缩容操作安排在业务低峰期；扩缩容前后暂停生产端发送等待消费端追平；使用分区顺序消息替代全局顺序消息缩小影响范围；5.x 版本使用 FIFO 消息类型 + Pop 消费保障扩缩容期间有序。（对应问题 #32）

### 3.3 Pop 消费与死信队列

- **Pop invisibleTime 无限重投**：Pop 消费模式 `invisibleTime` 小于实际处理耗时时引发无限重投循环。**【解决方案】**：`invisibleTime` 设置为 P99 处理耗时的 1.5~2 倍；消费端记录实际处理耗时并动态调整下次 invisibleTime；监控 `popReviveTps` 指标，突增时自动告警；设置最大重投次数后转入死信队列而非无限循环。（对应问题 #33）
- **异常不计入死信重试计数**：消费者抛出异常不计入死信重试计数，异常消息可能无限重试而不进死信队列。**【解决方案】**：消费端捕获异常后显式返回 `RECONSUME_LATER` 而非抛异常；或在 `MessageListenerConcurrently.consumeMessage` 中 try-catch 包裹全部逻辑，catch 块中根据 `reconsumeTimes` 判断是否达到阈值后返回 SUCCESS 并手动转存死信表；升级至 5.x 版本（异常已正确计入重试计数）。（对应问题 #54）

## 四、 存储引擎与底层资源

### 4.1 MappedFile 与内存管理

- **mlock 预热容器静默失效**：MappedFile 预热依赖 `mlock` 系统调用，在非 root 容器环境中静默失效。**【解决方案】**：容器启动时添加 `--cap-add=IPC_LOCK` 或在 K8s SecurityContext 中设置 `capabilities.add: ["IPC_LOCK"]`；启动日志检查 `mlock failed` 关键字；无法提权时关闭预热（`dataWarm=false`）并接受首批写入延迟毛刺。（对应问题 #9）
- **VmSize 持续增长 OOM**：Broker 删除过期 CommitLog 文件时仅解除内存映射但不释放虚拟地址空间，长期运行 VmSize 持续增长触发 OOM。**【解决方案】**：监控 `/proc/<pid>/status` 中 VmSize 指标（而非仅堆/堆外内存）；32 位 JVM 禁止用于生产；64 位环境设置 `-XX:MaxDirectMemorySize` 留足余量；定期滚动重启 Broker 释放虚拟地址空间；升级至 5.x 版本（已优化 MappedFile 生命周期管理）。（对应问题 #56）
- **堆外内存外部碎片化**：Broker 堆外内存在 MappedFile 频繁创建销毁中产生外部碎片，JMX 指标显示剩余充足但连续大块分配失败。**【解决方案】**：启用 `dataWarm=true` 减少文件切换频率；使用 jemalloc/tcmalloc 替代默认 malloc（`LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so`）；监控 `OutOfDirectMemoryError` 日志而非仅 JMX 指标；设置 `-XX:MaxDirectMemorySize` 为物理内存的 60%~70% 预留碎片缓冲。（对应问题 #59）
- **ConsumeQueue ByteBuffer 泄漏**：Broker 端 ConsumeQueue 构建临时 ByteBuffer 在高并发下未及时释放，堆外内存缓慢增长直至 OOM Killer 触发重启。**【解决方案】**：升级至 4.9.4+ / 5.x 版本（已修复 ByteBuffer 复用逻辑）；低版本降低 `flushConsumeQueueInterval` 减少并发构建压力；监控堆外内存增长斜率，日均增长 >100MB 时安排重启窗口。（对应问题 #50）
- **磁盘满保护不完整**：Broker 磁盘满保护仅检测 CommitLog 所在磁盘，ConsumeQueue/IndexFile 独立磁盘先满时不触发只读，导致索引损坏。**【解决方案】**：CommitLog、ConsumeQueue、IndexFile 部署在同一磁盘；或分别配置独立磁盘满检测脚本 + 自定义告警；升级至 5.x 版本（已支持多磁盘独立检测）；运维巡检加入各挂载点使用率差异化告警。（对应问题 #52）
- **CQ/CL 删除非原子索引悬空**：CommitLog 文件删除后 ConsumeQueue 仍保留指向已删除文件的 offset，消费者拉取时返回空结果或异常，触发无效重试。**【解决方案】**：升级至 4.9.4+ / 5.x 版本（删除流程已增加引用计数保护）；低版本设置 `fileReservedTime=72` 延长 CommitLog 保留时间，确保 ConsumeQueue 消费位点追过待删文件后再清理；监控消费者端 `pullMessageFromSlaveException` 与 Broker 端 `dispatchBehind` 指标，突增时暂停 CommitLog 删除任务；运维脚本删除 CommitLog 前先校验对应 ConsumeQueue 最大消费位点是否已超过待删文件最大 offset。（对应问题 #36）

### 4.2 索引与过滤性能

- **IndexFile 哈希槽上限**：IndexFile 哈希槽数量固定为 500万，超大规模 Topic 下查询性能急剧下降。**【解决方案】**：单 Topic 消息量预估超 500万/天时拆分 Topic；按 Key 查询改用外部 ES/ClickHouse 索引；Tag 过滤优先于 Key 查询（Tag 走 ConsumeQueue 不受 IndexFile 限制）；监控 `getIndexTps` 与查询延迟 P99。（对应问题 #19）
- **Tag/SQL92 互斥**：Tag 过滤与 SQL92 过滤不能同时生效，组合使用时仅 SQL92 有效。**【解决方案】**：需要组合过滤时统一使用 SQL92 表达式（`TAGS='xxx' AND age>18`）；纯 Tag 过滤场景避免混用 SQL92 语法以获得更好的 ConsumeQueue 位图过滤性能；文档与代码注释明确标注互斥规则。（对应问题 #28）
- **Tag 哈希假命中**：Tag 哈希过滤在低基数场景下假命中率高，浪费带宽与 CPU。**【解决方案】**：低基数 Tag（<100种）场景改用 SQL92 过滤；或增加 Tag 组合维度提高基数（如 `order_create_success` 替代 `success`）；监控 Broker 端 `dispatchBehind` 与消费者端 `filterMissRate` 评估假命中影响。（对应问题 #42）

### 4.3 时钟与延迟消息

- **时钟回拨数据错乱**：Broker 依赖系统时钟生成时间戳与延迟投递判定，NTP step 调整或人工修改导致时钟回拨时 ConsumeQueue 二分查找返回错误位点、延迟消息被立即投递。**【解决方案】**：服务器时间同步必须使用 chrony 平滑同步（`makestep` 仅在启动时允许，运行时禁用 step）；禁止人工 `date -s` 修改时间；升级至 5.x 版本使用逻辑时钟替代物理时钟；监控 `system_clock_backward` 自定义指标并设置最高级别告警。（对应问题 #63）
- **延迟消息主备切换精度漂移**：延迟消息在主备切换时定时轮询重置，投递精度漂移 ±1s。**【解决方案】**：延迟消息精度要求 <1s 的场景使用外部调度系统 + 即时消息替代；接受 ±1s 漂移的业务在 SLA 中标注；5.x 版本使用 TimerWheel 延迟消息引擎，主备切换精度漂移降至毫秒级。（对应问题 #38）

## 五、 网络协议与客户端 SDK

### 5.1 协议兼容与连接管理

- **Remoting Header 跨语言解析不一致**：Remoting 协议 Header 长度字段在不同语言 SDK 中解析方式不一致。**【解决方案】**：多语言混合部署时统一使用 gRPC 代理模式（5.x Proxy）；Remoting 模式下所有 SDK 升级至相同主版本；跨语言通信增加协议兼容性集成测试。（对应问题 #25）
- **gRPC Pop ACK 语义差异**：gRPC 代理模式下 Pop 消费的 ACK 语义与原生协议存在细微差异。**【解决方案】**：从 Remoting 迁移至 gRPC 代理时进行 ACK 行为回归测试；ACK 失败重试逻辑在两种模式下保持一致；阅读 Proxy 模块源码确认 ACK 超时与重试策略差异点。（对应问题 #31）
- **Netty 连接池失效连接残留**：客户端 Netty 连接池在 Broker IP 变更后未清理失效连接，恢复后仍有数十秒发送失败窗口。**【解决方案】**：升级 SDK 至 4.9.4+（已优化连接健康检测）；低版本在 Broker 重启/IP 变更后手动调用 `producer.shutdown()` + `producer.start()` 重建连接池；配置 `clientChannelMaxIdleTimeSeconds=60` 缩短空闲连接回收周期；监控发送端 `connectTimeout` 异常频次。（对应问题 #55）
- **DNS 永久缓存**：Java 客户端 DNS 解析结果被 JVM 永久缓存（`networkaddress.cache.ttl=-1`），NameServer/Broker IP 变更后永不刷新。**【解决方案】**：JVM 启动参数设置 `-Dsun.net.inetaddr.ttl=30 -Dsun.net.inetaddr.negative.ttl=10`；K8s 环境使用 Service ClusterIP 而非 Headless Service + DNS；IP 变更时滚动重启客户端进程；5.x gRPC 模式支持服务端推送地址变更。（对应问题 #51）
- **长轮询超时中间件断连**：消费者长轮询挂起超时由 Broker 端 `longPollingEnable=true` 时的 `suspendMaxTimeMills`（默认 15s）控制，不受客户端 `pullRequestTimeout` 影响；当中间网关/LB 空闲超时 <15s 时，长轮询连接被中间件主动断开，导致消费者频繁重连、吞吐量骤降。**【解决方案】**：所有中间件（Nginx/SLB/API Gateway）空闲超时设置为 ≥30s；Broker 端 `suspendMaxTimeMills` 调整为 ≤ 中间件空闲超时的 80%；监控消费者端 `pullConnectionClosedTps` 指标，突增时排查中间件超时配置；5.x gRPC 模式使用服务端流式推送替代长轮询，彻底规避此问题。（对应问题 #58）

### 5.2 可观测性与指标上报

- **消息轨迹静默丢弃**：消息轨迹异步发送失败时静默丢弃，与主消息状态不一致。**【解决方案】**：关键业务轨迹改用同步发送或写入本地文件 + Filebeat 采集；监控 `traceSendFailTps` 指标；轨迹 Topic 单独部署 Broker 避免与业务 Topic 资源竞争；接受轨迹少量丢失的场景在 SLA 中标注。（对应问题 #41）
- **Metrics 上报阻塞业务线程**：客户端 Metrics 上报线程与业务线程共享 EventLoop，采集端响应慢时阻塞正常消息收发。**【解决方案】**：生产环境禁用客户端内置 Metrics（`enableMetric=false`）；改用 Broker 端 Prometheus Exporter 采集服务端指标；确需客户端指标时使用独立线程池上报；监控客户端 EventLoop 任务队列长度。（对应问题 #60）
- **冷启动预热延迟毛刺**：Broker 冷启动预热未完成即接收流量，首批消息写入延迟毛刺。**【解决方案】**：启动后执行 `warmUpMappedFile` 命令或等待 `dataWarm=true` 预热完成后再注册 NameServer；K8s Readiness Probe 检查预热完成标志文件；流量灰度引入，启动后前 30s 仅导入 10% 流量。（对应问题 #43）

### 5.3 消息属性与序列化

- **UserProperty 中转丢失**：消息自定义属性在延迟/事务中转后丢失，关键标识不应依赖 UserProperty。**【解决方案】**：关键业务标识放入 Message Keys 或 Body JSON 字段；UserProperty 仅用于非关键元数据（如链路追踪 spanId）；延迟/事务消息中转后需校验关键属性完整性；5.x 版本已修复部分属性透传问题但仍建议遵循此规范。（对应问题 #53）
- **MsgId 非稳定去重失效**：消息 ID 不包含业务语义且非全局单调递增，同一消息经延迟/事务中转后 MsgId 重新生成，基于 MsgId 的去重逻辑失效。**【解决方案】**：业务去重严格使用 Message Keys 或 Body 内业务唯一键；MsgId 仅用于日志关联与运维排查；文档与代码注释明确标注‘MsgId 不可作为幂等键’；监控因 MsgId 变更导致的重复消费告警。（对应问题 #57）

## 六、 运维部署与安全治理

### 6.1 部署配置陷阱

- **存储路径冲突**：`storePathRootDir` 与 `storePathCommitLog` 配置相同时删除策略冲突。**【解决方案】**：两者必须配置不同路径；`storePathCommitLog` 指向专用 SSD/NVMe 磁盘；`storePathRootDir` 指向普通磁盘存放 ConsumeQueue/IndexFile/Config；部署脚本增加路径重复校验。（对应问题 #13）
- **GC 日志未默认开启**：JVM GC 日志未默认开启，OOM/GC 停顿排查困难。**【解决方案】**：启动脚本强制添加 `-Xloggc:/data/logs/gc.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M`（JDK8）或等效 JDK11+ 参数；CI/CD 模板内置 GC 日志配置。（对应问题 #14）
- **vm.max_map_count 不足**：Docker 容器中 `vm.max_map_count` 不足导致 MappedFile 创建失败。**【解决方案】**：宿主机执行 `sysctl -w vm.max_map_count=262144` 并写入 `/etc/sysctl.conf`；K8s DaemonSet 自动设置；容器启动脚本检查当前值并告警；文档明确标注此为前置必要条件。（对应问题 #16）
- **多网卡 IP 选择错误**：多网卡环境下 Broker 自动选择的 IP 可能为内网不可达地址。**【解决方案】**：显式配置 `brokerIP1=<业务可达IP>`；K8s 环境使用 Pod IP 并通过 StatefulSet 固定；启动后执行 `mqadmin clusterList` 验证注册 IP 可达性；多网卡机器部署脚本增加 IP 白名单校验。（对应问题 #17）
- **优雅停机未等待刷盘**：优雅停机脚本未等待后台刷盘线程完成即退出。**【解决方案】**：停机脚本先发送 SIGTERM 触发 Broker 内部 shutdown hook，再 `sleep 30` 等待刷盘完成，最后 SIGKILL；或直接使用 `mqshutdown broker` 命令（内置等待逻辑）；K8s preStop hook 设置 `terminationGracePeriodSeconds=60`。（对应问题 #23）
- **多磁盘监控指标歧义**：监控指标 `commitLogDiskRatio` 在多磁盘配置下取值逻辑不明确。**【解决方案】**：多磁盘场景改用 `diskPartitionUsage` 按挂载点分别监控；Prometheus 采集脚本遍历所有 store 路径独立暴露指标；告警规则按磁盘分区配置而非全局阈值。（对应问题 #24）
- **K8s 停机钩子断层**：K8s Pod 优雅停机钩子与 MQ 客户端 shutdown 信号传递断层。**【解决方案】**：preStop hook 中主动调用应用 HTTP 下线接口触发 MQ 客户端 shutdown；Spring Boot 应用注册 `SmartLifecycle` 确保 MQ 客户端在容器销毁前完成关闭；设置 preStop sleep 5s 给客户端足够注销时间。（对应问题 #37）
- **客户端日志无上限**：客户端 SDK 默认日志路径位于用户 Home 目录且无大小滚动上限，长期运行可导致磁盘耗尽引发业务进程崩溃。**【解决方案】**：显式配置 `-Drocketmq.client.logRoot=/data/logs/mq -Drocketmq.client.logLevel=WARN -Drocketmq.client.logFileMaxIndex=20`；日志收集 Agent 监控客户端日志目录大小；CI/CD 模板内置日志滚动配置。（对应问题 #46）

### 6.2 安全与权限

- **ACL 热加载延迟**：ACL 配置文件热加载存在秒级生效延迟。**【解决方案】**：权限变更后等待 5s 再验证；关键权限变更通过 `mqadmin updateAcl` 命令实时生效；监控 ACL 加载日志确认生效时间点；生产环境 ACL 变更纳入变更管理窗口。（对应问题 #27）
- **ACL 缓存无主动失效**：ACL 鉴权结果缓存无主动失效机制，权限变更后需等待 TTL 过期或重启 Broker 才生效，期间存在越权访问窗口。**【解决方案】**：权限收紧操作后立即滚动重启 Broker；或升级至 5.x 版本（支持 ACL 缓存主动失效）；权限放宽操作可接受 TTL 延迟；审计日志覆盖权限变更窗口期内的敏感操作。（对应问题 #44）
- **Admin 权限粒度粗**：Admin 账号权限粒度粗，无法限制特定 Topic 操作。**【解决方案】**：日常运维使用最小权限子账号；Admin 账号仅限紧急故障处理；ACL 配置中为每个业务团队创建独立账号并限定 Topic 前缀；5.x 版本支持更细粒度 RBAC。（对应问题 #30）

### 6.3 集群架构与一致性

- **NameServer 路由不同步**：NameServer 间无路由同步，跨机房部署时最终一致性窗口被放大。**【解决方案】**：跨机房部署时每个机房独立部署 NameServer 集群 + Broker 集群，通过专线同步路由；或使用 5.x Controller 模式统一路由管理；监控各 NameServer 路由表 diff 告警；客户端配置多机房 NameServer 地址列表。（对应问题 #39）
- **NameServer 静默脑裂**：NameServer 集群节点间无选举与数据同步机制，网络分区时各节点独立提供不一致路由，形成静默脑裂且无任何告警。**【解决方案】**：部署奇数个 NameServer 节点；监控各节点路由表版本号差异，差异持续 >30s 触发最高级别告警；客户端配置 `namesrvAddr` 包含所有节点并启用故障隔离；5.x Controller 模式引入 Raft 选举彻底解决脑裂。（对应问题 #47）
- **Controller DLedger 双写丢消息**：Controller 模式 DLedger 与 CommitLog 双写在 IO 抖动时可能静默丢消息。**【解决方案】**：DLedger 与 CommitLog 部署在独立磁盘避免 IO 竞争；启用 `dledgerFlushDiskType=SYNC_FLUSH`；监控 `dledgerCommitLogDiff` 指标，差异持续 >1s 告警；升级至最新 5.x 补丁版本修复已知双写 bug。（对应问题 #34）
- **消费者组元数据不清理**：消费者组元数据在客户端停止订阅后不会自动清理，废弃组配置永久驻留导致 Broker 启动耗时与内存占用异常升高。**【解决方案】**：定期执行 `mqadmin deleteSubGroup -g <groupName> -b <brokerAddr>` 清理废弃组；编写自动化脚本扫描 30 天无消费记录的组并通知负责人确认后删除；监控 Broker 配置文件中 SubscriptionGroupConfig 条目数，超 1000 条告警；5.x 版本支持组元数据 TTL 自动过期。（对应问题 #61）

### 6.4 版本升级与兼容

- **跨版本 CommitLog 不兼容**：跨版本升级时 CommitLog 文件格式变更可能导致旧文件无法读取。**【解决方案】**：升级前查阅 Release Notes 确认存储格式兼容性；大版本升级采用“新建集群 + 双写迁移”而非原地升级；升级后保留旧 CommitLog 文件至少 7 天作为回滚备份；测试环境先行验证升级路径。（对应问题 #22）

## 七、 背压与流控

- **Broker 无背压直接拒绝**：Broker 端无背压机制，高负载下直接拒绝请求而非排队等待。**【解决方案】**：客户端实现自适应限流（根据 `SYSTEM_BUSY` 响应动态降低发送速率）；Broker 端配置 `rejectRequestWhenSlow=true` + `osPageCacheBusyTimeOutMills=1000` 提前拒绝避免雪崩；前置网关/Sentinel 做入口流控；监控 `rejectTps` 指标联动弹性扩容。（对应问题 #10）

## 八、 源码与实现细节补充

- **ConsumeQueue offset 校验边界**：ConsumeQueue 构建过程中的 offset 校验逻辑未在文档中体现边界条件处理。**【解决方案】**：源码阅读时关注 `ConsumeQueue.recoverNormally()` 中 offset 对齐与截断逻辑；运维排查消费位点异常时检查 `consumequeue` 文件大小是否为 `CQ_STORE_UNIT_SIZE * entriesPerFile` 整数倍；非整数倍文件视为损坏需重建。（对应问题 #8）

---

## 附录 A：问题速查索引

|编号|问题摘要|所属章节|
|:--|:--|:--|
|1|异步刷盘丢消息|1.1|
|2|主从同步数据回退|1.1|
|3|广播消费进度不持久化|1.1|
|4|顺序消息进程级阻塞|1.1|
|5|4MB 上限含 Header|1.2|
|6|消费者组变更不继承堆积|2.1|
|7|send() 重试次数偏差|2.1|
|8|ConsumeQueue offset 校验边界|八|
|9|mlock 预热容器失效|4.1|
|10|Broker 无背压直接拒绝|七|
|11|Rebalance 重复消费窗口|3.2|
|12|Broker 注册时序竞争|1.1|
|13|存储路径冲突|6.1|
|14|GC 日志未开启|6.1|
|15|批量发送部分失败不可知|2.1|
|16|vm.max_map_count 不足|6.1|
|17|多网卡 IP 选择错误|6.1|
|18|过滤表达式静默失败|3.1|
|19|IndexFile 哈希槽上限|4.2|
|22|跨版本 CommitLog 不兼容|6.4|
|23|优雅停机未等待刷盘|6.1|
|24|多磁盘监控指标歧义|6.1|
|25|Remoting Header 跨语言不一致|5.1|
|26|本地事务超时状态不一致|2.2|
|27|ACL 热加载延迟|6.2|
|28|Tag/SQL92 互斥|4.2|
|29|回查接口缺幂等|2.2|
|30|Admin 权限粒度粗|6.2|
|31|gRPC Pop ACK 语义差异|5.1|
|32|全局顺序 Rebalance 失效|3.2|
|33|Pop invisibleTime 无限重投|3.3|
|34|Controller 双写丢消息|6.3|
|35|sendMsgTimeout 含排队|2.1|
|36|CQ/CL 删除非原子索引悬空|4.1|
|37|K8s 停机钩子断层|6.1|
|38|延迟消息主备切换漂移|4.3|
|39|NameServer 路由不同步|6.3|
|40|AVG 分配永久倾斜|3.2|
|41|消息轨迹静默丢弃|5.2|
|42|Tag 哈希假命中|4.2|
|43|冷启动预热延迟毛刺|5.2|
|44|ACL 缓存无主动失效|6.2|
|45|回查挤占消费线程池|2.2|
|46|客户端日志无上限|6.1|
|47|NameServer 静默脑裂|6.3|
|48|订阅关系内存态丢失|3.1|
|49|压缩阈值判断错误|1.2|
|50|CQ ByteBuffer 泄漏|4.1|
|51|DNS 永久缓存|5.1|
|52|磁盘满保护不完整|4.1|
|53|UserProperty 中转丢失|5.3|
|54|异常不计入死信计数|3.3|
|55|Netty 连接池失效残留|5.1|
|56|VmSize 持续增长 OOM|4.1|
|57|MsgId 非稳定去重失效|5.3|
|58|长轮询超时中间件断连|5.1|
|59|堆外内存外部碎片化|4.1|
|60|Metrics 上报阻塞业务|5.2|
|61|消费者组元数据不清理|6.3|
|62|批量事务语义互斥|2.2|
|63|时钟回拨数据错乱|4.3|
|64|固定间隔重试加剧过载|2.1|
|65|pullBatchSize 字节溢出|3.1|
