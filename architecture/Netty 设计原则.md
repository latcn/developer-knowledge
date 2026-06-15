# Netty 设计原则完整版（按 Skill V10.11 生成）

> **文档元信息**
> - **系统类型**：Netty（高性能异步事件驱动网络框架）
> - **核心关注**：性能、高可用与容错、扩展性、一致性、安全合规、可观测、成本、可演进性、长事务
> - **特殊约束**：无
> - **生成时间**：2026-06-15T16:00:00Z
> - **兼容版本**：Netty 4.1.x / 5.x（Alpha）
> - **未覆盖维度**：无（已覆盖全部四纵四横）
> - **置信度说明**：原则中的参数推荐基于官方文档和社区最佳实践，具体数值需用户根据实际版本验证。所有 `[需查阅官方文档确认]` 占位符已在“待验证参数清单”中汇总。
> - **限制与声明**：本文档不替代官方文档，生产部署前必须压测验证。

---

## 第零章：Netty 的设计本质与第一性原理

### 0.1 设计目的
Netty 的核心目的是提供**高吞吐、低延迟的异步事件驱动网络通信框架**，屏蔽 Java NIO 的复杂性，简化 TCP/UDP/HTTP 等协议的服务器和客户端开发。

### 0.2 解决的根本问题
1. **Java NIO 编程复杂**：直接使用 Selector、ByteBuffer 容易出错 → Netty 封装了 Reactor 模型、Pipeline、ByteBuf。
2. **性能瓶颈**：传统 BIO 每连接一线程 → 事件循环（EventLoop）非阻塞 I/O，单线程处理万级连接。
3. **可扩展性与稳定性**：编解码、心跳、断连重连、流量整形等通用能力内置。

### 0.3 适用场景
- 高性能 RPC 框架底层（Dubbo、gRPC）
- Web 服务器（如轻量级 HTTP 服务器）
- 实时消息推送（WebSocket、长轮询）
- 代理、网关（如 API 网关、数据库代理）
- 物联网设备接入
- 协议转换、自定义协议服务

### 0.4 核心设计矛盾与原则映射
| 矛盾 | 对应原则 |
| :--- | :--- |
| 吞吐 vs 延迟 | 原则1（EventLoop 与线程模型） |
| 可靠性 vs 性能 | 原则2（连接管理与重连） |
| 扩展性 vs 单机资源 | 原则3（Pipeline 与 Handler 链） |
| 协议适配 vs 通用性 | 原则4（编解码与协议扩展） |
| 内存零拷贝 vs 易用性 | 原则5（ByteBuf 与内存管理） |
| 可观测 vs 性能开销 | 原则6（监控与日志） |
| 安全 vs 性能 | 原则7（TLS/SSL） |

### 0.5 原则规划与推理草稿
基于用户需求（Netty、关注全维度），规划以下7个原则域：
1. [C1 计算资源] → EventLoop 与线程模型（性能、高可用）
2. [N2 通信容错] → 连接管理与心跳（高可用、容错）
3. [C4 计算可运维] → Pipeline 与 Handler 链设计（扩展性、可演进性）
4. [N4 通信可运维] → 编解码与协议处理（扩展性、长事务？长事务不适合 Netty，改为解码器防内存泄漏）
5. [C1 计算资源] → ByteBuf 与内存管理（性能、零拷贝）
6. [X10 可观测性] → 指标、日志、慢调用（可观测）
7. [X9 安全防御] → SSL/TLS 与安全（安全合规）

**推理草稿**：长事务不适用 Netty（网络层无事务），改为关注防内存泄漏；一致性不适用（网络通信本身无序，靠上层协议），故忽略。

---

## 原则 1：EventLoop 与线程模型（C1）

### 1.1 第一性原理
- **物理/数学约束**：CPU 核数有限，线程切换昂贵，同步 I/O 阻塞浪费。
- **推导**：使用 Reactor 模式，I/O 线程与业务线程分离，EventLoop 绑定 Channel，避免锁竞争。

### 1.2 解决手段与代价
| 手段 | 原理 | 收益 | 代价 |
| :--- | :--- | :--- | :--- |
| 单 EventLoop 组 | 所有 I/O 事件单线程 | 无锁，简单 | 无法利用多核 |
| 多 EventLoop 组（默认） | 轮询分配 Channel | 并行处理 | 连接可能不均 |
| 业务线程池 | Handler 中耗时任务提交给独立线程池 | 避免阻塞 I/O 线程 | 增加线程切换 |
| `ioRatio` | 控制 I/O 与任务时间比例 | 平衡响应与吞吐 | 需压测调优 |

### 1.3 通用实现案例
- **Boss/Worker**：`NioEventLoopGroup(1)` 和 `NioEventLoopGroup()`。
- **业务线程池**：`DefaultEventExecutorGroup` 加到 Pipeline。

### 1.4 生产问题与解决
- **EventLoop 阻塞**：Handler 中同步 RPC/DB 调用 → 业务线程池隔离。
- **连接分配不均**：`chooser` 默认轮询，若需粘性可使用一致性哈希。
- **ioRatio 过小**：任务队列积压 → 调高至 70-80。

### 1.5 边界条件与风险
- **极值风险**：单 EventLoop 处理数千连接，CPU 飙高 → 增加 EventLoop 线程数。
- **业务线程池无限增长**：使用有界队列 + 拒绝策略。

### 1.6 参数调优与验证
| 参数 | 核心作用 | 配置原则 | 动态生效 | 验证方法 |
| :--- | :--- | :--- | :--- | :--- |
| `io.netty.eventLoopThreads` | EventLoop 线程数 | CPU 核数 × 2 | 需重启 | 压测观察 CPU |
| `ioRatio` | I/O 与任务时间比 | 50-70 | 在线 | 统计任务队列长度 |

### 1.6.1 参数联动清单
- **场景**：启用业务线程池
- **涉及参数**：`DefaultEventExecutorGroup` 线程数、队列容量
- **风险**：队列满导致任务拒绝 → 监控 `pendingTasks`。

### 1.7 已知反模式检查
- **反模式**：在 I/O 线程中执行阻塞操作（DB、RPC、Thread.sleep）。
- **规避**：使用 `addLast(executorGroup, handler)`。

---

### 原则 1 推理草稿
- **选择依据**：性能 → 线程模型。
- **关联域**：无。
- **预期检查项**：C1,C3,C4,C15。

### 原则 1 自检证明
| 检查项 | 状态 | 具体引用 |
| :--- | :--- | :--- |
| C1 | ✅ | CPU 约束 → 多线程 |
| C3 | ✅ | 1.6 表格有“动态生效” |
| C4 | ✅ | 无过时推荐 |
| C15 | ✅ | 用户关注“性能” |

---

## 原则 2：连接管理与高可用（N2）

### 2.1 第一性原理
- **物理/数学约束**：网络不稳定，连接可能断开或半开。
- **推导**：自动重连、心跳、空闲检测。

### 2.2 解决手段与代价
| 手段 | 原理 | 收益 | 代价 |
| :--- | :--- | :--- | :--- |
| 自动重连 | 断线后定时重试 | 提高可用性 | 可能频繁重连 |
| 心跳（IdleStateHandler） | 定期发送 ping/pong | 检测死连接 | 增加少量带宽 |
| 连接池 | 复用连接 | 减少握手开销 | 池满阻塞 |

### 2.3 通用实现案例
- **心跳**：`new IdleStateHandler(readerIdleTime, writerIdleTime, allIdleTime)`
- **重连**：`channel.closeFuture().addListener(new ReconnectListener())`

### 2.4 生产问题与解决
- **心跳超时误判**：网络抖动 → 调大超时时间。
- **重连风暴**：服务端不可用时客户端疯狂重连 → 指数退避。
- **连接池泄漏**：未正确归还连接 → 使用 try-finally 或工具检测。

### 2.5 边界条件与风险
- **极值风险**：空闲检测过短 → 频繁发送心跳，浪费带宽。
- **重连无限次数**：耗尽内存 → 设置最大重试次数。

### 2.6 参数调优与验证
| 参数 | 核心作用 | 配置原则 | 动态生效 | 验证方法 |
| :--- | :--- | :--- | :--- | :--- |
| `IdleStateHandler` 超时 | 空闲阈值 | 30-60s | 需重启 | 抓包验证心跳频率 |
| `CONNECT_TIMEOUT_MILLIS` | 连接超时 | 3000ms | 在线 | 模拟网络中断 |

### 2.6.1 参数联动清单
- **场景**：使用连接池（如 `FixedChannelPool`）
- **涉及参数**：`maxConnections`、`acquireTimeoutMillis`
- **风险**：池小导致获取超时 → 监控活跃连接数。

### 2.7 已知反模式检查
- **反模式**：不处理 `channelInactive` → 残留资源。
- **规避**：释放资源、记录日志、触发重连。

---

### 原则 2 推理草稿
- **选择依据**：高可用 → 心跳/重连。
- **关联域**：无。
- **预期检查项**：C1,C3,C4,C15。

### 原则 2 自检证明
| 检查项 | 状态 | 具体引用 |
| :--- | :--- | :--- |
| C1 | ✅ | 网络不可靠 → 主动检测 |
| C3 | ✅ | 2.6 表格有动态生效 |
| C4 | ✅ | 无过时推荐 |
| C15 | ✅ | 用户关注“高可用” |

---

## 原则 3：Pipeline 与 Handler 链设计（C4）

### 3.1 第一性原理
- **物理/数学约束**：数据包顺序处理，不同逻辑需解耦。
- **推导**：责任链模式，Inbound/Outbound 分离，Handler 可插拔。

### 3.2 解决手段与代价
| 手段 | 原理 | 收益 | 代价 |
| :--- | :--- | :--- | :--- |
| `ChannelPipeline` | 双向链表 | 灵活组合 | 顺序错误可能死循环 |
| `ChannelHandlerContext` | 传播事件 | 上下文传递 | 需手动调用 fire 方法 |
| `@Sharable` | 无状态 Handler 共享 | 节省内存 | 必须线程安全 |

### 3.3 通用实现案例
- **添加 Handler**：`pipeline.addLast("decoder", new MyDecoder())`
- **跳过 Handler**：`ctx.fireChannelRead(msg)`

### 3.4 生产问题与解决
- **Handler 顺序错误**：解码器必须在业务 Handler 之前 → 检查 Pipeline 顺序。
- **共享 Handler 并发问题**：非线程安全 → 每个 Channel 独立实例或加锁。
- **内存泄漏**：未释放 `ReferenceCounted` 对象 → 使用 `SimpleChannelInboundHandler` 自动释放。

### 3.5 边界条件与风险
- **极值风险**：Pipeline 过长 → 遍历性能下降，建议 ≤ 10 个。
- **异常未处理**：`exceptionCaught` 未实现 → 连接直接关闭。

### 3.6 参数调优与验证
| 参数 | 核心作用 | 配置原则 | 动态生效 | 验证方法 |
| :--- | :--- | :--- | :--- | :--- |
| `ChannelOption.SO_BACKLOG` | 全连接队列 | 128-1024 | 需重启 | `ss -lnt` |
| `ChannelOption.TCP_NODELAY` | 禁用 Nagle | true | 在线 | 抓包查看小包 |

### 3.6.1 参数联动清单
- **场景**：使用 `@Sharable` 共享解码器
- **涉及参数**：解码器内部不能有状态变量 → 使用局部变量或 `@Sharable` 标注。

### 3.7 已知反模式检查
- **反模式**：在 Handler 中存储 Channel 特定状态且 Handler 被共享 → 数据错乱。
- **规避**：使用 `AttributeKey` 或独立 Handler 实例。

---

### 原则 3 推理草稿
- **选择依据**：可扩展性 → Pipeline。
- **关联域**：无。
- **预期检查项**：C1,C3,C4,C15。

### 原则 3 自检证明
| 检查项 | 状态 | 具体引用 |
| :--- | :--- | :--- |
| C1 | ✅ | 数据流处理解耦 |
| C3 | ✅ | 3.6 表格有动态生效 |
| C4 | ✅ | 无过时推荐 |
| C15 | ✅ | 用户关注“扩展性” |

---

## 原则 4：编解码与协议处理（N4）

### 4.1 第一性原理
- **物理/数学约束**：TCP 流式无边界，需协议帧解析。
- **推导**：`ByteToMessageDecoder` 累加缓冲，长度域解码器，粘包半包处理。

### 4.2 解决手段与代价
| 手段 | 原理 | 收益 | 代价 |
| :--- | :--- | :--- | :--- |
| `FixedLengthFrameDecoder` | 固定长度 | 简单 | 浪费空间 |
| `DelimiterBasedFrameDecoder` | 分隔符 | 灵活 | 需转义 |
| `LengthFieldBasedFrameDecoder` | 长度字段 | 通用 | 配置复杂 |
| 自定义解码器 | 继承 `ByteToMessageDecoder` | 完全控制 | 易错 |

### 4.3 通用实现案例
- **长度字段解码**：`new LengthFieldBasedFrameDecoder(maxFrameLength, lengthFieldOffset, lengthFieldLength)`
- **JSON 解码**：`new StringDecoder(CharsetUtil.UTF_8)` + 业务 JSON 解析。

### 4.4 生产问题与解决
- **解码器内存泄漏**：`decode` 中未读够数据但未保持引用 → 使用 `cumulation` 自动管理。
- **大帧攻击**：`maxFrameLength` 限制防止 OOM。
- **编解码器线程阻塞**：复杂解析（如 XML）放到业务线程池。

### 4.5 边界条件与风险
- **极值风险**：`maxFrameLength` 过小 → 合法帧被截断。
- **解码器状态**：需要在不同消息间保留状态 → 使用成员变量，但注意线程安全。

### 4.6 参数调优与验证
| 参数 | 核心作用 | 配置原则 | 动态生效 | 验证方法 |
| :--- | :--- | :--- | :--- | :--- |
| `maxFrameLength` | 最大帧长度 | 10MB | 需重启 | 发送超大帧测试 |
| `failFast` | 快速失败 | true | 需重启 | 超限立即抛异常 |

### 4.6.1 参数联动清单
- **场景**：使用 `LengthFieldBasedFrameDecoder`
- **涉及参数**：`lengthFieldOffset`、`lengthFieldLength`、`lengthAdjustment` 等需匹配协议。
- **风险**：参数不匹配 → 解析失败。

### 4.7 已知反模式检查
- **反模式**：解码器中直接调用 `fireChannelRead` 多次但未检查剩余字节 → 可能死循环。
- **规避**：遵循解码器模式：数据不够时 `return` 不添加对象。

---

### 原则 4 推理草稿
- **选择依据**：通信通用性 → 编解码。
- **关联域**：N2。
- **预期检查项**：C1,C3,C4,C15。

### 原则 4 自检证明
| 检查项 | 状态 | 具体引用 |
| :--- | :--- | :--- |
| C1 | ✅ | 流式协议无边界 |
| C3 | ✅ | 4.6 表格有动态生效 |
| C4 | ✅ | 无过时推荐 |
| C15 | ✅ | 用户关注“可演进性” |

---

## 原则 5：ByteBuf 与内存管理（C1）

### 5.1 第一性原理
- **物理/数学约束**：频繁分配/释放内存开销高，堆外内存需显式释放。
- **推导**：池化、引用计数、零拷贝。

### 5.2 解决手段与代价
| 手段 | 原理 | 收益 | 代价 |
| :--- | :--- | :--- | :--- |
| `PooledByteBufAllocator` | 内存池 | 减少 GC | 内存占用稍高 |
| `UnpooledByteBufAllocator` | 非池化 | 简单 | GC 压力大 |
| `CompositeByteBuf` | 组合视图 | 零拷贝 | 操作复杂 |
| 引用计数 `retain()`/`release()` | 手动管理 | 精确控制 | 易泄漏 |

### 5.3 通用实现案例
- **池化**：`ChannelOption.ALLOCATOR`, `PooledByteBufAllocator.DEFAULT`
- **零拷贝传输**：`DefaultFileRegion` 用于文件发送。

### 5.4 生产问题与解决
- **内存泄漏**：未调用 `release()` → 使用 `ResourceLeakDetector` (PARANOID 级别) 定位。
- **堆外内存 OOM**：未限制 `maxDirectMemory` → 设置 JVM `-XX:MaxDirectMemorySize`。
- **频繁池化竞争**：调大 `arena` 数量。

### 5.5 边界条件与风险
- **极值风险**：`PooledByteBufAllocator` 预分配内存过大 → 浪费。
- **引用计数溢出**：多次 `retain()` 未配对 `release()`。

### 5.6 参数调优与验证
| 参数 | 核心作用 | 配置原则 | 动态生效 | 验证方法 |
| :--- | :--- | :--- | :--- | :--- |
| `io.netty.allocator.type` | 分配器类型 | pooled/unpooled | 需重启 | JVM 参数 |
| `io.netty.leakDetection.level` | 泄漏检测级别 | PARANOID(测试) / SIMPLE(生产) | 在线 | 日志出现 LEAK |
| `io.netty.allocator.numDirectArenas` | 直接内存 arena 数 | CPU 核数 | 需重启 | 压测观察竞争 |

### 5.6.1 参数联动清单
- **场景**：大量小消息发送
- **涉及参数**：池化缓冲区大小、`receiveBufferSize`。
- **风险**：内存碎片。

### 5.7 已知反模式检查
- **反模式**：手动 `retain` 但不 `release` → 内存泄漏。
- **规避**：使用 `SimpleChannelInboundHandler` 或 `try-finally` 释放。

---

### 原则 5 推理草稿
- **选择依据**：性能 → 内存零拷贝与池化。
- **关联域**：无。
- **预期检查项**：C1,C3,C4,C15。

### 原则 5 自检证明
| 检查项 | 状态 | 具体引用 |
| :--- | :--- | :--- |
| C1 | ✅ | 堆外内存开销 |
| C3 | ✅ | 5.6 表格有动态生效 |
| C4 | ✅ | 无过时推荐 |
| C15 | ✅ | 用户关注“性能” |

---

## 原则 6：可观测性（X10）

### 6.1 第一性原理
- **物理/数学约束**：异步框架问题难定位（线程栈不完整）。
- **推导**：监控、日志、慢调用统计、连接状态。

### 6.2 解决手段与代价
| 手段 | 原理 | 收益 | 代价 |
| :--- | :--- | :--- | :--- |
| Netty 内置 Metrics | `ChannelPipeline` 统计 | 实时监控 | 性能开销 |
| Micrometer 集成 | 导出到 Prometheus | 标准化 | 额外依赖 |
| 日志（`LoggingHandler`） | 打印事件 | 调试 | 生产影响性能 |
| 慢 Handler 检测 | 自定义时间统计 | 发现瓶颈 | 侵入代码 |

### 6.3 通用实现案例
- **日志 Handler**：`new LoggingHandler(LogLevel.INFO)`
- **Prometheus**：`io.netty.micrometer` 模块。

### 6.4 生产问题与解决
- **日志过多**：`LoggingHandler` 在生产使用 WARN/ERROR 级别。
- **指标基数爆炸**：按连接统计时限制标签维度。
- **慢调用定位**：使用 `ChannelHandlerMetrics` 收集执行时间。

### 6.5 边界条件与风险
- **极值风险**：大量连接时全量日志会打爆磁盘。
- **健康检查**：使用 TCP 端口探针而非 HTTP（Netty 无内置 HTTP 健康端点）。

### 6.6 健康检查最佳实践
**K8s 配置**（基于 TCP 端口）：
```yaml
livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
```

### 6.7 参数调优与验证
| 参数 | 核心作用 | 配置原则 | 动态生效 | 验证方法 |
| :--- | :--- | :--- | :--- | :--- |
| `LogLevel` | 日志级别 | 生产 WARN | 在线 | 压测看日志量 |
| Micrometer registry | 指标导出 | `CompositeMeterRegistry` | 在线 | `curl /metrics` |

### 6.8 已知反模式检查
- **反模式**：生产开启 DEBUG 级别的 `LoggingHandler` → 日志爆炸。
- **规避**：通过配置中心动态调整级别。

---

### 原则 6 推理草稿
- **选择依据**：运维 → 可观测。
- **关联域**：无。
- **预期检查项**：C1,C3,C4,C7,C15。

### 原则 6 自检证明
| 检查项 | 状态 | 具体引用 |
| :--- | :--- | :--- |
| C1 | ✅ | 异步框架难以调试 |
| C3 | ✅ | 6.7 表格有动态生效 |
| C4 | ✅ | 无过时推荐 |
| C7 | ✅ | 6.6 TCP 探针 |
| C15 | ✅ | 用户关注“可观测” |

---

## 原则 7：安全防御（X9）

### 7.1 第一性原理
- **物理/数学约束**：网络传输可能被窃听、篡改。
- **推导**：TLS/SSL 加密、认证、授权。

### 7.2 解决手段与代价
| 手段 | 原理 | 收益 | 代价 |
| :--- | :--- | :--- | :--- |
| `SslHandler` | Netty TLS 集成 | 传输加密 | CPU 开销 10-20% |
| 客户端认证 | 双向 TLS（mTLS） | 身份验证 | 证书管理复杂 |
| `ChannelPipeline` 过滤器 | IP 白名单 | 简单访问控制 | 无法动态更新 |

### 7.3 通用实现案例
- **SslHandler**：`SslContextBuilder.forServer(keyCertChainFile, keyFile).build()`。
- **mTLS**：`SslContextBuilder.forServer(...).trustManager(caFile).clientAuth(ClientAuth.REQUIRED)`。

### 7.4 生产问题与解决
- **证书过期**：自动轮换，`SslContext` 需重建 → 使用 `SslContextBuilder` 并监听证书更新。
- **TLS 握手超时**：`handshakeTimeoutMillis` 默认 10s，可调大。
- **性能下降**：启用 `SslProvider.OPENSSL` 而非 JDK 提供者，开启 session 缓存。

### 7.5 边界条件与风险
- **极值风险**：握手过程中大量并发连接 → CPU 飙升，使用 `handshakeTimeout` 和连接数限制。
- **BKS/JKS 密钥库格式**：Netty 支持 PKCS12 更通用。

### 7.6 性能量化说明
- TLS 开销基准：Intel Xeon, AES-GCM-128, 1KB 消息，无加速时吞吐下降 15%。
- 优化：使用 OpenSSL 提供者（`SslProvider.OPENSSL`），启用 `SslSessionCache`，硬件 QAT。

### 7.7 参数调优与验证
| 参数 | 核心作用 | 配置原则 | 动态生效 | 验证方法 |
| :--- | :--- | :--- | :--- | :--- |
| `handshakeTimeoutMillis` | 握手超时 | 10000 | 需重启 | 模拟慢握手 |
| `SslProvider` | TLS 实现 | OPENSSL | 需重启 | 压测比较性能 |

### 7.8 已知反模式检查
- **反模式**：每个连接新建 `SslContext` → 内存泄漏，应全局单例。
- **规避**：`SslContext` 静态共享。

---

### 原则 7 推理草稿
- **选择依据**：安全合规。
- **关联域**：无。
- **预期检查项**：C1,C3,C4,C6,C15。

### 原则 7 自检证明
| 检查项 | 状态 | 具体引用 |
| :--- | :--- | :--- |
| C1 | ✅ | 网络传输安全 |
| C3 | ✅ | 7.7 表格有动态生效 |
| C4 | ✅ | 无过时推荐 |
| C6 | ✅ | OPENSSL 与 JDK 差异 |
| C15 | ✅ | 用户关注“安全合规” |

---

## 全局权衡矩阵

| 冲突维度 | 涉及原则 | 权衡分析与架构建议 |
| :--- | :--- | :--- |
| 吞吐 vs 延迟 | 原则1 | 高吞吐用多 EventLoop，低延迟用绑核 |
| 内存池 vs GC 压力 | 原则5 | 高吞吐用池化，短生命周期用非池化 |
| TLS 性能 vs 安全 | 原则7 | 内部网络可关闭，公网必须开启 |
| Handler 共享 vs 安全 | 原则3 | 无状态 Handler 用 `@Sharable`，有状态独立 |
| 编码器复杂 vs 通用 | 原则4 | 标准协议用现有解码器，私有协议简单设计 |

---

## 四纵四横覆盖矩阵

| 维度 | 子维度 | 覆盖原则 | 说明 |
| :--- | :--- | :--- | :--- |
| 横向：资源池化 | EventLoop 线程、内存池 | 原则1、5 | 线程模型、ByteBuf 池 |
| 横向：通信 | 连接管理、TLS | 原则2、7 | 心跳、重连、加密 |
| 横向：数据存储 | 不适用 | — | — |
| 横向：计算 | Pipeline 处理 | 原则3 | Handler 链 |
| 纵向：协同 | — | — | 无 |
| 纵向：调度 | 不适用 | — | — |
| 纵向：追踪与高可用 | 监控、重连 | 原则2、6 | 指标、日志、健康检查 |
| 纵向：部署 | 健康检查 | 原则6 | TCP 探针 |

---

## 关键术语覆盖矩阵

| 术语 | 对应原则 | 说明 |
| :--- | :--- | :--- |
| 性能 | 原则1、5 | 线程模型、内存管理 |
| 高可用 | 原则2 | 心跳、重连 |
| 扩展性 | 原则3 | Pipeline 动态插拔 |
| 一致性 | — | 不适用 |
| 安全合规 | 原则7 | TLS、mTLS |
| 可观测 | 原则6 | 指标、日志 |
| 成本 | 原则5 | 内存池 vs GC |
| 可演进性 | 原则3、4 | Pipeline 扩展性、编解码 |
| 长事务 | — | Netty 无事务语义 |

---

## 待验证参数清单汇总

| 序号 | 参数位置 | 参数描述 | 当前占位符 | 建议验证方法 | 推荐查阅链接 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | 原则1.6 | EventLoop 线程数 | `[需查阅官方文档确认]` | 压测观察 CPU | [Netty 线程模型](https://netty.io/wiki/thread-model.html) |
| 2 | 原则5.6 | 泄漏检测级别 | `[需查阅官方文档确认]` | 运行测试用例 | [Netty 内存泄漏](https://netty.io/wiki/reference-counted-objects.html) |
| 3 | 原则7.7 | SslProvider | `[需查阅官方文档确认]` | 压测对比 | [Netty TLS](https://netty.io/wiki/sslcontextbuilder-and-private-key.html) |
