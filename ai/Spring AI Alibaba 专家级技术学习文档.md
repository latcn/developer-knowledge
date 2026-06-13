> **文档定位**：通过费曼学习法（通俗解释）与第一性原理（从本质推导设计）的双重视角，系统掌握 Spring AI Alibaba 的核心原理、生产实践与调优方法。学完本文档后，将具备独立搭建生产级 AI 应用、定位线上问题并对常规场景进行性能调优的能力。

## 一、架构设计

### 1.1 快速入门：最小化项目配置

Spring AI 1.0+ 支持 Spring Boot 3.4.x 和 3.5.x 版本，所有工件均发布在 Maven Central。首先添加 BOM 进行依赖版本管理：

```xml
<!-- pom.xml BOM 配置 -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.0.2</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

> ⚠️ **重要版本变更说明**：Spring AI 1.0 对 artifact ID 的命名模式进行了重大变更。旧命名如 `spring-ai-{model}-spring-boot-starter` 和 `spring-ai-{store}-store-spring-boot-starter`，新规范统一为 `spring-ai-starter-model-{model}` 和 `spring-ai-starter-vector-store-{store}`。

然后添加具体厂商的 Starter：

```xml
<!-- DashScope 模型支持 -->
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-alibaba-starter-dashscope</artifactId>
</dependency>
```

添加依赖后，Spring AI 的自动配置（Auto-Configuration）会自动创建原型级别的 `ChatClient.Builder` bean 供注入使用——这是 Spring AI 自动配置的核心原理。只需配置 API Key 即可：

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
    
    // Spring AI 自动装配的 ChatClient.Builder 会通过构造函数注入
    public ChatController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }
    
    @GetMapping("/ai")
    public String chat(@RequestParam String msg) {
        return chatClient.prompt().user(msg).call().content();
    }
}
```

> 🔬 **自动配置原理**：`AiAutoConfiguration`（或类似命名的自动配置类）在检测到类路径中存在 `ChatModel` 相关依赖时，会自动创建一个 `ChatClient.Builder` 原型 bean。该 bean 已经预配置了从 `application.yml` 读取的模型参数（temperature、maxTokens 等），开发者可直接注入使用。若需要多个 `ChatClient` 实例（例如使用不同模型或不同配置），可通过设置 `spring.ai.chat.client.enabled=false` 禁用自动配置，然后以编程方式创建多个实例。

### 1.2 Spring AI vs Spring AI Alibaba 定位说明

| 维度       | Spring AI                | Spring AI Alibaba                                                                           |
| -------- | ------------------------ | ------------------------------------------------------------------------------------------- |
| **定位**   | AI 应用开发**底层框架**          | AI 智能体**高层框架**                                                                              |
| **提供能力** | 模型适配、工具定义、向量数据库存取等底层原子抽象 | 基于图的智能体编程 Graph 框架（包括 ReactAgent、SequentialAgent、ParallelAgent），让开发者更容易开发工作流、multi-agent 应用 |
| **类比**   | “发动机”——提供 AI 交互的核心能力     | “整车”——提供完整的工作流、Agent 编排能力                                                                   |

> Spring AI 定位 AI 应用开发底层框架，提供 AI 开发需要的底层原子抽象；Spring AI Alibaba 则定位 AI 智能体开发框架，在前者基础上提供了 Graph 智能体编程框架，让开发者更容易开发 ChatBot、工作流、Multi-Agent 应用。

### 1.3 整体分层架构

Spring AI Alibaba 项目从架构上包含如下三层：

| 层级 | 核心能力 | 说明 |
|------|----------|------|
| **Agent Framework** | 以 ReactAgent 为核心的 Agent 开发框架 | 使开发者能够构建具备自动上下文工程和人机交互等核心能力的 Agent |
| **Graph** | 低级别的工作流和多 Agent 协调框架 | Graph 是 Agent Framework 的底层运行时基座 |
| **Augmented LLM** | Spring AI 框架底层原子抽象 | 包括 Model、Tool、MCP、Message、VectorStore 等 |

> 🧠 **费曼解释**：三层架构就像三层餐厅服务体系——最上层是拿着菜单点单的“服务员”（Agent Framework），中间层是协调后厨各工位的“厨房调度”（Graph），最底层则是每个具体烹饪动作的“厨具和基础设备”（Augmented LLM）。

> 🔬 **第一性原理推理**：Graph 是整个 Agent 系统的“地基”——它只关心一件事：确保工作流中的节点按正确顺序执行、状态在节点之间传递。AI 工作流的本质是“有向图上的状态迁移”：每个节点（Node）执行一个原子操作（如调用模型、检索文档、执行工具），边（Edge）定义节点之间的依赖关系与数据流向。


## 二、核心原理与算法

### 2.1 同步/异步与流式处理

Spring AI 定义了两个核心接口：

```java
public interface ChatModel extends Model<Prompt, ChatResponse>, StreamingChatModel {
    ChatResponse call(Prompt prompt);                    // 同步：阻塞直到完整返回
    Flux<ChatResponse> stream(Prompt prompt);            // 异步流式：逐块返回
}
```

流式响应底层使用 Project Reactor 的 `Flux` 实现。

> 🔬 **第一性原理推理**：LLM 生成是一个“从 Token 分布中逐步采样”的过程。同步调用相当于“等全部计算完再一次性发送”，流式调用相当于“每算出一个块就立即发送”。首块延迟取决于模型计算第一个响应块的时间，而总时间不变。这是 I/O 模型与用户体验之间的本质权衡。

### 2.2 Tool Calling（工具调用）

Tool calling（也称为函数调用）是 AI 应用程序中的常见模式，允许模型与一组 API 或工具交互以增强其能力。LLM 本身不能实际调用工具，它只负责决策和参数填充。

#### 2.2.1 工具定义方式

Spring AI 支持两种定义工具的方式：

**1. Method as Tools（声明式，推荐）** ：使用 `@Tool` 注解标记方法

```java
@Component
public class WeatherTools {
    @Tool(description = "获取指定城市的天气信息")
    public WeatherResponse getWeather(String city) {
        // 工具实现，使用 city 参数
        return new WeatherResponse(city, 28.5, "晴");
    }
}
```

**2. Function as Tools（编程式）** ：通过 `FunctionToolCallback` 或 `MethodToolCallback` 显式构建 ToolCallback 对象

```java
@Bean
public ToolCallback customTool() {
    return FunctionToolCallback.builder("customTool", (String input) -> "processed: " + input)
        .description("处理用户输入")
        .inputType(String.class)
        .build();
}
```

> ⚠️ **重要限制**：作为工具的方法，其参数和返回类型**不支持**以下类型：`Optional`、异步类型（`CompletableFuture`、`Future`）、响应式类型（`Flow`、`Mono`、`Flux`）、函数类型（`Function`、`Supplier`、`Consumer`）。

#### 2.2.2 API 变更说明

Spring AI 1.0+ 版本中，工具调用的 API 经历了标准化升级：早期版本的 `FunctionCallback` 接口已被标记为过时，统一更名为 `ToolCallback`（≥1.0.0）。`ChatClient` 中原本用于工具注册的 `defaultFunctions()` 和 `functions()` 方法已被移除，应使用 `toolCallbacks()` 或 `defaultToolCallbacks()` 方法。

```java
// 在 ChatClient 中注册工具的正确方式（1.0+）
String response = ChatClient.create(chatModel)
    .prompt("What day is tomorrow?")
    .tools(new DateTimeTools())          // 使用 .tools() 而非已弃用的 .functions()
    .call()
    .content();
```

#### 2.2.3 工具降级方案

Spring AI 官方的 `@Tool` 注解并未提供 `fallbackMethod` 属性。以下两种方式可实现工具调用的降级：

**方案一：工具内部自行实现降级逻辑**

```java
@Component
public class WeatherTools {
    @Tool(description = "获取指定城市的天气信息")
    public WeatherResponse getWeather(String city) {
        try {
            return weatherApiClient.query(city);
        } catch (TimeoutException e) {
            return getCachedWeather(city);           // 返回缓存数据
        } catch (Exception e) {
            return new WeatherResponse(city, null, "服务暂时不可用");
        }
    }
}
```

**方案二：通过 ToolCallback 包装器实现装饰器模式降级**

通过实现 `ToolCallback` 接口来包装原始方法，在不修改原方法代码的情况下为工具添加统一的降级逻辑：

```java
@Component
public class ResilientToolCallback implements ToolCallback {
    private final ToolCallback delegate;
    
    public ResilientToolCallback(ToolCallback delegate) {
        this.delegate = delegate;
    }
    
    @Override
    public ToolCallResult call(ToolCallRequest request) {
        try {
            return delegate.call(request);
        } catch (Exception e) {
            log.error("Tool execution failed: {}", delegate.getName(), e);
            return new ToolCallResult(request.getId(), request.getName(), "服务暂时不可用，请稍后重试");
        }
    }
}
```

> 🧠 **费曼解释**：LLM 不知道怎么做数学计算，就像一个人不知道怎么修车。但这个人可以：①（在请求时）告诉你“如果遇到修车问题，可以打电话给李师傅”【提供工具定义】；②（遇到问题时）决定“哦，这是个修车问题，我得打给李师傅”【模型决定调用】；③（收到修好后的反馈）“我已经让李师傅修好了”【应用执行工具并回传】。Tool Calling 就是让 LLM 在需要时“打电话找专家”。

### 2.3 RAG 实现原理

#### 2.3.1 RAG 架构

RAG 的本质是“**在推理时引入外部知识**”，分为离线准备和在线问答两个阶段。Spring AI 提供了 ETL（Extract, Transform, Load）管道作为数据处理的脊梁，协调从原始数据源到结构化向量存储的整个流程。

离线阶段：外部文档 → DocumentReader（读取） → DocumentTransformer（分块等转换） → EmbeddingModel（向量化） → VectorStore（存储）。

#### 2.3.2 标准文本分块：TokenTextSplitter

```java
// 使用 TokenTextSplitter 进行标准分块
@Autowired
private EmbeddingModel embeddingModel;
@Autowired
private VectorStore vectorStore;

public void ingestDocument(Resource pdfResource) {
    // 1. 读取 PDF 文档
    PagePdfDocumentReader pdfReader = new PagePdfDocumentReader(pdfResource);
    List<Document> documents = pdfReader.get();
    
    // 2. 使用 TokenTextSplitter 进行标准分块（基于 Token 计数）
    TokenTextSplitter splitter = new TokenTextSplitter();
    List<Document> chunks = splitter.apply(documents);
    
    // 3. 生成 Embedding 并存储
    vectorStore.accept(chunks);
}
```

TokenTextSplitter 是 TextSplitter 的一个实现，采用 CL100K_BASE 编码，根据 Token 数量将文本分割成块，并向前查找最近的英文分隔符（句号、问号、叹号、换行符）以保证语义完整性。主要参数包括：

- `defaultChunkSize`：每个文本块的目标 Token 数量，默认 800
- `minChunkSizeChars`：每个文本块的最小字符数，默认 350
- `keepSeparator`：是否保留分隔符，默认 true

#### 2.3.3 Advisor 链与 RAG 增强

配置 RAG Advisor 的标准方式：

```java
var qaAdvisor = QuestionAnswerAdvisor.builder(vectorStore)
    .searchRequest(SearchRequest.builder()
        .similarityThreshold(0.8d).topK(6).build())
    .build();

ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(qaAdvisor)
    .build();
```

> 🔬 **Advisor 链执行原理**：框架从用户的 Prompt 创建一个 `AdvisedRequest`，同时创建一个空的 `AdvisorContext` 对象。链中的每个 advisor 处理请求，可能会修改它或选择不继续。框架提供的最终 advisor 将请求发送给 Chat Model，响应再通过 advisor 链传回。`QuestionAnswerAdvisor` 在此过程中执行向量检索并将检索结果增强到 Prompt 上下文中。

### 2.4 对话记忆的实现

`ChatMemory` 必须通过 `MessageChatMemoryAdvisor` 注册到 `ChatClient` 的 advisor 链中：

```java
ChatMemory chatMemory = new InMemoryChatMemory();
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
    .build();
```

默认情况下，Spring AI 自动配置单个 `ChatClient.Builder` bean。如果需要多个 ChatClient 实例（如使用不同模型），可通过设置 `spring.ai.chat.client.enabled=false` 禁用自动配置，然后手动创建。


## 三、主要功能

### 3.1 核心能力矩阵

| 功能类别 | 核心实现 | 典型场景 |
|----------|----------|----------|
| 同步/异步/流式对话 | `ChatModel.call()` / `ChatClient.stream()` | 实时聊天、文本生成 |
| 结构化输出 | `ChatClient.entity(Class.class)` | 报表提取、信息抽取 |
| Tool Calling | `@Tool` / `ToolCallback` | 数学计算、外部 API 调用 |
| RAG 文档问答 | `QuestionAnswerAdvisor` + VectorStore | 企业知识库问答 |
| Agent 多智能体 | ReactAgent / SequentialAgent / ParallelAgent | 复杂任务自动规划 |
| Graph 工作流 | StateGraph API | 复杂业务流程编排 |

> 🧠 **费曼解释**：主要功能可以理解为“给 AI 添加各种外挂”：结构化输出 = 强迫 AI 填表格；RAG = 给 AI 开卷考试；Agent = 让 AI 自己规划怎么做；Tool Calling = 让 AI 能使用计算器等工具。


## 四、常见生产问题分类与解决办法

### 4.1 工具调用 API 版本问题

| 问题类型 | 典型现象 | 深层原因 | 解决方案 |
|----------|----------|----------|----------|
| 工具调用静默失败 | LLM 响应中无工具调用行为，但代码看似正确 | 从 Spring AI 1.0 M7 升级到 M8 时，旧版 `tools()` 方法的内部实现发生了破坏性变更 | 确保使用正确的工具注册方式（`.tools()` 而非已弃用的 `.functions()`），并检查 Spring AI 版本对应的 API 变更说明 |
| `FunctionCallback` 找不到 | 编译错误或运行时找不到类 | `FunctionCallback` 在 1.0+ 版本中已被标记为 `@Deprecated`，推荐使用 `ToolCallback` | 将 `FunctionCallback` 迁移为 `ToolCallback`；对于 `Function` 接口类型的工具，使用 `FunctionToolCallback` 包装；对于普通方法，使用 `@Tool` 注解配合 `.tools()` 注册 |
| 异步/响应式类型返回异常 | 工具方法返回 `Mono<T>` 或 `Flux<T>` 导致序列化错误 | `ToolCallback` 期望的是一个可序列化的具体对象，而响应式类型无法被直接序列化 | 工具方法的返回类型不能使用响应式类型，需改为同步返回 `T` |

### 4.2 网络与限流类

| 问题类型 | 解决方案 |
|----------|----------|
| 连接超时 | 检查网络安全组/防火墙规则；验证 API Key 和端点配置 |
| 401 Unauthorized | 按顺序排查：`application.yml` → 环境变量 → 配置文件覆盖优先级 |
| 429 Too Many Requests | 实现客户端限流器；实现延迟重试 + 请求排队策略 |

### 4.3 解析与转换类

| 问题类型 | 解决方案 |
|----------|----------|
| 工具调用参数 JSON 解析失败 | 在 `@Tool` description 中明确参数 JSON 格式；检查是否使用了不支持的类型 |
| 结构化输出映射失败 | 使用 `ParameterizedTypeReference` 处理泛型集合；先获取原始响应进行验证 |


## 五、调优

### 5.1 模型参数调优

```yaml
# 通用配置使用标准前缀，保证跨模型可移植性
spring:
  ai:
    chat:
      options:
        temperature: 0.7
        max-tokens: 1000
    dashscope:
      api-key: ${AI_DASHSCOPE_API_KEY}
```

| 参数            | 事实性任务            | 创意任务      | 说明              |
| ------------- | ---------------- | --------- | --------------- |
| `temperature` | 0.01~0.1（不要设为 0） | 0.7~0.9   | 事实性任务低温度确保确定性输出 |
| `topP`        | 0.9~1.0          | 0.9~1.0   | 通常保持 0.9 即可     |
| `maxTokens`   | 500~1000         | 1500~4000 | 根据场景调整          |

> 🔬 **第一性原理推理**：`temperature` 不要设为 0——当概率分布过于极端时，模型可能陷入“死循环”，反复输出相同的 Token 序列。

### 5.2 RAG 分块策略

| 文档类型 | chunk 大小 | overlap | 说明 |
|----------|------------|---------|------|
| 技术文档 | 256~512 tokens | 10~20% | 使用 `TokenTextSplitter`，避免手写分块 |
| 法律/政策 | 512~1024 tokens | 15~25% | 条款间关联紧密 |
| 对话记录 | 128~256 tokens | 5~10% | 每轮对话短，小块更精准 |

### 5.3 向量检索调优

| 参数 | 推荐值 | 调优方向 |
|------|--------|----------|
| `similarityThreshold` | 0.7 | 噪音多 → 提高到 0.8；召回少 → 降低到 0.6 |
| `topK` | 3~5 | 过多会挤占 Token 限额 |

### 5.4 流式输出线程隔离

⚠️ **生产级流式输出线程隔离**（防止 Event Loop 阻塞）：

```java
@RestController
public class StreamingController {
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamChat(@RequestParam String query) {
        Flux<String> aiResponse = chatClient.prompt().user(query).stream().content();
        
        // 阻塞操作（如 MySQL 日志）必须隔离到 boundedElastic
        aiResponse = aiResponse.flatMap(chunk -> 
            Mono.fromRunnable(() -> saveAuditLog(chunk))
                .subscribeOn(Schedulers.boundedElastic())
                .thenReturn(chunk)
        );
        
        return aiResponse;
    }
}
```


## 六、最佳实践

### 6.1 结构化输出设计模式

```java
// ✅ 推荐：使用 Record + Bean Validation + 默认值兜底
public record CustomerOrder(
    @NotBlank String customerName,
    @Min(1) @Max(999999) Long orderId,
    @Default("PENDING") String status
) {}
```

### 6.2 可观测性配置

要启用可观测性，需要 `spring-boot-actuator` 模块：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Spring AI 为其核心组件（ChatClient、ChatModel、EmbeddingModel、ImageModel 和 VectorStore）提供指标和追踪能力。配置观测属性（使用正确的配置前缀）：

```yaml
spring:
  ai:
    chat:
      client:
        observations:
          log-prompt: true       # 记录提示内容（调试用）
          log-completion: true   # 记录完成内容（调试用）
```

> ⚠️ **配置前缀说明**：1.0.0-RC1 版本中，配置属性进行了重命名——`spring.ai.chat.client.observations.include-prompt` → `spring.ai.chat.client.observations.log-prompt`，`spring.ai.chat.observations.include-prompt` → `spring.ai.chat.observations.log-prompt`，`spring.ai.vectorstore.observations.include-query-response` → `spring.ai.vectorstore.observations.log-query-response`。提示和完成数据通常较大且可能包含敏感信息，默认不导出，通过 `log-prompt` 和 `log-completion` 属性可选开启。

### 6.3 流式响应资源管理

流式响应中，必须确保 `Flux` 被正确管理，避免内存泄漏：

```java
// ✅ 正确：返回 Flux 让 WebFlux 自动管理生命周期
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> stream() {
    return chatClient.prompt().user("Tell me a story").stream().content()
        .timeout(Duration.ofSeconds(30))
        .doOnTerminate(() -> log.info("Stream completed"));
}

// ❌ 错误：手动订阅而不释放
public void badStream() {
    chatClient.prompt().user("Hello").stream().content()
        .subscribe(chunk -> process(chunk));  // 危险：订阅未管理，可能内存泄漏
}
```

### 6.4 ChatClient.Builder 自动配置说明

默认情况下，Spring AI 自动配置单个 `ChatClient.Builder` bean。在多 ChatClient 场景中，可以通过设置 `spring.ai.chat.client.enabled=false` 禁用自动配置，然后以编程方式手动创建多个 `ChatClient` 实例。手动创建时，每个 `ChatClient` 可以拥有独立的配置（如系统提示、工具列表、Advisor 链），适用于为不同类型任务使用不同模型、实现回退机制、A/B 测试等场景。

### 6.5 依赖坐标速查表

| 使用场景 | 正确依赖坐标（1.0+ 标准格式） |
|----------|------------------------------|
| DashScope 模型支持 | `<groupId>com.alibaba.cloud.ai</groupId><artifactId>spring-ai-alibaba-starter-dashscope</artifactId>` |
| 通用 Spring AI BOM | `<groupId>org.springframework.ai</groupId><artifactId>spring-ai-bom</artifactId>` |
| OpenAI 模型 Starter（参考格式） | `<groupId>org.springframework.ai</groupId><artifactId>spring-ai-starter-model-openai</artifactId>` |
| Elasticsearch 向量存储（参考格式） | `<groupId>org.springframework.ai</groupId><artifactId>spring-ai-starter-vector-store-elasticsearch</artifactId>` |


## 七、代码示例

### 示例一：快速入门

```java
@RestController
public class AiController {
    private final ChatClient chatClient;
    
    public AiController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }
    
    @GetMapping("/chat")
    public String chat(@RequestParam String message) {
        return chatClient.prompt().user(message).call().content();
    }
}
```

### 示例二：带降级的工具调用

```java
@Component
public class WeatherTools {
    private final Map<String, Double> CACHE = Map.of("杭州", 28.5, "北京", 22.3);
    
    @Tool(description = "获取指定城市的天气信息")
    public WeatherResponse getWeather(String city) {
        try {
            return apiClient.query(city);      // 可能异常
        } catch (TimeoutException e) {
            return new WeatherResponse(city, CACHE.getOrDefault(city, 25.0), "缓存数据");
        } catch (Exception e) {
            return new WeatherResponse(city, null, "服务暂时不可用");
        }
    }
}
```

### 示例三：RAG 流程（标准实现）

```xml
<!-- 使用正确的向量存储 Starter -->
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-alibaba-starter-dashscope</artifactId>
</dependency>
```

```java
@Service
public class RagService {
    @Autowired private VectorStore vectorStore;
    @Autowired private EmbeddingModel embeddingModel;
    
    public void ingest(Resource pdfResource) {
        PagePdfDocumentReader reader = new PagePdfDocumentReader(pdfResource);
        List<Document> docs = reader.get();
        TokenTextSplitter splitter = new TokenTextSplitter();  // 标准分块器
        List<Document> chunks = splitter.apply(docs);
        vectorStore.accept(chunks);                            // ETL 流式风格
    }
    
    public String ask(String question) {
        var advisor = QuestionAnswerAdvisor.builder(vectorStore)
            .searchRequest(SearchRequest.builder().topK(5).similarityThreshold(0.7).build())
            .build();
        return ChatClient.create(chatModel).prompt()
            .user(question).advisors(advisor).call().content();
    }
}
```


## 八、从入门到专家自检清单

**概念理解**
1. Spring AI 的 Auto-Configuration 自动配置机制是什么？`ChatClient.Builder` bean 是如何注入的？
2. Spring AI vs Spring AI Alibaba 的核心定位差异是什么？（提示：前者是底层框架，后者是智能体开发框架）

**核心原理**
3. Tool Calling 中 LLM 是否真正执行代码？Method as Tools 不支持哪些返回类型？（提示：不支持响应式类型）
4. `FunctionCallback` 和 `ToolCallback` 是什么关系？1.0 版本中哪些 API 被弃用了？
5. RAG 的 ETL 流程中 `TokenTextSplitter`、`DocumentReader`、`VectorStore` 各自扮演什么角色？

**配置与集成**
6. 1.0 版本中 artifact ID 的命名规范是什么？（提示：`spring-ai-starter-model-{model}` 格式）
7. 如何使用可观测性功能？需要添加什么依赖？`log-prompt` 和 `log-completion` 的作用是什么？

**排错能力**
8. 从 1.0 M7 升级到 M8 时工具调用静默失败的原因和解决方案？
9. 多实例部署时 Agent 状态不一致怎么办？

**调优能力**
10. RAG 中使用 `TokenTextSplitter` 而非手写分块的优势是什么？其核心参数 `defaultChunkSize` 和 `keepSeparator` 的含义是什么？
11. `temperature` 为什么不要设为 0？

**扩展能力**
12. 如何通过 ToolCallback 包装器实现非侵入式降级？（提示：装饰器模式 + 组合而非继承）
13. 如何手动创建多个 `ChatClient` 实例？什么情况下需要禁用 `ChatClient.Builder` 自动配置？


**参考资源**：
- Spring AI Alibaba 官方文档：https://java2ai.com
- GitHub 源码：https://github.com/alibaba/spring-ai-alibaba
- Spring AI 官方参考文档：https://docs.spring.io/spring-ai/reference
- Spring AI 1.0 升级指南：https://docs.spring.io/spring-ai/reference/upgrade-notes.html
- Spring AI Tool Calling 示例：https://github.com/springaialibaba/spring-ai-alibaba-examples


## 九、附录
### 问题 1：Spring AI 的 Auto-Configuration 自动配置机制是什么？`ChatClient.Builder` Bean 是如何注入的？

**答案**：Spring AI 的自动配置遵循 Spring Boot 的核心设计理念：面向接口编程 + 条件化配置。
1.  **机制原理**：当项目添加了 `spring-ai-starter-model-*`（如 `spring-ai-starter-model-openai`）后，Spring Boot 启动时，其 `AutoConfiguration` 类会扫描 classpath。一旦检测到相关依赖存在，它就会根据 `application.yml` 中的配置（如 `spring.ai.chat.options.temperature`），在 IoC 容器中自动创建一个 `ChatModel` 的实现类 Bean 和一个原型级别的 `ChatClient.Builder` Bean。
2.  **注入方式**：开发者可在代码中直接通过构造函数注入 `ChatClient.Builder`。该 Builder 已预置了从配置文件读取的通用参数，开发者只需调用 `builder.build()` 即可创建 `ChatClient` 实例。

### 问题 2：Spring AI vs Spring AI Alibaba 的核心定位差异是什么？

**答案**：两者是底层与高层的关系，Spring AI Alibaba 是构建在 Spring AI 之上的企业级智能体框架。
*   **Spring AI**：定位为 **AI 应用开发底层框架**，提供模型适配、工具定义、向量数据库存取等基础的原子抽象。它更像一个规范，定义了如何与各种 AI 模型进行标准化的交互。
*   **Spring AI Alibaba**：定位为 **AI 智能体开发框架**。它以 Spring AI 为基础，**深度集成了阿里云百炼平台**，并提供了基于图的智能体编程框架——Graph。这让开发者能更方便地开发工作流和多智能体应用，并将底层复杂的流程编排和状态管理封装起来。

简单来说，Spring AI 提供的是“发动机”，而 Spring AI Alibaba 提供了一辆完整的“整车”。

### 问题 3：Tool Calling 中 LLM 是否真正执行代码？Method as Tools 不支持哪些返回类型？

**答案**：
1.  **LLM 不执行代码**：LLM 在 Tool Calling 过程中**不直接执行任何业务代码**。它的作用是输出一个符合特定 JSON Schema 的工具调用指令，这个指令包含了要调用的工具名称和参数。实际的代码执行是由 Spring AI 框架在本地完成的。
2.  **不支持的返回类型**：使用 `@Tool` 注解的方式（Method as Tools）**不**支持以下返回类型：
    *   `Optional`
    *   异步类型：`CompletableFuture`、`Future`
    *   响应式类型：`Flow`、`Mono`、`Flux` (这意味着工具方法不能返回 `Mono<T>` 或 `Flux<T>`)
    *   函数类型：`Function`、`Supplier`、`Consumer`

### 问题 4：`FunctionCallback` 和 `ToolCallback` 是什么关系？1.0 版本中哪些 API 被弃用了？

**答案**：
1.  **名称统一**：`ToolCallback` 是 `FunctionCallback` 的**标准化替代品**。在 Spring AI 1.0+ 版本中，`FunctionCallback` 接口及其相关的 `FunctionCallbackWrapper` 已被标记为 `@Deprecated`，推荐统一使用 `ToolCallback` 及其具体实现（如 `MethodToolCallback`、`FunctionToolCallback`）。
2.  **API 变更**：
    *   **工具注册**：`ChatClient` 中用于注册工具的 `defaultFunctions()` 和 `.functions()` 方法已被移除，应使用 `defaultToolCallbacks()` 和 `.tools()` 方法。
    *   **坐标变更**：Spring AI 启动器 artifact 的命名模式已发生变更。例如，模型启动器由 `spring-ai-{model}-spring-boot-starter` 迁移为 `spring-ai-starter-model-{model}`。向量存储启动器的命名模式也类似。

### 问题 5：RAG 的 ETL 流程中 `TokenTextSplitter`、`DocumentReader`、`VectorStore` 各自扮演什么角色？

**答案**：这三个组件共同构成了 Spring AI 中 RAG 能力的 ETL（提取、转换、加载）管道，协调从原始数据到向量存储的整个流程。
*   **`DocumentReader`（提取）**：负责从各种数据源（如 PDF、Markdown、JSON 文件）中**读取和解析**原始数据，并将其转换为框架能统一处理的 `Document` 对象。
*   **`TokenTextSplitter`（转换）**：属于 `DocumentTransformer`，是 **ETL 中的转换（Transform）环节**。它将长文档按**Token 数量**智能地切割成更小的文本块（Chunk）。
*   **`VectorStore`（加载）**：是 **ETL 中的加载（Load）环节**。它接收经过 `TokenTextSplitter` 处理后的文本块，**负责将文本块及其对应的向量表示高效地存储到向量数据库中**，以便后续进行快速的相似性检索。

### 问题 6：1.0 版本中 artifact ID 的命名规范是什么？

**答案**：
1.  **统一前缀**：所有 Starter 的 artifact ID 均以 `spring-ai-starter-` 开头。
2.  **模型 Starter**：格式为 `spring-ai-starter-model-{模型名称}`。例如：`spring-ai-starter-model-openai`、`spring-ai-starter-model-ollama`。
3.  **向量存储 Starter**：格式为 `spring-ai-starter-vector-store-{数据库名称}`。例如：`spring-ai-starter-vector-store-elasticsearch`、`spring-ai-starter-vector-store-redis`。

### 问题 7：如何使用可观测性功能？`log-prompt` 和 `log-completion` 的作用是什么？

**答案**：
1.  **激活方式**：在项目中添加必要的依赖后，Spring AI 会自动为 `ChatClient`、`ToolCalling`、`EmbeddingModel`、`VectorStore` 等核心组件提供指标和追踪功能。
2.  **`log-prompt` 与 `log-completion`**：
    *   **作用**：这两个配置属性**仅用于调试和问题排查**。`log-prompt` 用于在日志中**记录发送给模型的提示内容**，`log-completion` 用于记录模型**返回的完成内容**。
    *   **配置**：它们属于 `Observations` 配置的一部分。示例配置如下：
        ```yaml
        spring:
          ai:
            chat:
              client:
                observations:
                  log-prompt: true       # 记录提示内容
                  log-completion: true   # 记录完成内容
        ```

### 问题 8：从 1.0 M7 升级到 M8 时工具调用静默失败的原因和解决方案？

**答案**：
1.  **根本原因**：这是从 Spring AI 1.0 M7 升级到 M8 时遇到的**破坏性变更**。核心问题是`tools()` 方法的内部实现方式发生了变化，导致之前注册工具回调的代码会遭遇静默失败。
2.  **解决方案**：升级后，开发者需要**使用推荐的替代方案**，即使用新的 `ToolCallback` API 和 `tools()` 方法。具体步骤通常包括：
    *   使用 `@Tool` 注解定义工具，并通过 `tools()` 方法进行注册。
    *   或者，采用编程式方式，通过实现 `ToolCallback` 接口来构建更复杂的工具，并同样使用 `tools()` 注册。

### 问题 9：多实例部署时 Agent 状态不一致怎么办？

**答案**：根本原因是使用了基于内存的 `InMemoryChatMemory`，它无法在多实例间共享状态。解决方案是**使用支持分布式的 `ChatMemory` 实现**，例如：
1.  **使用 Redis**：推荐实现 `RedisSaver` 或 `RedisChatMemory`，将 Agent 状态和会话历史持久化到 Redis。这种方式不仅支持多实例共享，还能在服务重启后保持会话状态。
2.  **使用数据库**：可以基于 JDBC 实现 `JdbcChatMemoryRepository`，将状态存储在关系型数据库中。

### 问题 10：RAG 中使用 `TokenTextSplitter` 而非手写分块的优势是什么？其核心参数的含义是什么？

**答案**：
1.  **优势**：手写分块逻辑（如按字符数分割）容易切断单词甚至代码，破坏语义完整性。而 `TokenTextSplitter` 基于 OpenAI 推荐的 CL100K_BASE 编码方案进行**Token 级别**的分割。
2.  **核心参数**：
    *   `defaultChunkSize`（默认 800）：每个文本块的**目标 Token 数量**。
    *   `minChunkSizeChars`（默认 350）：每个文本块的**最小字符数**。
    *   `keepSeparator`（默认 true）：**是否在生成的块中保留分隔符**（如句号、换行符等），这有助于保持文本的自然结构和可读性。

### 问题 11：`temperature` 为什么不要设为 0？

**答案**：理论上是允许设为 0 的，但在实践中，`temperature` 设得**过低（接近 0）会导致模型输出高度确定**，这可能带来以下问题：
*   **陷入重复循环**：模型可能会在某个 token 概率极高时反复选择它，导致输出变成无意义的**文本或词语的重复**，无法继续生成新的内容，这在技术上被称为“死循环”或“输出崩塌”。
*   **缺乏创造力**：`temperature` 为 0 时，模型总是选择概率最高的 token，这使得它无法应对任何需要一点变化或创造性的任务，输出显得非常机械和死板。

因此，即使对于事实性任务，一个极低但不为零的 `temperature`（例如 0.01）通常是更安全的选择。

### 问题 12：如何通过 `ToolCallback` 包装器实现非侵入式降级？

**答案**：可以实现 `ToolCallback` 接口的**装饰器模式（Decorator Pattern）**，通过组合而非继承的方式，在无需修改原工具代码的情况下，为工具添加统一的降级、重试或缓存逻辑。
**示例代码**：
```java
// 1. 定义一个通用的降级装饰器
public class ResilientToolCallback implements ToolCallback {
    private final ToolCallback delegate;
    
    public ResilientToolCallback(ToolCallback delegate) {
        this.delegate = delegate;
    }
    
    @Override
    public ToolCallResult call(ToolCallRequest request) {
        try {
            return delegate.call(request);
        } catch (Exception e) {
            log.error("Tool '{}' execution failed.", delegate.getName(), e);
            // 降级返回一个友好的错误消息
            return new ToolCallResult(request.getId(), request.getName(), 
                                      "服务暂时不可用，请稍后重试。");
        }
    }
}

// 2. 在创建 ChatClient 时，用装饰器包裹原始工具回调
ToolCallback originalTool = MethodToolCallback.builder(new MyService()).build();
ToolCallback resilientTool = new ResilientToolCallback(originalTool);

ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultToolCallbacks(resilientTool)
    .build();
```

### 问题 13：如何手动创建多个 `ChatClient` 实例？什么情况下需要禁用 `ChatClient.Builder` 自动配置？

**答案**：
1.  **手动创建**：当需要为不同类型任务配置完全独立的 `ChatClient` 实例时（如一个用于客服，一个用于创意写作），可直接使用 `new ChatClient.Builder(chatModel)` 来手动构建。
2.  **禁用自动配置**：当项目中存在**多个 `ChatModel` 类型的 Bean**时，Spring AI 的 `ChatClientAutoConfiguration` 会因无法确定注入哪一个而产生歧义，导致 `ChatClient.Builder` 的自动装配失败。此时，可通过设置 `spring.ai.chat.client.enabled=false` 来禁用其自动配置，之后所有 `ChatClient` 实例都需手动创建。