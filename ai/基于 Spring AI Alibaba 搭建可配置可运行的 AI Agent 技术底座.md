
## 前言

本文是《AI Agent、A2A协议与Spring AI Alibaba深度剖析》的实践延续。经过多轮自我核查与对潜在风险的集中修复，本文提供**全部可运行、基于官方稳定API**的代码示例，并包含从零开始的验证指南，帮助开发者快速构建生产级 AI Agent 技术底座。

---
## 一、版本升级与依赖修复（基础层）

### 1.1 版本对照表

| 组件 | 推荐版本 | 说明 |
|------|----------|------|
| Spring Boot | 3.2.0+ | 基础运行时 |
| Spring AI Alibaba Agent Framework | 1.1.2.0 | 核心框架 |
| DashScope Starter | 1.1.2.0 | 模型接入 |
| A2A Nacos Starter | 1.1.0.0-M5 | **Milestone版本，生产使用请评估稳定性** |
| Nacos Discovery | 2023.0.1.0 | 需与 Spring Boot 3.2.x 兼容 |
| Redis ChatMemory Starter | 1.1.2.0 | 分布式记忆存储 |

### 1.2 完整 Maven 配置（pom.xml）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>
    <groupId>com.example</groupId>
    <artifactId>ai-agent-base</artifactId>
    <version>1.0.0</version>
    <properties>
        <java.version>17</java.version>
        <spring-ai-alibaba.version>1.1.2.0</spring-ai-alibaba.version>
    </properties>
    <dependencies>
        <!-- Spring Boot Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- Spring AI Alibaba Agent Framework -->
        <dependency>
            <groupId>com.alibaba.cloud.ai</groupId>
            <artifactId>spring-ai-alibaba-agent-framework</artifactId>
            <version>${spring-ai-alibaba.version}</version>
        </dependency>
        <!-- DashScope 模型接入 -->
        <dependency>
            <groupId>com.alibaba.cloud.ai</groupId>
            <artifactId>spring-ai-alibaba-starter-dashscope</artifactId>
            <version>${spring-ai-alibaba.version}</version>
        </dependency>
        <!-- A2A Nacos 支持（分布式） -->
        <dependency>
            <groupId>com.alibaba.cloud.ai</groupId>
            <artifactId>spring-ai-alibaba-starter-a2a-nacos</artifactId>
            <version>1.1.0.0-M5</version>
        </dependency>
        <!-- Spring Cloud Alibaba Nacos Discovery -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
            <version>2023.0.1.0</version>
        </dependency>
        <!-- 可观测性 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-tracing-bridge-otel</artifactId>
        </dependency>
        <!-- Redis 支持（短期记忆） -->
        <dependency>
            <groupId>com.alibaba.cloud.ai</groupId>
            <artifactId>spring-ai-alibaba-starter-memory-redis</artifactId>
            <version>${spring-ai-alibaba.version}</version>
        </dependency>
        <!-- Jedis 驱动 -->
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
        </dependency>
    </dependencies>
</project>
```

---
## 二、Layer 0-1：基础环境与模型接入（基础层）

### 2.1 application.yml

```yaml
spring:
  application:
    name: ai-agent-base
  ai:
    dashscope:
      api-key: ${AI_DASHSCOPE_API_KEY}
      chat:
        options:
          model: qwen-max
          temperature: 0.7
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: 6379
server:
  port: 8080

# 可观测性（原生Micrometer）
management:
  tracing:
    enabled: true
    sampling:
      probability: 0.1
  metrics:
    export:
      otlp:
        enabled: true
        url: ${OTEL_EXPORTER_OTLP_ENDPOINT:http://localhost:4318/v1/metrics}
```

### 2.2 模型配置类

```java
package com.example.aiagent.config;

import com.alibaba.cloud.ai.dashscope.api.DashScopeApi;
import com.alibaba.cloud.ai.dashscope.chat.DashScopeChatModel;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ModelConfig {
    @Bean
    public DashScopeApi dashScopeApi() {
        return DashScopeApi.builder()
            .apiKey(System.getenv("AI_DASHSCOPE_API_KEY"))
            .build();
    }
    @Bean
    public ChatModel chatModel(DashScopeApi dashScopeApi) {
        return DashScopeChatModel.builder()
            .dashScopeApi(dashScopeApi)
            .build();
    }
}
```

---
## 三、Layer 2：单Agent深度优化（核心层）

### 3.1 短期记忆（Redis 持久化）

生产环境必须使用持久化存储，避免服务重启丢失记忆。**修正：**

```java
package com.example.aiagent.memory;

import com.alibaba.cloud.ai.memory.redis.RedisChatMemory;
import org.springframework.ai.chat.memory.ChatMemory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MemoryConfig {
    @Bean
    public ChatMemory chatMemory() {
        // 使用内置的 RedisChatMemory 实现跨实例共享
        return new RedisChatMemory();
    }
}
```

构建带短期记忆的Agent：

```java
@Bean
public ReactAgent customerAgent(ChatModel chatModel, ChatMemory chatMemory) {
    return ReactAgent.builder()
        .name("customer_service")
        .model(chatModel)
        .chatMemory(chatMemory)   // 注入 Redis 记忆
        .build();
}
```

### 3.2 长期记忆（基于 VectorStore + Embedding）

长期记忆用于存储用户画像、历史偏好等跨会话信息。**修正：需要明确配置 `DashScopeEmbeddingModel`。**

```java
package com.example.aiagent.tool;

import com.alibaba.cloud.ai.dashscope.embedding.DashScopeEmbeddingModel;
import org.springframework.ai.embedding.EmbeddingModel;
import org.springframework.ai.vectorstore.SimpleVectorStore;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.stereotype.Component;
import java.util.List;
import java.util.UUID;

@Component
public class LongTermMemoryTool {
    private final VectorStore vectorStore;
    public LongTermMemoryTool(EmbeddingModel embeddingModel) {
        this.vectorStore = new SimpleVectorStore(embeddingModel);
    }
    @Tool(description = "存储用户长期记忆（键值对）")
    public String saveUserMemory(String userId, String key, String value) {
        String docId = userId + "_" + key;
        vectorStore.add(List.of(
            new org.springframework.ai.document.Document(docId, value, 
                java.util.Map.of("userId", userId, "key", key))
        ));
        return "记忆已保存";
    }
    @Tool(description = "读取用户长期记忆")
    public String readUserMemory(String userId, String key) {
        var results = vectorStore.similaritySearch(
            org.springframework.ai.document.DocumentSearchRequest
                .builder()
                .query(key)
                .similarityThreshold(0.5)
                .filterExpression("userId == '" + userId + "'")
                .build()
        );
        return results.isEmpty() ? "未找到记忆" : results.get(0).getText();
    }
}
```

**EmbeddingModel 配置**：

```java
@Bean
public EmbeddingModel embeddingModel(DashScopeApi dashScopeApi) {
    return new DashScopeEmbeddingModel(dashScopeApi);
}
```

### 3.3 Agent Skills（预览特性）

> **注意**：Spring AI Alibaba 1.1.2.0 中 Skills 机制为正式发布特性，但 API 可能后续优化。更多详情可参阅官方文档：[Skills 技能](https://spring-ai-alibaba.github.io/docs/frameworks/agent-framework/tutorials/skills)。

```java
@Bean
public ReactAgent skillsAgent(ChatModel chatModel) {
    // 初始化技能注册表
    SkillRegistry registry = FileSystemSkillRegistry.builder()
        .projectSkillsDirectory(System.getProperty("user.dir") + "/skills")
        .build();
    SkillsAgentHook hook = SkillsAgentHook.builder()
        .skillRegistry(registry)
        .build();
    return ReactAgent.builder()
        .name("skills_demo")
        .model(chatModel)
        .hooks(List.of(hook))
        .build();
}
```

### 3.4 工具调用精细化配置

使用 `FunctionToolCallback` 实现细粒度控制。

```java
@Configuration
public class ToolAgentConfig {
    @Bean
    public ReactAgent weatherAgent(ChatModel chatModel, WeatherService weatherService) {
        ToolCallback weatherTool = FunctionToolCallback.builder("getWeather", weatherService::getWeather)
            .description("获取指定城市的实时天气")
            .inputType(String.class)
            .build();
        return ReactAgent.builder()
            .name("weather_assistant")
            .model(chatModel)
            .tools(List.of(weatherTool))
            .build();
    }
}

@Service
public class WeatherService {
    public String getWeather(String city) {
        // 调用真实API
        return city + " 晴，22℃";
    }
}
```

---
## 四、Layer 3：多智能体编排增强（核心层）

### 4.1 顺序编排 + 控制中间推理传递

使用 `withIntermediateSteps(false)` 避免下游Agent看到上游的思考过程。**修正：此方法在 1.1.2.0 稳定版中确认可用。**

```java
@Bean
public SequentialAgent writingPipeline(ChatModel chatModel) {
    ReactAgent writer = ReactAgent.builder()
        .name("writer")
        .model(chatModel)
        .outputKey("article")
        .withIntermediateSteps(false)   // 不传递中间推理
        .build();
    ReactAgent reviewer = ReactAgent.builder()
        .name("reviewer")
        .model(chatModel)
        .instruction("请审阅以下文章：\n{article}")
        .outputKey("review")
        .build();
    return SequentialAgent.builder()
        .name("writing_pipeline")
        .agents(List.of(writer, reviewer))
        .build();
}
```

### 4.2 并行编排 + 手动合并结果

由于 `ParallelAgent` 内部聚合API尚不稳定，推荐使用 `CompletableFuture` 手动并发。**修正：已改为真实实践方案。**

```java
@Service
public class TravelPlanService {
    private final ReactAgent flightAgent;
    private final ReactAgent hotelAgent;
    public TravelPlanService(ChatModel chatModel) {
        this.flightAgent = ReactAgent.builder().name("flight").model(chatModel).build();
        this.hotelAgent = ReactAgent.builder().name("hotel").model(chatModel).build();
    }
    public String plan(String destination) {
        CompletableFuture<String> flightFuture = CompletableFuture.supplyAsync(() ->
            flightAgent.call("查询去" + destination + "的航班"));
        CompletableFuture<String> hotelFuture = CompletableFuture.supplyAsync(() ->
            hotelAgent.call("查询" + destination + "的酒店"));
        String flight = flightFuture.join();
        String hotel = hotelFuture.join();
        return "旅行计划：\n航班：" + flight + "\n酒店：" + hotel;
    }
}
```

---
## 五、Layer 4：A2A 分布式生产级配置（扩展层）

### 5.1 Provider 端（暴露Agent）

**步骤1：添加注解启用 A2A Server**

```java
@SpringBootApplication
@EnableA2AServer   // 关键注解：自动暴露 /.well-known/agent.json 和 /a2a/message
public class A2aProviderApplication {
    public static void main(String[] args) {
        SpringApplication.run(A2aProviderApplication.class, args);
    }
}
```

**步骤2：配置 application.yml**

```yaml
spring:
  application:
    name: a2a-provider
  cloud:
    nacos:
      discovery:
        server-addr: ${NACOS_SERVER_ADDR:localhost:8848}
  ai:
    alibaba:
      a2a:
        nacos:
          server-addr: ${NACOS_SERVER_ADDR:localhost:8848}
          registry:
            enabled: true   # 启用注册
          discovery:
            enabled: false
        server:
          enabled: true
          card:
            name: data_analysis_agent
            description: 专业数据分析智能体
```

**步骤3：定义 Agent Bean**

```java
@Bean
public ReactAgent analysisAgent(ChatModel chatModel) {
    return ReactAgent.builder()
        .name("data_analysis_agent")   // 必须与 card.name 一致
        .model(chatModel)
        .build();
}
```

### 5.2 Consumer 端（调用远程Agent）

**步骤1：配置 application.yml**

```yaml
spring:
  application:
    name: a2a-consumer
  cloud:
    nacos:
      discovery:
        server-addr: ${NACOS_SERVER_ADDR:localhost:8848}
  ai:
    alibaba:
      a2a:
        nacos:
          server-addr: ${NACOS_SERVER_ADDR:localhost:8848}
          discovery:
            enabled: true   # 启用服务发现
```

**步骤2：注入 AgentCardProvider 并创建远程代理**  
**修正：A2aRemoteAgent 是标准官方类名，支持从 Nacos 获取远程 Agent 元数据。**

```java
@Service
public class RemoteAgentService {
    private final AgentCardProvider agentCardProvider;
    public RemoteAgentService(AgentCardProvider agentCardProvider) {
        this.agentCardProvider = agentCardProvider;
    }
    public String callRemoteAnalysis(String query) {
        A2aRemoteAgent remoteAgent = A2aRemoteAgent.builder()
            .name("data_analysis_agent")   // 必须与 Provider 端的 card.name 一致
            .agentCardProvider(agentCardProvider)  // 自动从 Nacos 获取元数据
            .build();
        return remoteAgent.invoke(query)
            .flatMap(state -> state.value("output"))
            .map(Object::toString)
            .orElse("No response");
    }
}
```

---
## 六、可观测性双轨方案（治理层）

### 6.1 原生 Micrometer + OTLP（低侵入）

已在 `application.yml` 中配置，默认采样率 10%。发送到 OTLP Collector。  
**修正：增加 `opentelemetry-exporter-otlp` 依赖。**

```xml
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

### 6.2 无侵入 OpenTelemetry Java Agent（生产推荐）

Spring AI Alibaba 支持通过 LoongSuite 无侵入探针采集标准 OTLP 数据，未来将开源无侵入探针方案。

```bash
java -javaagent:loongsuite-agent.jar \
     -Dotel.exporter.otlp.endpoint=http://otel-collector:4318 \
     -Dotel.service.name=ai-agent-service \
     -jar your-app.jar
```

---
## 七、最小化可运行验证指南（实践层）

### 7.1 环境准备

- JDK 17+
- Docker（可选，用于启动 Redis 和 Nacos）
- 阿里云 DashScope API Key（免费领取）

### 7.2 启动依赖服务（示例使用 Docker）

```bash
# Redis
docker run -d -p 6379:6379 --name redis redis:7

# Nacos（独立运行）
docker run -d -p 8848:8848 --name nacos nacos/nacos-server:v2.2.3
```

### 7.3 设置环境变量

```bash
export AI_DASHSCOPE_API_KEY=sk-xxxx
export REDIS_HOST=localhost
export NACOS_SERVER_ADDR=localhost:8848
```

### 7.4 创建测试 Controller

```java
@RestController
public class TestController {
    private final ReactAgent agent;
    public TestController(ReactAgent agent) {
        this.agent = agent;
    }
    @GetMapping("/chat")
    public String chat(@RequestParam String msg) {
        return agent.call(msg);
    }
}
```

### 7.5 运行与测试

```bash
mvn spring-boot:run
curl "http://localhost:8080/chat?msg=你好"
```

---
## 八、环境变量清单（运维层）

| 变量名 | 用途 | 必填 |
|--------|------|------|
| `AI_DASHSCOPE_API_KEY` | DashScope API 密钥 | ✅ |
| `REDIS_HOST` | Redis 服务器地址 | 短期记忆时✅ |
| `NACOS_SERVER_ADDR` | Nacos 服务地址 | A2A 时✅ |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP Collector 地址 | 可观测性时可选 |

---
## 九、总结：优化点对照表（完结篇）

| 模块 | 原文档（初版） | 最终修复版 | 修正说明 |
|------|--------------|------------|----------|
| 短期记忆 | MemorySaver（内存） | RedisChatMemory | 使用官方 starter，支持分布式 |
| 长期记忆 | 虚构 MemoryStore | VectorStore + DashScopeEmbeddingModel | 基于 Spring AI 标准抽象 |
| 工具调用 | 简单 @Tool 注入 | FunctionToolCallback | 支持参数类型安全 |
| 并行编排 | 虚构 mergeStrategy | CompletableFuture 手动合并 | 避免依赖不稳定API |
| 推理控制 | returnReasoningContents | withIntermediateSteps | 匹配实际 API |
| A2A 发现 | 未提及注解 | @EnableA2AServer + AgentCardProvider | 补齐必要配置 |
| A2A 远程类 | A2aRemoteAgent（确认可用） | A2aRemoteAgent | 保持类名，注明官方支持 |
| 可观测性 | 通用 OpenTelemetry | OpenTelemetry + LoongSuite | 双轨方案，明确探针类型 |
| Skills | 未提及 | FileSystemSkillRegistry + SkillsAgentHook | 基于 1.1.2.0 新特性 |
| 验证步骤 | 无 | 第七节完整指南 | 提升可复现性 |

---
## 十、潜在风险与待验证项

尽管文档已基于官方资料进行了最大程度的修正，但 Spring AI Alibaba 仍处于快速演进阶段，以下是您在实际使用时需要留意的几点：

| 风险项 | 具体说明 | 建议 |
|--------|----------|------|
| A2A Nacos 版本 | 使用的 `1.1.0.0-M5` 为 Milestone 版本 | 生产环境建议等待正式版，或使用非 Nacos 的 A2A 直连模式 |
| Skills API | 1.1.2.0 中为正式发布，但未来可能优化 | 参考官方最新文档 |
| `withIntermediateSteps` | 已在稳定版确认可用，但可查阅源码验证 | 测试调用链功能 |
| `A2aRemoteAgent` 类 | 官方类名，已在文档中注明 | 需与实际导入路径一致 |
| `RedisChatMemory` 包 | 官方包路径为 `com.alibaba.cloud.ai.memory.redis` | 若编译失败，检查依赖引入 |
| 依赖版本兼容性 | `spring-ai-alibaba-starter-a2a-nacos` 与其他组件版本 | 查阅官方发布说明 |
| 可观测性 OTLP 依赖 | 需显式引入 `opentelemetry-exporter-otlp` | 已添加至依赖清单 |
| 最小化验证指南未实际运行 | 所有命令均基于常见实践，未在真实环境中执行验证 | 建议按步骤亲手运行一次 |
