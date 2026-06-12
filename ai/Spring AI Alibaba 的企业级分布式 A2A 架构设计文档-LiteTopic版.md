
> **最后更新**：2026-06-12  
> **适用框架**：Spring AI Alibaba 1.1.2+、Nacos 3.1.0+、Apache RocketMQ 5.2+

---
## 1. 架构概述

本架构基于 **Spring AI Alibaba**，结合 **Nacos**（控制平面）与 **Apache RocketMQ**（数据平面），构建控制面与数据面分离的分布式 Agent-to-Agent (A2A) 协作系统，解决长耗时阻塞、算力过载、安全越权等痛点。核心设计：动态最小权限、异步事件驱动、高可用容错、全链路可观测。

---
## 2. 核心组件选型

| 组件 | 角色 | 核心职责 |
|------|------|----------|
| **Nacos 3.1.0+** | 控制平面 | Agent 注册（AgentCard）、动态配置、统一权限管理 |
| **RocketMQ 5.2+** | 数据平面 | 异步通信、会话级顺序消息、回调汇聚、死信队列、削峰填谷 |

---
## 3. 详细设计

### 3.1 Agent 注册与发现

Agent 启动时注册 `AgentCard` 至 Nacos，扩展 `dataPlane.inboundTopic`。

```json
{
  "name": "order-agent",
  "version": "1.0.0",
  "capabilities": ["query_order", "update_order"],
  "endpoint": "https://order-agent.example/api/a2a",
  "dataPlane": { "inboundTopic": "agent_order_inbound" }
}
```
> `endpoint` 用于同步握手、健康检查等控制面交互，业务数据走 RocketMQ。

### 3.2 异步协作数据流

#### 任务分发
1. 判断业务 Payload 大小：
   - 若 ≤1MB：直接以原始 Payload 作为消息体，计算其 SHA256 作为 `payload_hash`。
   - 若 >1MB：先将 Payload 上传至 OSS，生成预签名 URL（有效期 10 分钟），并设置自定义元数据 `x-oss-meta-md5` 为文件的 MD5 值（32 位十六进制）；以该 URL 字符串作为消息体，计算其 SHA256 作为 `payload_hash`。
2. 向**认证授权中心**请求 JIT Token，携带 `user_id`、`payload_hash`、所需权限范围。
3. Auth Center 使用私钥 (RS256) 签发 JWT（有效期 **60 秒**）并返回。
4. 构造 RocketMQ 消息：
   - 消息体：原始 Payload（≤1MB）或 OSS URL（>1MB）
   - 消息属性：`trace_id`、`session_id`、`jit_token`、`retry_count`（初始 0），以及大文件时的 `file_md5`
5. 根据 Nacos 中目标 Agent 的 `inboundTopic` 发送消息。

#### 目标 Agent 处理
1. 消费消息，提取 `jit_token`。
2. 使用 Auth Center 公钥验签 JWT，检查 `exp`（允许 +5s 偏差）。
   - 若 Token 已过期，返回特定错误回调 `error_code = "TOKEN_EXPIRED"`。
3. 从 JWT 中获取 `payload_hash`（小写十六进制字符串），计算实际消息体（若为 OSS URL 则计算 URL 字符串）的 SHA256 并比对；若为文件，下载后计算文件 MD5 并与消息属性中的 `file_md5` 比对。
4. 将 `user_id` 及权限条件存入 `SecurityContext`（ThreadLocal）。
5. 执行工具调用，`DataProxyAdvisor` 自动注入权限条件。
6. 结果发送至固定回调 Topic `agent_callback`，按 `session_id` 哈希选择队列。

#### 回调结果路由（队列分区）
- 预先创建固定队列数（如 64 或 128）的 Topic `agent_callback`。
- 队列选择：`int index = Math.abs(sessionId.hashCode()) % queueNum`。
- 每个 RouteAgent 实例通过 `AllocateMessageQueueAveragely` 消费部分队列。
- **安全建议**：为 `agent_callback` 配置 RocketMQ ACL，仅允许可信 Agent 发送，RouteAgent 仅消费。

#### Token 过期处理（重试刷新）
- 当 RouteAgent 收到 `TOKEN_EXPIRED` 回调或超时且 `retry_count < 1` 时，**重新执行完整分发流程**：
  - 对于大文件：OSS URL 保持不变（无需重新上传），用相同的消息体（URL）重新计算 `payload_hash`。
  - 对于小文件：直接用原始 Payload 重新计算 `payload_hash`。
  - 向 Auth Center 请求**新 JWT**。
  - 构造新消息，`retry_count` 加 1，重新发送。
- 若 `retry_count >= 1` 仍失败，则放弃并写入死信队列，触发 P0 告警。

### 3.3 顺序消息实现

**发送端（队列选择）**：
```java
producer.send(msg, (mqs, msg, arg) -> {
    String sessionId = (String) arg;
    int index = Math.abs(sessionId.hashCode()) % mqs.size();
    return mqs.get(index);
}, sessionId);
```

**消费端**：
```java
consumer.registerMessageListener(new MessageListenerOrderly() {
    public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext ctx) {
        // 顺序处理
        return ConsumeOrderlyStatus.SUCCESS;
    }
});
```
- 建议配置 `consumeThreadMax=64`，消费超时 30 秒。
- **异常处理**：
  - 对于**非严格顺序**业务，重试超过 3 次后返回 `SUCCESS` 并写死信队列，避免阻塞。
  - 对于**严格顺序**业务，重试失败后应暂停队列并触发 P0 告警，人工介入处理（不可跳过）。

### 3.4 安全与权限

#### 3.4.1 JIT Token（即时令牌）
- **签发**：独立 Auth Center 使用私钥 (RS256) 签发 JWT，RouteAgent 不可自行签名。
- **载荷**：
```json
{
  "iss": "auth-center",
  "exp": now+60s,
  "user_id": "12345",
  "payload_hash": "sha256(消息体)"
}
```
> `payload_hash` 为消息体原始字节的 SHA256 哈希值，表示为**小写十六进制字符串**（长度 64 字符）。消息体：小文件为原始 Payload，大文件为 OSS URL。
- **校验**：验签 → 检查 exp（允许 +5s 偏差） → 计算消息体哈希比对。
- **大文件**：`payload_hash` 校验 URL 完整性；文件内容 MD5 通过消息属性 `file_md5` 独立校验。
- **有效期设计理由**：60 秒有效期可在安全性与消息链路延迟容忍度之间取得平衡，配合重试刷新机制（见 3.2）解决过期问题。

#### 3.4.2 Data Proxy（自动权限注入）
目标 Agent 验签后将权限条件存入 `SecurityContext`：
```java
SecurityContext.setCondition("user_id = '12345'");
```
`DataProxyAdvisor` 拦截工具调用，注入条件：
```java
@Around("@annotation(com.example.Tool)")
public Object injectCondition(ProceedingJoinPoint pjp) {
    String condition = SecurityContext.getCurrentCondition();
    // 修改 SQL 参数...
    return pjp.proceed(args);
}
```
务必在请求结束时 `SecurityContext.clear()`。

#### 3.4.3 HITL（人机回环）
- 高风险任务挂起，生成 `task_token`（SecureRandom 32 字节十六进制）。
- Redis 存储：
```json
{
  "user_id": "12345",
  "original_request": {...},
  "expire_at": now+24h
}
```
- 前端调用确认接口，需同时提供 `task_token` 和用户 Access Token（HTTPS）。
- 后端校验 Access Token 有效且 `user_id` 匹配，执行任务并删除 `task_token`（一次性）。

#### 3.4.4 通信加密
- Nacos、RocketMQ 开启 TLS；A2A REST API 仅 HTTPS + 服务网格 mTLS。

### 3.5 异常处理与容错

| 场景 | 策略 |
|------|------|
| 消费失败（业务错误） | 指数退避重试 3 次 → 死信队列 |
| Token 过期 | RouteAgent 重新请求新 Token 并重发（最多 1 次），仍失败 → 死信队列 |
| 回调超时 | 60 秒超时，按 Token 过期重试逻辑处理（可能触发刷新） |
| 死信队列 | 自动重试 3 次仍失败 → P0 告警，运维通过 Dashboard 重投或跳过 |
| 顺序消息阻塞 | 根据业务严格程度选择跳过或暂停告警（见 3.3） |
| 优雅下线 | 注销 Nacos → `consumer.pause()` → 等待 `CountDownLatch` (30s) → 退出 |

### 3.6 大文件旁路处理
- Payload >1MB 上传至 OSS，生成预签名 URL（有效期 10 分钟），并设置自定义元数据 `x-oss-meta-md5`。
- 消息属性携带 `file_md5`（即自定义元数据中的 MD5 值）。
- 目标 Agent 下载后比对文件 MD5，不一致则拒绝处理。
- 消费后发送 `file_consumed` 触发立即删除；每日定时清理超 1 小时文件。

---
## 4. Spring AI Alibaba 集成要点

| 能力 | 用途 | 集成方式 |
|------|------|----------|
| Advisor | Data Proxy 权限注入 | 自定义 `MethodInterceptor` 注册到 `ChatClient` |
| Tool 注解 | 定义工具 | `@Tool` |
| Graph | 工作流编排 | `Graph.builder()` |
| Observability | 追踪与指标 | `ObservationRegistry` + `Micrometer` |

**示例**：
```java
@Configuration
class AIConfig {
    @Bean ChatClient chatClient(ChatClient.Builder builder, DataProxyAdvisor advisor, ObservationRegistry registry) {
        return builder.defaultAdvisors(advisor).observationRegistry(registry).build();
    }
}
```

---
## 5. 可观测性与告警

### 指标

| 指标 | 类型 | 说明 |
|------|------|------|
| `agent_call_duration_seconds` | Histogram | P99 延迟 |
| `agent_token_consumption_total` | Counter | Token 消耗 |
| `rocketmq_callback_queue_pending` | Gauge | 各队列堆积 |
| `dlq_messages_total` | Counter | 死信数量 |
| `security_auth_rejection_total` | Counter | 鉴权拒绝（含 Token 过期） |
| `token_refresh_retry_total` | Counter | Token 过期后刷新重试次数 |

### 告警
- P99 >2s 持续 5min → P2
- 任一队列堆积 >100 持续 1min → P1
- 死信深度 >0 持续 5min → P0
- 拒绝率 >5%/min → P1
- Token 刷新重试频率 >10/min → P2（可能表示系统延迟过高）

---
## 6. 总结

本架构通过 Nacos 与 RocketMQ 的分工协作，结合 Spring AI Alibaba 的生态能力，实现了安全、可靠、可运维的企业级多 Agent 系统。所有设计均已考虑异步完整性、防篡改、防越权、高并发容错，以及 **Token 过期自动刷新** 机制，确保消息链路延迟不会导致永久失败。