
> **文档定位**：通过费曼学习法（通俗解释）与第一性原理（从本质推导设计）的双重视角，系统掌握 Spring AI Alibaba 的核心原理、生产实践与调优方法。学完本文档后，将具备独立搭建生产级 AI 应用、定位线上问题并对常规场景进行性能调优的能力。

## 一、架构设计

### 1.1 快速入门：最小化项目配置

在深入了解架构原理之前，最快的方式是通过 Spring Boot Starter 快速启动一个 AI 项目。

```xml
<!-- pom.xml 依赖 -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-alibaba-starter</artifactId>
    <version>1.0.0.2</version>
</dependency>
```

只需添加这一依赖，Spring Boot 就会通过自动装配机制（Auto-Configuration）自动创建 `ChatClient.Builder`、`ChatModel` 等核心 Bean，开发者无需手写任何配置类便可直接注入使用。

```yaml
# application.yml
spring:
  ai:
    dashscope:
      api-key: ${AI_DASHSCOPE_API_KEY}
```

```java
@RestController
public class ChatController {
    private final ChatClient chatClient;
    
    public ChatController(ChatClient.Builder builder) {    // 框架自动注入 Builder
        this.chatClient = builder.build();
    }
    
    @GetMapping("/ai")
    public String chat(@RequestParam String msg) {
        return chatClient.prompt().user(msg).call().content();
    }
}
```


### 1.2 Spring AI vs Spring AI Alibaba 定位说明

| 维度 | Spring AI | Spring AI Alibaba |
|------|-----------|-------------------|
| **定位** | AI 应用开发**底层框架** | AI 智能体**高层框架** |
| **提供能力** | 模型适配、工具定义、向量数据库存取等底层原子抽象 | 基于图算法的智能体编程 Graph 框架，让开发者更容易开发工作流、multi-agent 应用 |
| **类比** | “发动机”——提供 AI 交互的核心能力 | “整车”——提供完整的工作流、Agent 编排能力 |

Spring AI 的设计目标是提供一个 Spring 友好的抽象层和 API，将 Spring 生态系统的设计原则（可移植性、模块化）应用到 AI 领域，促进使用 POJO 作为 AI 应用程序的构建块。Spring AI Alibaba 则以 Spring AI 为基础，深度集成百炼平台，构建更高层的智能体开发框架。Spring AI 提供的自动配置会自动创建一个原型化的 `ChatClient.Builder` Bean 供开发者注入使用，这是两个框架协同工作的关键机制。

### 1.3 整体分层架构

Spring AI Alibaba 项目从架构上包含如下三层：

| 层级 | 核心能力 | 说明 |
|------|----------|------|
| **Agent Framework** | 以 ReactAgent 为核心的 Agent 开发框架 | 使开发者能够构建具备自动上下文工程和人机交互等核心能力的 Agent |
| **Graph** | 低级别的工作流和多 Agent 协调框架 | Graph 是 Agent Framework 的底层运行时基座，具备丰富的预置节点和简化的图状态定义 |
| **Augmented LLM** | Spring AI 框架底层原子抽象 | 包括 Model、Tool、MCP、Message、VectorStore 等 |

> 🧠 **费曼解释**：三层架构就像一个三层餐厅服务体系：最上层是拿着菜单给你点单的“服务员”（Agent Framework），中间层是协调后厨各工位按正确顺序做菜的“厨房调度”（Graph），最底层则是负责每个具体烹饪动作的“厨具和基础设备”（Augmented LLM）。

> 🔬 **第一性原理推理**：Graph 是整个 Agent 系统的“地基”——它只关心一件事：确保工作流中的节点按正确顺序执行、状态在节点之间传递。AI 工作流的本质是“有向图上的状态迁移”：每个节点（Node）执行一个原子操作（如调用模型、检索文档、执行工具），边（Edge）定义节点之间的依赖关系与数据流向。ReactAgent 建立在这个地基之上，开发者可以直接使用预制能力而无需自己砌砖。


## 二、核心原理与算法

### 2.1 同步/异步与流式处理

Spring AI 定义了两个核心接口：

```java
// 同步模型接口
public interface Model<TReq extends ModelRequest<?>, TRes extends ModelResponse<?>> {
    TRes call(TReq request);
}

// 流式模型接口  
public interface StreamingModel<TReq extends ModelRequest<?>, TResChunk extends ModelResponse<?>> {
    Flux<TResChunk> stream(TReq request);
}
```

ChatModel 同时实现这两个模式：
```java
public interface ChatModel extends Model<Prompt, ChatResponse>, StreamingChatModel {
    ChatResponse call(Prompt prompt);        // 同步：阻塞直到完整返回
    Flux<ChatResponse> stream(Prompt prompt); // 异步流式：逐块返回（每个块包含一个或多个 Token）
}
```

> 🔬 **第一性原理推理**：LLM 生成是一个“从 Token 分布中逐步采样”的过程。同步调用相当于“等 LLM 生成完所有 Token 才一次返回”，流式调用则是“每生成一个块（可能包含多个 Token）就立即发送”。首块延迟取决于模型计算第一个响应块的时间，而总时间不变。选择同步还是流式，取决于你在“首块延迟”和“实现复杂度”之间如何权衡——这是 I/O 模型与用户体验之间的本质 Trade-off。

### 2.2 Tool Calling（函数调用）

> 📖 **术语说明**：该能力在官方最新文档中正式名称为 **Tool Calling**。LLM 本身不能实际调用工具，它们会在响应中表达调用特定工具的意图，应用程序执行该工具并报告结果给模型。

#### 工作流程

Tool Calling 允许大型语言模型（LLM）在必要时调用一个或多个可用的工具，这些工具通常由开发者定义。LLM 本身不能实际调用工具——它只负责决策和参数填充。

> 🔬 **第一性原理推理**：LLM 并不直接执行任何代码。它只是输出一个符合特定 JSON Schema 的工具调用指令（包含 name 和 arguments）。Spring AI Alibaba 的 Tool Calling 机制的底层逻辑是：框架在启动时通过反射扫描带有 `@Tool` 注解的方法，将其元数据（名称、描述、参数 Schema）注入到 Prompt 中；当 LLM 返回 `tool_calls` 指令时，`ToolCallback` 机制解析该指令，通过 Spring 的 `ApplicationContext` 获取对应 Bean，利用 Java 反射执行该方法，再将执行结果回填给 LLM，由 LLM 生成最终的人类可读回复。

#### 关键类型

Spring AI 支持两种定义工具的方式：**Method as Tools**（通过 `@Tool` 注解）和 **Function as Tools**。目前，Method as Tools 不支持以下类型的参数和返回类型：`Optional`、异步类型（`CompletableFuture`、`Future`）、响应式类型（`Mono`、`Flux`）、函数类型（`Function`、`Supplier`、`Consumer`）。

> 🧠 **费曼解释**：LLM 不知道怎么做数学计算，就像一个人不知道怎么修车。但这个人可以：①（在请求时）告诉你“如果遇到修车问题，可以打电话给李师傅”【提供工具定义】；②（遇到问题时）决定“哦，这是个修车问题，我得打给李师傅”【模型决定调用】；③（收到修好后的反馈）“我已经让李师傅修好了，这是结果”【应用执行工具并回传】。Tool Calling 就是让 LLM 在需要时“打电话找专家”，而不是自己硬答。

#### ⚠️ 工具调用降级方案（生产必读）

⚠️ Spring AI 官方的 `@Tool` 注解并未提供 `fallbackMethod` 属性。要实现工具调用的降级，需采用以下两种方式之一：

**方案一：工具内部自行实现降级逻辑**
```java
@Component
public class WeatherTools {
    @Tool(description = "获取指定城市的天气信息")
    public WeatherResponse getWeather(String city) {
        try {
            return weatherApiClient.query(city);   // 可能超时或异常
        } catch (TimeoutException e) {
            // 降级：返回缓存数据或友好提示
            return getCachedWeather(city);
        } catch (Exception e) {
            return new WeatherResponse(city, null, "天气服务暂时不可用");
        }
    }
    
    private WeatherResponse getCachedWeather(String city) {
        // 读取本地缓存或返回预设默认值
        return cache.getOrDefault(city, new WeatherResponse(city, 25.0, "celsius", "数据待更新"));
    }
}
```

**方案二：通过外层拦截器实现统一降级**
```java
@Component
public class FallbackToolInterceptor implements ToolCallInterceptor {
    @Override
    public Object aroundCall(ToolCallContext context, ToolInvocationChain chain) {
        try {
            return chain.proceed(context);
        } catch (Exception e) {
            log.error("Tool {} execution failed", context.getToolName(), e);
            return new FallbackResponse("服务暂时不可用");
        }
    }
}
```


### 2.3 RAG 实现原理

#### 基础版 RAG 流程

**离线阶段（数据准备）** ：外部文档 → 文档加载 → 文本分块 → Embedding 向量化 → 向量数据库存储。

**在线阶段（问答）** ：用户提问 → 问题向量化 → 向量数据库相似性搜索 → 检索结果与用户问题拼接 → 构建 Prompt → LLM 生成 → 返回答案。

Spring AI Alibaba 中，向量数据库存储 AI 模型不知道的数据。当用户问题发送到 AI 模型时，`QuestionAnswerAdvisor` 会查询向量数据库以获取与用户问题相关的文档，并附加到用户文本中，为 AI 模型提供生成响应的上下文。

```java
// 配置 RAG Advisor
var qaAdvisor = QuestionAnswerAdvisor.builder(vectorStore)
    .searchRequest(SearchRequest.builder()
        .similarityThreshold(0.8d).topK(6).build())
    .build();

// 通过 ChatClient 使用 RAG（构建时注册）
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(qaAdvisor)
    .build();
```

> 🔬 **第一性原理推理**：RAG 的本质是“**在推理时引入外部知识**”。LLM 的原始训练数据被冻结在某个时间点，无法覆盖私有数据或最新信息。RAG 将“检索”和“生成”分离：检索负责从外部知识库中找到相关信息，生成负责基于检索结果回答问题。这种设计的核心优势在于：知识更新成本低（只需更新向量数据库，无需重新训练模型），且可解释性强（可以直接看到检索到的来源）。


## 三、主要功能

### 3.1 核心能力矩阵

| 功能类别 | 核心实现 | 典型场景 |
|----------|----------|----------|
| 同步/异步/流式对话 | `ChatModel.call()` / `ChatClient.stream()` | 实时聊天、文本生成 |
| 结构化输出 | `ChatClient.entity(Class.class)` | 报表提取、信息抽取、数据整理 |
| 工具调用 | `@Tool` 注解 + `ToolCallback` | 数学计算、外部 API 调用、搜索 |
| RAG 文档问答 | `QuestionAnswerAdvisor` + VectorStore | 企业知识库、智能客服 |
| Agent 多智能体 | ReactAgent / SequentialAgent / ParallelAgent | 复杂任务自动规划执行 |
| Graph 工作流 | StateGraph API 编排多 Agent | 复杂业务流程编排 |

> 🧠 **费曼解释**：主要功能可以理解为“给 AI 添加各种外挂”：结构化输出 = 强迫 AI 填表格；RAG = 给 AI 开卷考试；Agent = 让 AI 自己规划怎么做；Tool Calling = 让 AI 能使用计算器、搜索引擎等工具。

### 3.2 结构化输出示例

将 AI 模型输出映射到 POJOs（Structured Output）是 Spring AI Alibaba 的核心功能之一。

```java
// 定义返回实体
public record ActorMovies(String actor, List<String> movies) { }

// Controller 中使用
@RestController
@RequestMapping("/movies")
public class MovieAiController {
    private final ChatClient chatClient;
    
    public MovieAiController(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder.build();
    }
    
    @RequestMapping
    public ActorMovies movies(@RequestParam(value = "message") String message) {
        return this.chatClient.prompt()
            .user(message)
            .call()
            .entity(ActorMovies.class);  // 自动映射为 Java Record
    }
}
```


## 四、常见生产问题分类与解决办法

### 4.1 网络与限流类

| 问题类型 | 典型报错特征 | 深层原因 | 排查顺序与解决方案 |
|----------|-------------|----------|------------------|
| 连接超时 | `Connection timed out` | 网络策略未放行或端点配置错误 | 检查网络安全组/防火墙规则；验证 `spring.ai.dashscope.api-key` 和端点配置 |
| 认证失败 | `401 Unauthorized` | API Key 错误或配置被覆盖 | 按顺序排查：application.yml → 环境变量 → 配置文件覆盖优先级 |
| QPS 超限 | `429 Too Many Requests` | 不仅是 QPS 超限，还可能是 Token Per Minute (TPM) 超限 | 实现客户端限流器；实现延迟重试 + 请求排队策略 |
| Token 配额耗尽 | `Quota exceeded` | 账户余额不足或月配额用完 | 检查账户余额和账单；实现降级策略（缓存、本地回复、切换备用模型） |

### 4.2 解析与转换类

| 问题类型 | 典型报错特征 | 深层原因 | 解决方案 |
|----------|-------------|----------|----------|
| 工具调用参数解析失败 | `JSON parse error` | LLM 在高并发或上下文过长时生成格式错误的 JSON；**工具方法返回类型不支持 Flux 等异步类型** | 在 `@Tool` description 中强制约束 JSON 格式；将工具方法的返回类型从 `Flux<T>` 改为 `T` 类型；检查是否使用了不支持的参数/返回类型 |
| 结构化输出类型转换异常 | `Conversion failed` | LLM 返回的内容无法映射为目标 Java 类型 | 使用 `ParameterizedTypeReference` 处理泛型集合；在测试环境中先获取原始响应进行格式验证 |
| 流式响应解析不完整 | 返回内容被截断 | 未正确累积 `Flux<ChatResponse>` 中的所有块 | 使用 `collectList()` 或 `reduce()` 进行流式聚合 |

### 4.3 内存与性能类

| 问题类型 | 典型现象 | 深层原因 | 解决方案 |
|----------|----------|----------|----------|
| 流式响应未关闭导致内存泄漏 | 内存持续增长、GC 频繁 | Reactor 的 `Flux` 如果未被最终订阅或设置背压，会导致数据在内存中堆积 | 在 Controller 层返回 `Flux` 让 WebFlux 自动管理生命周期；使用 `.limitRate()` 设置背压；使用 `.timeout()` 设置超时保护 |
| 长上下文 Token 浪费 | Token 用量远超预期、费用飙升 | 每次请求都携带完整对话历史 | 实现窗口管理：仅保留最近 N 轮对话；对过期对话进行摘要压缩 |
| 大文档一次性加载 OOM | `OutOfMemoryError` | 将超大文档整体加载到内存中处理 | 分块流式处理文档；使用框架内置的 `DocumentSplitter` 逐块处理 |

> 🔬 **第一性原理推理**：流式响应内存泄漏的根源在于响应式编程的资源生命周期管理。在 Reactor 中，`Flux` 是一个惰性的发布者——仅仅创建它不会执行任何操作，只有被订阅（`subscribe()`）时数据才会开始流动。如果 `Flux` 被创建但从未被正确订阅或取消，它所引用的资源（网络连接、缓冲区、线程上下文）就不会被释放。

### 4.4 并发与线程类

| 问题类型 | 典型现象 | 解决方案 |
|----------|----------|----------|
| Reactor 线程阻塞 | 应用响应变慢、线程堆积，在 WebFlux 线程中执行阻塞操作导致超时 | 使用 `.subscribeOn(Schedulers.boundedElastic())` 将阻塞操作转移到专用线程池；**避免在响应式链中直接调用同步 JDBC 或 HTTP 调用** |
| MDC 上下文丢失 | `ThreadLocal` 数据在异步线程中丢失 | 使用 Reactor 的 `Context` 传递元数据；或使用 `MDCContextLifter` 自动传递 MDC 上下文 |


## 五、调优

### 5.1 模型参数调优

Spring AI 支持在三个级别设置 `ChatOptions`：应用默认（通过自动配置）、`ChatClient.Builder` 默认、以及单次请求覆盖。**为了保证代码的可移植性（避免绑定到特定模型提供商）** ，通用配置应使用标准前缀 `spring.ai.chat.options.*`，厂商特有配置再使用各自前缀（如 `spring.ai.dashscope.*`）。

```yaml
# ✅ 推荐：通用配置使用标准前缀
spring:
  ai:
    chat:
      options:
        temperature: 0.7
        max-tokens: 1000
    dashscope:
      api-key: ${AI_DASHSCOPE_API_KEY}   # 厂商特有配置

# ⚠️ 避免：与厂商混在一起，降低可移植性
# spring:
#   ai:
#     dashscope:
#       chat:
#         options:
#           temperature: 0.7   # 仅 DashScope 生效，切换模型后不生效
```

| 参数 | 作用 | 事实性任务（FAQ、客服） | 创意任务（写作、头脑风暴） |
|------|------|------------------------|--------------------------|
| `temperature` | 控制输出随机性（0~1），调整 Softmax 分布锐度 | 0.01~0.1（**不要设为 0**） | 0.7~0.9 |
| `topP` | 核采样阈值，动态调整采样词汇范围 | 0.9~1.0 | 0.9~1.0 |
| `maxTokens` | 最大输出 Token 数 | 500~1000 | 1500~4000 |

> 🔬 **第一性原理推理**：`temperature` 通过调整 Softmax 输出的“分布锐度”控制多样性：高 temperature 使概率分布更平缓、模型更“敢选”低概率词；低 temperature 使分布更尖锐、模型固定选最高概率词。**关键经验**：`temperature` 不要设为 0——当概率分布过于极端时，模型可能陷入“死循环”，反复输出相同的 Token 序列。

### 5.2 提示词工程优化

Spring AI Alibaba 支持通过 Nacos 配置中心实现动态 Prompt 模板管理：

| 优化策略 | 做法 | 效果 |
|----------|------|------|
| 少样本设计 | 在系统提示中提供 2~3 个“问题—标准答案”示例 | 将零样本准确率从 ~60% 提升到 ~85% |
| 指令清晰化 | 使用分隔符（`###`、`---`）区分指令、上下文和用户输入 | 减少 LLM 误读指令的概率 |
| 动态模板管理 | 将 Prompt 托管至 Nacos 配置中心，实现不重启应用的热更新 | A/B 测试 Prompt 效果、快速响应业务需求变化 |

### 5.3 RAG 分块策略与向量检索调优

#### 分块大小与重叠

| 文档类型 | 推荐 chunk 大小 | 重叠（overlap）比例 | 理由 |
|----------|----------------|---------------------|------|
| 技术文档 | 256~512 tokens | 10~20% | 代码块较短，小块更易命中；**建议使用 `TokenTextSplitter` 而不是简单的字符截断** |
| 法律/政策 | 512~1024 tokens | 15~25% | 条款间关联紧密，适当重叠防止信息断裂 |
| 对话记录 | 128~256 tokens | 5~10% | 每轮对话短，小块更精准 |

#### 向量检索调优

| 参数 | 推荐值 | 调优方向 |
|------|--------|----------|
| `similarityThreshold` | 0.7 | 召回结果太多噪音 → 提高到 0.8；召回结果太少 → 降低到 0.6 |
| `topK` | 3~5 | 过多的上下文会挤占 Token 限额，导致模型“看不全”问题 |

### 5.4 超时与流式调优

```java
// 流式调优示例：背压控制和超时保护
Flux<String> stream = chatClient.prompt()
    .user(query)
    .stream()
    .content()
    .limitRate(20)           // 背压：每次最多请求 20 个块
    .timeout(Duration.ofSeconds(30))  // 超时保护
    .retryWhen(Retry.backoff(3, Duration.ofSeconds(1)))  // 重试策略
    .onErrorResume(e -> Flux.just("抱歉，服务暂时不可用"));
```

⚠️ **生产级流式输出线程隔离**（关键性能实践）：

在 Controller 层处理流式输出时，如果需要在流中记录审计日志到 MySQL 或执行其他阻塞 I/O 操作，**必须通过 `subscribeOn(Schedulers.boundedElastic())` 将阻塞操作转移到专用线程池，防止阻塞 Event Loop 线程**。

```java
@RestController
public class StreamingController {
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamChat(@RequestParam String query) {
        Flux<String> aiResponse = chatClient.prompt()
            .user(query)
            .stream()
            .content();
        
        // 将阻塞操作（如 MySQL 日志）隔离到 boundedElastic 线程池
        aiResponse = aiResponse.flatMap(chunk -> 
            Mono.fromRunnable(() -> saveAuditLog(chunk))   // 阻塞操作
                .subscribeOn(Schedulers.boundedElastic())  // 关键：线程隔离
                .thenReturn(chunk)
        );
        
        return aiResponse;
    }
    
    private void saveAuditLog(String chunk) {
        // 同步 JDBC 操作，阻塞风险高，必须在 boundedElastic 中执行
        jdbcTemplate.update("INSERT INTO ai_audit_log (chunk) VALUES (?)", chunk);
    }
}
```

`Schedulers.boundedElastic()` 会为每个阻塞操作创建一个专用线程来等待阻塞资源，不会影响其他非阻塞处理，同时确保线程数量有界。


## 六、最佳实践

### 6.1 结构化输出设计模式

```java
// ✅ 推荐：使用 Record 配合 Bean Validation，并提供默认值兜底
public record CustomerOrder(
    @NotBlank String customerName,
    @Min(1) @Max(999999) Long orderId,
    @Positive BigDecimal totalAmount,
    @Pattern(regexp = "PENDING|PAID|SHIPPED|DELIVERED")
    String status,
    @Default("当前时间") String timestamp   // 提供默认值兜底
) {}
```

### 6.2 安全工具调用（降级处理）

⚠️ 工具内部必须实现降级逻辑。由于 `@Tool` 注解不支持 `fallbackMethod`，需在工具方法内部显式捕获所有异常并返回降级结果：

```java
@Component
public class SafeWeatherFunction {
    @Tool(description = "获取指定城市的天气信息")
    public WeatherResponse getWeather(String city) {
        try {
            return weatherApiClient.query(city);
        } catch (TimeoutException e) {
            // 降级：返回缓存中的天气数据
            return getCachedWeather(city);
        } catch (Exception e) {
            log.error("Weather query failed for city: {}", city, e);
            return new WeatherResponse(city, null, "天气服务暂时不可用");
        }
    }
    
    private WeatherResponse getCachedWeather(String city) {
        // 读取本地缓存或返回预设默认值
        return cache.getOrDefault(city, new WeatherResponse(city, 25.0, "celsius", "数据待更新"));
    }
}
```

### 6.3 RAG 构建最佳实践

构建 RAG 应用的全过程分为以下三步：

1. **数据加载与清洗**：从外部知识库加载数据，使用 DocumentReader 读取文档，通过 Splitter 进行分块
2. **向量化存储**：使用 EmbeddingModel 将文本向量化后存储到向量数据库
3. **检索增强**：通过 `QuestionAnswerAdvisor` 为大模型提供上下文信息

#### ⚠️ 依赖说明

Spring AI 1.0+ 在 Starter 模块的 artifact 名称上有重大变化。使用向量存储的正确方式是引入对应的 **starter-vector-store-{database}** 依赖，该 Starter 内部已包含 RAG 所需的所有 Advisor 和 Auto-Configuration。

```xml
<!-- ✅ 正确：使用具体的向量数据库 Starter -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-elasticsearch</artifactId>
</dependency>

<!-- ⚠️ 已废弃：组合式坐标，早期版本存在，当前版本不再推荐 -->
<!-- 
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-advisors-vector-store</artifactId>
</dependency>
-->
```
Spring AI 为 Elasticsearch 向量存储提供了 Spring Boot 自动配置。添加依赖后，可通过 `@Autowired` 注入 `ElasticsearchVectorStore` 进行使用。对于 Spring Boot 3.3.0 之前的版本，需要显式添加 `elasticsearch-java` 依赖。

#### RAG 问答服务配置

```java
@Configuration
public class RagConfig {
    
    @Bean
    public VectorStore vectorStore(EmbeddingModel embeddingModel) {
        return new ElasticsearchVectorStore(elasticsearchClient(), embeddingModel);
    }
    
    @Bean
    public QuestionAnswerAdvisor qaAdvisor(VectorStore vectorStore) {
        return QuestionAnswerAdvisor.builder(vectorStore)
            .searchRequest(SearchRequest.builder()
                .topK(5)
                .similarityThreshold(0.7)
                .build())
            .build();
    }
}
```

### 6.4 生产可观测性（Micrometer + OpenTelemetry）

> ⚠️ **核心原则**：Spring AI 1.0+ 深度集成了 Micrometer Observations，**无需手动埋点**。框架会自动为 ChatClient、ToolCalling、Embedding、VectorStore 等核心组件捕获 metrics、traces 和 logs。

#### 启用自动观测

只需添加观测相关依赖，并在 `application.yml` 中开启观测功能：

```xml
<!-- 观测依赖 -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    observability:
      enabled: true          # 开启自动观测
  zipkin:
    base-url: http://localhost:9411   # 配置 Zipkin 端点
```

启用后，Spring AI Alibaba 自动完成以下观测：
- `ChatClient` 请求/响应的完整追踪
- `ToolCalling` 的入参和出参信息
- `Embedding` 向量化操作的时间分布
- `VectorStore` 检索操作的时间分布

```java
// 无需手动埋点，框架自动收集 metrics
@RestController
public class ChatController {
    private final ChatClient chatClient;  // 框架自动观测此调用
    
    @GetMapping("/ai")
    public String chat(@RequestParam String msg) {
        return chatClient.prompt().user(msg).call().content();
    }
}
```

如需扩展自定义观测指标（如业务相关的计数器），可参考 Spring AI 提供的 `ObservationHandler` 扩展机制。


## 七、代码示例

### 示例一：快速入门——使用 Starter 自动装配

```java
// ========== pom.xml ==========
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-alibaba-starter</artifactId>
    <version>1.0.0.2</version>
</dependency>

// ========== application.yml ==========
spring:
  ai:
    dashscope:
      api-key: ${AI_DASHSCOPE_API_KEY}

// ========== Controller（框架自动注入 ChatClient.Builder）==========
@RestController
public class AiController {
    private final ChatClient chatClient;
    
    public AiController(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("你是一个友好的助手")
            .build();
    }
    
    @GetMapping("/chat")
    public String chat(@RequestParam String message) {
        return chatClient.prompt()
            .user(message)
            .call()
            .content();
    }
}
```

### 示例二：实现自定义工具（查询天气）

```java
// 1. 定义请求/响应实体
public record WeatherResponse(String city, double temperature, String condition) {}

// 2. 实现工具类（内置降级逻辑）
@Component
public class WeatherTools {
    private final Map<String, Double> CACHE = Map.of("杭州", 28.5, "北京", 22.3, "上海", 26.8);
    
    @Tool(description = "获取指定城市的当前天气信息，参数 city 为城市名称")
    public WeatherResponse getWeather(String city) {
        try {
            Double temp = queryApi(city);   // 可能超时
            return new WeatherResponse(city, temp, "晴");
        } catch (Exception e) {
            // 降级：使用缓存数据
            return new WeatherResponse(city, CACHE.getOrDefault(city, 25.0), "数据待更新");
        }
    }
    
    private Double queryApi(String city) {
        // 模拟外部 API 调用
        return CACHE.get(city);
    }
}
```

### 示例三：RAG 流程（标准分块 + 向量存储）

```xml
<!-- 向量数据库 Starter（推荐，包含 RAG 所需的所有组件） -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-elasticsearch</artifactId>
    <version>1.0.0.2</version>
</dependency>
```

```java
@Service
@Slf4j
public class DocumentIngestionService {
    
    @Autowired
    private VectorStore vectorStore;
    @Autowired
    private EmbeddingModel embeddingModel;
    
    /**
     * 加载本地文档目录，使用 TokenTextSplitter 进行标准分块
     */
    public void ingestDocuments(String directoryPath) {
        // 1. 读取文档
        List<File> files = FileUtils.listFiles(new File(directoryPath), 
            new String[]{"txt", "md", "pdf"}, true);
        
        // 2. 使用框架内置的 TokenTextSplitter（避免手写不健壮的分块逻辑）
        TextSplitter splitter = new TokenTextSplitter()
            .withChunkSize(500)
            .withOverlap(100);
        
        for (File file : files) {
            String content = FileUtils.readFileToString(file, StandardCharsets.UTF_8);
            List<Document> chunks = splitter.apply(List.of(new Document(content)));
            
            // 3. 存储到向量数据库
            for (Document chunk : chunks) {
                chunk.getMetadata().put("filename", file.getName());
            }
            vectorStore.add(chunks);
        }
    }
}
```


## 八、从入门到专家自检清单

**概念理解**
1. ChatModel 和 ChatClient 的区别？何时选前者？
2. Spring AI vs Spring AI Alibaba 的核心定位差异是什么？
3. Graph 框架中 Node 和 Edge 的核心职责分别是什么？

**核心原理**
4. 流式响应为什么能降低首块延迟？Reactor Flux 背压的作用？
5. Tool Calling 中 LLM 是否真正执行代码？`@Tool` 注解不支持哪些返回类型？
6. RAG 的「检索→增强→生成」流程中，向量数据库的作用是什么？

**配置与集成**
7. 如何在最小化代码的情况下快速启动一个 AI 项目？（提示：`spring-ai-alibaba-starter` + 自动装配）
8. 如何使用 `spring.ai.chat.options.*` 配置通用模型参数，保证跨模型可移植性？

**排错能力**
9. 遇到 `401 Unauthorized` 错误，按什么顺序排查？
10. 多实例部署时 Agent 状态不一致怎么办？
11. 工具调用在什么情况下会出现 `Flux` 类型不支持错误？（提示：Method as Tools 不支持响应式类型）

**调优能力**
12. 事实问答场景 temperature 应高还是低？为什么不要设为 0？
13. RAG chunk size 过大或过小的影响？为什么建议使用 `TokenTextSplitter` 而非手写分块？
14. 在 Controller 流式返回中，如何正确隔离阻塞操作（如 MySQL 日志），避免 Event Loop 阻塞？

**可观测性**
15. Spring AI Alibaba 的可观测性如何实现？需要手动埋点吗？（提示：基于 Micrometer Observations，**无需手动埋点**，只需开启 `spring.ai.observability.enabled=true`）

**扩展能力**
16. 如何为一个新的模型服务商实现 Spring AI 集成？
17. 如何自定义 Advisor 实现请求前/后处理？
18. 工具调用如无官方 `fallbackMethod`，应如何实现降级？（提示：工具方法内部自行捕获异常并返回降级结果，或通过外置拦截器统一处理）


**参考资源**：
- Spring AI Alibaba 官方文档：https://java2ai.com
- GitHub 源码：https://github.com/alibaba/spring-ai-alibaba
- 阿里云开发者社区系列教程
- Spring AI 工具调用示例：https://github.com/springaialibaba/spring-ai-alibaba-examples/tree/main/spring-ai-alibaba-tool-calling-example
- Spring AI Alibaba 可观测性示例：https://github.com/springaialibaba/spring-ai-alibaba-examples/tree/main/spring-ai-alibaba-observability-example


## 附录：核心修正记录汇总

| 序号 | 问题类别 | 核心痛点 | 修正动作 |
|------|----------|----------|----------|
| 1 | 依赖坐标 | 使用了废弃的组合式坐标 | 修正为 `spring-ai-starter-vector-store-elasticsearch` 等正式 Starter 格式 |
| 2 | 配置属性 | 混用厂商前缀，跨模型不可移植 | 通用配置统一使用 `spring.ai.chat.options.*` 标准前缀 |
| 3 | RAG工程化 | 手写 `splitText` 分块逻辑不健壮 | 补充 `TokenTextSplitter` 框架内置组件的使用说明 |
| 4 | 响应式编程 | 缺少线程池隔离，存在阻塞风险 | 补充 `subscribeOn(Schedulers.boundedElastic())` 完整示例 |
| 5 | 可观测性 | 手动埋点方式陈旧 | 改为 Micrometer Observations 自动观测最佳实践 |
| 6 | 工具调用 | 缺失降级处理方案 | 补充工具内部降级逻辑，说明 `@Tool` 不支持 `fallbackMethod` |
| 7 | 快速入门 | 缺失 Starter 自动装配指南 | 增加独立“快速入门”章节，展示最小化项目配置 |
| 8 | 版本管理 | 版本号表述不完整 | 统一为四位版本号格式 |


## 生产部署核对清单

在将 Spring AI Alibaba 应用部署到生产环境前，请确认以下事项：

1. **配置确认**
   - ✅ 通用模型参数使用 `spring.ai.chat.options.*` 标准前缀
   - ✅ 厂商特有配置（如 API Key）与通用配置分離
   - ✅ API Key 通过环境变量注入，不硬编码

2. **可观测性检查**
   - ✅ 启用 `spring.ai.observability.enabled=true`
   - ✅ 添加 Micrometer Tracing 依赖（micrometer-tracing-bridge-brave + zipkin-reporter-brave）
   - ✅ 配置 Zipkin 或其他 Trace 收集端点

3. **工具调用检查**
   - ✅ 工具方法不返回 `Flux`/`Mono` 等响应式类型（Method as Tools 不支持）
   - ✅ 工具方法内部包含完整的异常捕获和降级逻辑
   - ✅ 使用 `@ToolParam` 为参数提供 `description`，提升调用准确性

4. **流式处理检查**
   - ✅ Controller 返回 `Flux`，使用 `.limitRate()` 设置背压
   - ✅ 阻塞操作（JDBC 日志、文件写入）使用 `.subscribeOn(Schedulers.boundedElastic())` 隔离
   - ✅ 流式响应设置 `.timeout()` 超时保护

5. **RAG 检查**
   - ✅ 使用框架内置 `TokenTextSplitter` 替代手写分块逻辑
   - ✅ 向量存储使用 `spring-ai-starter-vector-store-*` 正式 Starter
   - ✅ 正确设置 `similarityThreshold` 和 `topK`，根据召回效果持续调优

6. **依赖版本检查**
   - ✅ Spring Boot 版本 ≥ 3.4.x
   - ✅ Spring AI Alibaba 版本使用稳定版（如 1.0.0.2）
   - ✅ 使用 Spring AI BOM 进行依赖版本管理