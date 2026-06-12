
> **文档定位**：通过费曼学习法（通俗解释）与第一性原理（从本质推导设计）的双重视角，系统掌握 Spring AI Alibaba 的核心原理、生产实践与调优方法。学完本文档后，将具备独立搭建生产级 AI 应用、定位线上问题并对常规场景进行性能调优的能力。

## 一、架构设计

### 1.1 整体分层架构

Spring AI Alibaba 项目从架构上包含如下三层：

| 层级 | 核心能力 | 说明 |
|------|----------|------|
| **Agent Framework** | 以 ReactAgent 为核心的 Agent 开发框架 | 使开发者能够构建具备自动上下文工程和人机交互等核心能力的 Agent |
| **Graph** | 低级别的工作流和多 Agent 协调框架 | Graph 是 Agent Framework 的底层运行时基座，具备丰富的预置节点和简化的图状态定义 |
| **Augmented LLM** | Spring AI 框架底层原子抽象 | 包括 Model、Tool、MCP、Message、VectorStore 等 |

> 🧠 **费曼解释**：三层架构就像一个三层餐厅服务体系：最上层是拿着菜单给你点单的“服务员”（Agent Framework），中间层是协调后厨各工位按正确顺序做菜的“厨房调度”（Graph），最底层则是负责每个具体烹饪动作的“厨具和基础设备”（Augmented LLM）。

> 🔬 **第一性原理推理**：Graph 是整个 Agent 系统的“地基”——它只关心一件事：确保工作流中的节点按正确顺序执行、状态在节点之间传递。AI 工作流的本质是“有向图上的状态迁移”：每个节点（Node）执行一个原子操作（如调用模型、检索文档、执行工具），边（Edge）定义节点之间的依赖关系与数据流向。ReactAgent 建立在这个地基之上，开发者可以直接使用预制能力而无需自己砌砖。

Spring AI Alibaba Graph 是社区核心实现之一，也是整个框架在设计理念上区别于 Spring AI 只做底层原子抽象的地方——Spring AI Alibaba 期望帮助开发者更容易地构建智能体应用。

### 1.2 Spring AI vs Spring AI Alibaba 定位说明

| 维度 | Spring AI | Spring AI Alibaba |
|------|-----------|-------------------|
| **定位** | AI 应用开发**底层框架** | AI 智能体**高层框架** |
| **提供能力** | 模型适配、工具定义、向量数据库存取等底层原子抽象 | 基于图算法的智能体编程 Graph 框架，让开发者更容易开发工作流、multi-agent 应用 |
| **类比** | “发动机”——提供 AI 交互的核心能力 | “整车”——提供完整的工作流、Agent 编排能力 |

Spring AI Alibaba 是一款以 Spring AI 为基础，深度集成百炼平台，支持 ChatBot、工作流、多智能体应用开发模式的 AI 框架。

### 1.3 核心组件与交互流程

#### 组件全景图

```
┌─────────────────────────────────────────────────────────────────┐
│                        应用层（Application）                      │
│   ChatClient（Fluent API）  │  ReactAgent  │  Workflow/Graph    │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                     编排层（Orchestration）                       │
│   Advisor（RAG / 安全）  │  ChatMemory（对话记忆）  │  Function    │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                     执行层（Execution）                          │
│   ChatModel  │  EmbeddingModel  │  VectorStore  │  OutputParser │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                     基础设施层（Infrastructure）                  │
│   通义千问 / 百炼平台  │  Redis / ES  │  阿里云基础设施          │
└─────────────────────────────────────────────────────────────────┘
```

#### 核心组件详解

**ChatModel vs ChatClient**

| 维度 | ChatModel | ChatClient |
|------|-----------|------------|
| 定位 | 底层原子接口，直接封装模型通信 | 高层 Fluent API，构建于 ChatModel 之上 |
| 特点 | 功能单一、灵活度高、需写大量样板代码 | 链式调用、内置系统提示、模板、工具调用 |
| 类比 | “手动挡汽车”——精准控制但操作繁琐 | “智能点餐机”——集成丰富功能一键完成 |

Spring AI Chat Model API 设计为一个简单且可移植的接口，用于与各种 AI 模型交互，允许开发人员以最小的代码更改在不同模型之间切换。

**交互流程**：
```
用户请求 → ChatClient.prompt().user(msg) → 
Advisor链（RAG检索/安全过滤）→ ChatModel.call(Prompt) → 
LLM → ChatResponse解析 → 函数调用（可选）→ 返回结果
```


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

> 📖 **术语说明**：该能力在官方最新文档中正式名称为 **Tool Calling**。在早期版本中也被称为“Function Calling”，本文档为便于与历史资料保持对照，将继续以“函数调用”为例进行解释，请读者知晓两种命名的等价关系。

#### 工作流程

Tool Calling 允许大型语言模型（LLM）在必要时调用一个或多个可用的工具，这些工具通常由开发者定义。LLM 本身不能实际调用工具——它只负责决策和参数填充。

**Step 1**：用户代码定义工具方法并通过 `@Tool` 注解暴露给模型：
```java
@Component
public class WeatherTools {
    @Tool(description = "获取指定城市的天气信息")
    public WeatherResponse getWeather(String city) {
        // 实际的天气查询逻辑
        return new WeatherResponse(city, 28.5, "晴");
    }
}
```

**Step 2**：请求中包含工具定义，LLM 决定是否调用。

**Step 3**：LLM 返回工具调用请求（非纯文本，而是包含 `name`、`arguments` 的结构化指令）。

**Step 4**：框架执行 Java 工具方法并将结果回传给 LLM 生成最终回复。

> 🧠 **费曼解释**：LLM 不知道怎么做数学计算，就像一个人不知道怎么修车。但这个人可以：①（在请求时）告诉你“如果遇到修车问题，可以打电话给李师傅”【提供工具定义】；②（遇到问题时）决定“哦，这是个修车问题，我得打给李师傅”【模型决定调用】；③（收到修好后的反馈）“我已经让李师傅修好了，这是结果”【应用执行工具并回传】。Tool Calling 就是让 LLM 在需要时“打电话找专家”，而不是自己硬答。

#### 关键类型

Spring AI 支持两种定义工具的方式：**Method as Tools**（通过 `@Tool` 注解）和 **Function as Tools**。

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

// 通过 ChatClient 使用 RAG
ChatResponse response = ChatClient.builder(chatModel)
    .build()
    .prompt()
    .advisors(new QuestionAnswerAdvisor(vectorStore))
    .user(userText)
    .call()
    .chatResponse();
```

> 🔬 **第一性原理推理**：RAG 的本质是“**在推理时引入外部知识**”。LLM 的原始训练数据被冻结在某个时间点，无法覆盖私有数据或最新信息。RAG 将“检索”和“生成”分离：检索负责从外部知识库中找到相关信息，生成负责基于检索结果回答问题。这种设计的核心优势在于：知识更新成本低（只需更新向量数据库，无需重新训练模型），且可解释性强（可以直接看到检索到的来源）。

### 2.4 对话记忆的实现

LLM 的 API 是无状态的，每次调用都是独立的对话回合。Spring AI Alibaba 通过 **`ChatMemory`** 接口解决多轮对话的上下文连续性。

`ChatMemory` 抽象旨在管理聊天内存，允许存储和检索与当前对话上下文相关的消息。

#### 实现方式

**内存版（适合开发/演示）** ：
```java
ChatMemory chatMemory = new InMemoryChatMemory();  // 需确保作为单例存在
```

**生产级实现（JdbcChatMemoryRepository）** ：
```java
// Spring AI 和 Spring AI Alibaba 均提供了基于 JDBC 的聊天记忆存储实现
JdbcChatMemoryRepository repository = new JdbcChatMemoryRepository(dataSource);
```

**对话记忆与 Advisor 集成**：
`ChatMemory` 必须通过 `MessageChatMemoryAdvisor` 注册到 `ChatClient` 的 advisor 链中，才能在多轮对话中自动管理历史记录。

```java
ChatMemory chatMemory = new InMemoryChatMemory();   // 或 Redis/JDBC 版本
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(new MessageChatMemoryAdvisor(chatMemory))
    .build();
```

> 🧠 **费曼解释**：`threadId` 就像对话的房间号。每次用户说话，系统根据房间号从存储里取出之前的所有聊天记录，连同当前问题一起发给 LLM。这样 LLM 就知道“刚才我们聊过什么”。使用 Redis 或数据库可以跨服务器实例共享状态。正确的生命周期管理和单例模式是保证聊天记忆功能正常工作的关键。

## 三、主要功能

### 3.1 核心能力矩阵

Spring AI Alibaba 提供以下核心能力，让开发者可以快速构建自己的 Agent、Workflow 或 Multi-Agent 应用：

| 功能类别 | 核心实现 | 典型场景 |
|----------|----------|----------|
| 同步/异步/流式对话 | `ChatModel.call()` / `ChatClient.stream()` | 实时聊天、文本生成 |
| 结构化输出 | `ChatClient.entity(Class.class)` | 报表提取、信息抽取、数据整理 |
| 函数调用（Tool Calling） | `@Tool` 注解 + 工具方法 | 数学计算、外部 API 调用、搜索 |
| RAG 文档问答 | `QuestionAnswerAdvisor` + VectorStore | 企业知识库、智能客服 |
| Agent 多智能体 | ReactAgent / SequentialAgent / ParallelAgent | 复杂任务自动规划执行 |
| 多模态输入 | `UserMessage` 集成 `MediaContent` | 图文问答、图像理解 |
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

### 3.3 Graph 工作流编排

Spring AI Alibaba Graph 的核心概念包括：

- **StateGraph（状态图）** ：用于定义节点和边
- **Node（节点）** ：封装具体操作或模型调用
- **Edge（边）** ：表示节点间的跳转关系
- **OverAllState（全局状态）** ：贯穿流程共享数据

在 Spring AI Alibaba Graph 中，Node 是工作流的基本执行单元。每个 Node 负责处理特定的业务逻辑，接收状态（State）作为输入，并返回更新后的状态。

```java
// Graph 工作流构建示例
StateGraph graph = StateGraph.builder()
    .addNode("retrieve", retrieveNode)
    .addNode("generate", generateNode)
    .addEdge("retrieve", "generate")
    .build();
CompiledGraph compiled = graph.compile();
```

## 四、常见生产问题分类与解决办法

### 4.1 网络与限流类

| 问题类型 | 典型报错特征 | 排查顺序 | 解决方案 |
|----------|-------------|----------|----------|
| 连接超时 | `Connection timed out` | 检查防火墙设置，确保能访问模型服务端点 | 检查网络安全组/防火墙规则是否放行目标地址；验证 `spring.ai.dashscope.api-key` 和端点配置 |
| 认证失败 | `401 Unauthorized` | 先看 application.yml 里有没有写死 Key → 环境变量是否注入 → 配置文件是否被覆盖 | 按顺序排查配置优先级，确保 API Key 正确生效 |
| QPS 超限 | `429 Too Many Requests` | 检查请求频率是否超过服务端限流阈值 | 实现请求排队或熔断机制；使用客户端限流器控制并发数 |
| Token 配额耗尽 | `Quota exceeded` | 检查账户余额和账单状态 | 实现降级策略（缓存、本地回复、切换备用模型） |
| 图像生成任务超时 | `Image generation still pending` | 版本兼容性问题，早期 SNAPSHOT 版本处理视频或图像生成任务逻辑尚不完善 | 升级到最新的稳定版本 |

> 🧠 **费曼解释**：网络问题就像打电话——要么对方号码不对（API Key 错误），要么对方占线忙线（QPS 超限），要么信号不好没接通（连接超时）。排查时要按顺序逐一排除，不能乱猜。

### 4.2 解析与转换类

| 问题类型 | 典型报错特征 | 原因分析 | 解决方案 |
|----------|-------------|----------|----------|
| 工具调用参数解析失败 | `JSON parse error` | LLM 返回的 JSON 参数格式与工具定义不匹配 | 在工具描述中明确参数类型和格式；使用 `@Tool` 注解的 `description` 属性提供详细说明 |
| 结构化输出类型转换异常 | `Conversion failed` | LLM 返回的内容无法映射为目标 Java 类型 | 使用 `ParameterizedTypeReference` 处理泛型集合；在测试环境中先获取原始响应进行格式验证 |
| 流式响应解析不完整 | 返回内容被截断 | 未正确累积 `Flux<ChatResponse>` 中的所有块 | 使用 `collectList()` 或 `reduce()` 进行流式聚合 |

### 4.3 内存与性能类

| 问题类型 | 典型现象 | 原因分析 | 解决方案 |
|----------|----------|----------|----------|
| 流式响应未关闭导致内存泄漏 | 内存持续增长、GC 频繁 | 响应式流 `Flux` 未被正确订阅或释放 | 确保每个 `Flux` 流有正确的生命周期管理；使用 `.timeout()` 设置超时保护 |
| 长上下文 Token 浪费 | Token 用量远超预期、费用飙升 | 每次请求都携带完整对话历史，历史过长造成冗余 | 实现窗口管理：仅保留最近 N 轮对话；对过期对话进行摘要压缩 |
| 大文档一次性加载 OOM | `OutOfMemoryError` | 将超大文档整体加载到内存中处理 | 分块流式处理文档；使用 Spring AI 的分块器（`DocumentSplitter`）逐块处理 |

### 4.4 并发与线程类

| 问题类型 | 典型现象 | 解决方案 |
|----------|----------|----------|
| Reactor 线程阻塞 | 应用响应变慢、线程堆积 | 使用 `.subscribeOn(Schedulers.boundedElastic())` 将阻塞操作转移到专用线程池 |
| MDC 上下文丢失 | `ThreadLocal` 数据在异步线程中丢失 | 使用 Reactor 的 `Context` 传递元数据；或使用 `MDCContextLifter` 自动传递 MDC 上下文 |

### 4.5 部署与环境类

| 问题类型 | 典型现象 | 原因分析 | 解决方案 |
|----------|----------|----------|----------|
| Agent Skill 打包后加载失败 | `Skill not found`，本地运行正常但打包成 JAR 后报错 | SpringBoot 打包后路径机制变化，`ClasspathSkillRegistry` 无法正常加载 resources 下的 Skill 目录 | 切换至 `FileSystemSkillRegistry` 并从文件系统指定 Skill 路径；关注官方 Issue 修复进展 |
| 版本兼容性问题 | 自动装配失败或功能异常 | 使用了过老的版本 | 升级到 Spring AI Alibaba 最新稳定版本 |
| 多实例状态不一致 | Agent 状态错乱 | 使用内存版 ChatMemory 进行多实例部署 | 切换到 Redis/数据库版本的 ChatMemory 实现，实现分布式状态共享 |

> 关于 Agent Skill 打包问题，详细的排查与解决方案可参考阿里云开发者社区文章。

## 五、调优

### 5.1 模型参数调优

| 参数 | 作用 | 事实性任务（FAQ、客服） | 创意任务（写作、头脑风暴） |
|------|------|------------------------|--------------------------|
| `temperature` | 控制输出随机性（0~1） | 0.1~0.3 | 0.7~0.9 |
| `topP` | 核采样阈值，动态调整采样词汇范围 | 0.9~1.0 | 0.9~1.0 |
| `maxTokens` | 最大输出 Token 数 | 500~1000 | 1500~4000 |
| `presence_penalty` | 降低重复话题，越大越不重复 | -0.2~0.2 | 0.3~0.5 |
| `frequency_penalty` | 降低重复词汇，越大越避免词频重复 | -0.2~0.2 | 0.3~0.5 |

> 🔬 **第一性原理推理**：`temperature` 通过调整 Softmax 输出的“分布锐度”控制多样性：高 temperature 使概率分布更平缓、模型更“敢选”低概率词；低 temperature 使分布更尖锐、模型固定选最高概率词。本质上这是一个在“确定性”和“多样性”之间滑动的人工调节熵的过程。

### 5.2 提示词工程优化

Spring AI Alibaba 支持通过 Nacos 配置中心实现动态 Prompt 模板管理：

| 优化策略 | 做法 | 效果 |
|----------|------|------|
| 少样本设计 | 在系统提示中提供 2~3 个“问题—标准答案”示例 | 将零样本准确率从 ~60% 提升到 ~85% |
| 指令清晰化 | 使用分隔符（`###`、`---`）区分指令、上下文和用户输入 | 减少 LLM 误读指令的概率 |
| 动态模板管理 | 将 Prompt 托管至 Nacos 配置中心，实现不重启应用的热更新 | A/B 测试 Prompt 效果、快速响应业务需求变化 |
| 角色设定 | 使用 `SystemMessage` 固定 AI 角色身份和输出规范 | 确保输出风格一致性 |

> Spring AI Alibaba 通过集成 Nacos，实现了从动态 Prompt 模板的灵活调整到模型参数的即时优化，再到敏感信息的安全加密存储的全套配置管理方案。

### 5.3 RAG 分块策略与向量检索调优

Spring AI Alibaba 提供了 40+ 的 document-reader 和 parser 插件，用来将数据加载到 RAG 应用中。

#### 分块大小与重叠

| 文档类型 | 推荐 chunk 大小 | 重叠（overlap）比例 | 理由 |
|----------|----------------|---------------------|------|
| 技术文档 | 256~512 tokens | 10~20% | 代码块较短，小块更易命中 |
| 法律/政策 | 512~1024 tokens | 15~25% | 条款间关联紧密，适当重叠防止信息断裂 |
| 对话记录 | 128~256 tokens | 5~10% | 每轮对话短，小块更精准 |

#### 向量数据库配置示例

```yaml
spring:
  ai:
    dashscope:
      embedding:
        options:
          model: text-embedding-v2
    vector-store:
      elasticsearch:
        index-name: knowledge_base
        dims: 1536
        similarity: cosine
```

### 5.4 缓存策略优化

| 策略 | 实现方式 | 适用场景 | 建议 TTL |
|------|----------|----------|----------|
| 相同请求缓存 | Caffeine/Redis 缓存结果 | 高频重复问题（官网地址、公司政策等） | 5~30 分钟 |
| Embedding 缓存 | 对已向量化的文本进行 Hash 索引缓存 | 知识库更新场景，避免重复向量化 | 随文档更新同步失效 |
| 二级缓存架构 | 本地缓存（Caffeine）→ 分布式缓存（Redis） | 极致性能场景，按需引入 | 本地：分钟级，Redis：小时级 |

### 5.5 超时与流式调优

```yaml
# application.yml 超时配置
spring:
  ai:
    dashscope:
      connect-timeout: 30s   # 连接超时
      read-timeout: 60s      # 读取超时
```

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

// ❌ 不推荐：DTO 字段过多或未标注必填信息
```

### 6.2 安全函数回调（降级处理）

```java
@Component
public class SafeWeatherFunction {
    @Tool(description = "获取指定城市的天气信息")
    public WeatherResponse getWeather(String city) {
        try {
            return weatherApiClient.query(city);
        } catch (TimeoutException e) {
            return new WeatherResponse(city, null, "服务暂时不可用，请稍后重试");
        } catch (Exception e) {
            return new WeatherResponse(city, null, "查询天气失败");
        }
    }
}
```

### 6.3 RAG 构建最佳实践

构建 RAG 应用的全过程分为以下三步：

1. **数据加载与清洗**：从外部知识库加载数据，使用 DocumentReader 读取文档，通过 Splitter 进行分块
2. **向量化存储**：使用 EmbeddingModel 将文本向量化后存储到 Elasticsearch 等向量数据库
3. **检索增强**：通过 `QuestionAnswerAdvisor` 为大模型提供上下文信息

```java
// 完整 RAG 流程
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

### 6.4 生产可观测性

| 维度 | 实践方式 |
|------|----------|
| **Metrics 指标** | 成功/失败计数器、耗时直方图，接入 Prometheus + Grafana |
| **日志埋点** | 记录每次调用的请求内容、响应耗时、Token 用量 |
| **链路追踪** | MDC 注入 traceId，支持跨服务调用链追踪 |

```java
@Component
public class ObservableChatClient {
    private final MeterRegistry meterRegistry;
    
    public String chat(String question) {
        long start = System.currentTimeMillis();
        try {
            String response = chatClient.prompt().user(question).call().content();
            meterRegistry.counter("ai.chat.success").increment();
            return response;
        } catch (Exception e) {
            meterRegistry.counter("ai.chat.error").increment();
            throw e;
        } finally {
            meterRegistry.timer("ai.chat.duration").record(Duration.ofMillis(System.currentTimeMillis() - start));
        }
    }
}
```

## 七、代码示例

### 示例一：配置并创建一个带 Tool Calling 的 ChatClient

```java
// ========== 1. 定义工具类 ==========
@Component
public class CalculateTools {
    @Tool(description = "计算两个整数的乘积")
    public int multiply(int a, int b) {
        return a * b;
    }
    
    @Tool(description = "计算两个整数的和")
    public int add(int a, int b) {
        return a + b;
    }
}

// ========== 2. 配置 ChatClient ==========
@Configuration
public class AiConfig {
    
    @Bean
    public ChatClient chatClient(ChatClient.Builder builder, CalculateTools tools) {
        return builder
            .defaultSystem("你是一个数学助手，只能使用提供的工具进行数学运算。")
            .build();
    }
}

// ========== 3. 使用示例 ==========
@RestController
public class MathController {
    private final ChatClient chatClient;
    
    public MathController(ChatClient chatClient) {
        this.chatClient = chatClient;
    }
    
    @PostMapping("/calculate")
    public String calculate(@RequestBody String question) {
        return chatClient.prompt()
            .user(question)
            .call()
            .content();
    }
    // 输入："请帮我计算 128 乘以 256"
    // 输出："128 × 256 = 32768"
}
```

### 示例二：实现自定义工具（查询天气）

```java
// 1. 定义请求/响应实体
public record WeatherRequest(String city, String unit) {}
public record WeatherResponse(String city, double temperature, String unit, String condition) {}

// 2. 实现工具类
@Component
public class WeatherTools {
    private static final Map<String, Double> TEMP_DATA = Map.of(
        "杭州", 28.5, "北京", 22.3, "上海", 26.8
    );
    
    @Tool(description = "获取指定城市的当前天气信息，参数 city 为城市名称，unit 为温度单位（celsius 或 fahrenheit）")
    public WeatherResponse getWeather(String city, @ToolParam(description = "温度单位，默认为 celsius") String unit) {
        Double temp = TEMP_DATA.getOrDefault(city, 25.0);
        return new WeatherResponse(city, temp, unit != null ? unit : "celsius", "晴");
    }
}

// 3. 使用（框架自动将天气工具集成到对话中）
@RestController
public class WeatherController {
    private final ChatClient chatClient;
    
    @GetMapping("/weather")
    public String askWeather(@RequestParam String question) {
        return chatClient.prompt().user(question).call().content();
    }
}
```

### 示例三：搭建 RAG 流程（加载本地文档 → 检索 → 增强生成）

**1. 添加依赖**

```xml
<!-- Maven：使用 QuestionAnswerAdvisor 必须添加此依赖 -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-advisors-vector-store</artifactId>
    <version>${spring-ai.version}</version>
</dependency>
<!-- 向量数据库依赖（以 Elasticsearch 为例） -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-elasticsearch-store</artifactId>
</dependency>
```

**2. 文档加载与向量化服务**

```java
@Service
@Slf4j
public class DocumentIngestionService {
    
    @Autowired
    private VectorStore vectorStore;
    @Autowired
    private EmbeddingModel embeddingModel;
    
    /**
     * 加载本地文档目录，分块后存入向量数据库
     */
    public void ingestDocuments(String directoryPath) {
        // 1. 读取目录下所有文件
        List<File> files = FileUtils.listFiles(new File(directoryPath), 
            new String[]{"txt", "md", "pdf"}, true);
        
        for (File file : files) {
            // 2. 提取文本内容
            String content = FileUtils.readFileToString(file, StandardCharsets.UTF_8);
            
            // 3. 文本分块（每块 500 字符，重叠 100 字符）
            List<String> chunks = splitText(content, 500, 100);
            log.info("文件 {} 被分割为 {} 个块", file.getName(), chunks.size());
            
            // 4. 向量化并存储
            for (int i = 0; i < chunks.size(); i++) {
                String chunk = chunks.get(i);
                Document doc = new Document(chunk, Map.of(
                    "filename", file.getName(),
                    "chunk_index", i,
                    "source", file.getAbsolutePath()
                ));
                vectorStore.add(List.of(doc));
            }
        }
        log.info("文档入库完成。共处理 {} 个文件", files.size());
    }
    
    private List<String> splitText(String text, int chunkSize, int overlap) {
        List<String> chunks = new ArrayList<>();
        for (int start = 0; start < text.length(); start += chunkSize - overlap) {
            int end = Math.min(start + chunkSize, text.length());
            chunks.add(text.substring(start, end));
            if (end == text.length()) break;
        }
        return chunks;
    }
}
```

**3. RAG 问答服务**

```java
@Service
public class RagChatService {
    
    @Autowired
    private ChatClient chatClient;
    @Autowired
    private VectorStore vectorStore;
    
    /**
     * 使用 QuestionAnswerAdvisor 实现 RAG
     */
    public String askWithAdvisor(String question) {
        // 创建 RAG Advisor
        var qaAdvisor = QuestionAnswerAdvisor.builder(vectorStore)
            .searchRequest(SearchRequest.builder()
                .similarityThreshold(0.7d)
                .topK(5)
                .build())
            .build();
        
        // 通过 ChatClient 使用 Advisor
        return chatClient.prompt()
            .user(question)
            .advisors(qaAdvisor)   // 👈 Advisor 自动完成检索+增强
            .call()
            .content();
    }
}
```

**4. Controller 暴露接口**

```java
@RestController
@RequestMapping("/api/rag")
public class RagController {
    
    @Autowired
    private RagChatService ragService;
    @Autowired
    private DocumentIngestionService ingestionService;
    
    @PostMapping("/ingest")
    public ResponseEntity<String> ingest(@RequestParam String path) {
        ingestionService.ingestDocuments(path);
        return ResponseEntity.ok("文档已入库");
    }
    
    @GetMapping("/ask")
    public ResponseEntity<String> ask(@RequestParam String question) {
        String answer = ragService.askWithAdvisor(question);
        return ResponseEntity.ok(answer);
    }
}
```

## 八、从入门到专家自检清单

完成以下自问自答，可以验证你对 Spring AI Alibaba 的掌握程度：

### 概念理解
1. ChatModel 和 ChatClient 的区别是什么？在什么场景下选择 ChatModel 而不是 ChatClient？
2. Spring AI Alibaba 的三层架构（Agent Framework / Graph / Augmented LLM）各自解决什么问题？
3. `Prompt`、`ChatResponse`、`ChatOptions` 分别封装了 LLM 交互的哪些信息？

### 核心原理
4. 流式响应为什么能降低首块延迟？说说 Reactor Flux 的实现原理。*（提示：首块延迟取决于模型计算第一个响应块的时间）*
5. Tool Calling 的工作流程分哪几步？LLM 本身是否实际执行工具调用？
6. RAG 的「检索→增强→生成」流程中，向量数据库和 EmbeddingModel 各自扮演什么角色？

### 配置与集成
7. 四位版本号的兼容性规则是什么？Spring AI Alibaba 前三位与 Spring AI 主版本对应，社区在前三位基础之上持续迭代第四位。
8. 如何将 Nacos 作为 Prompt 模板的动态配置中心？这样做的好处是什么？
9. 使用 `QuestionAnswerAdvisor` 需要额外添加什么依赖？

### 排错能力
10. 遇到 `401 Unauthorized` 错误，你的排查顺序是怎样的？*（提示：先看 application.yml → 环境变量 → 配置文件覆盖）*
11. 分布式多实例部署时，Agent 状态不一致怎么办？`InMemoryChatMemory` 和 `JdbcChatMemoryRepository` 的适用场景分别是什么？
12. 打包成 JAR 后报 `Skill not found`，可能的原因和解决方案是什么？*（提示：ClasspathSkillRegistry 失效 → 切换 FileSystemSkillRegistry）*

### 调优能力
13. 在事实问答场景下，`temperature` 应该设高还是设低？为什么？
14. RAG 的 chunk 大小和 overlap 对检索效果的影响是什么？给出技术文档场景的经验值。
15. Reactor 线程阻塞问题如何诊断？*（提示：使用 BlockHound 或 jstack 查看阻塞线程）*

### 扩展能力
16. 如何为一个新的模型服务商实现 Spring AI Alibaba 集成？
17. 如何自定义 Advisor 来实现业务特定的请求前/后处理逻辑？
18. Graph 框架中 Node 和 Edge 的核心职责分别是什么？


**参考资源**：

- Spring AI Alibaba 官方文档：https://java2ai.com
- GitHub 源码：https://github.com/alibaba/spring-ai-alibaba
- 阿里云开发者社区系列教程：https://developer.aliyun.com/group/spring-cloud-alibaba-ai
- Spring AI Alibaba 快速入门指南与常见问题解答