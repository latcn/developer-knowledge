
> **文档版本**：2.0
> **日期**：2026-07-21
> **技术栈**：Spring AI Alibaba 1.1.2.0+、AgentScope Java 2.0+
> **部署目标**：V1.0 MVP（单机部署，具备分布式扩展能力）

## 一、文档说明

本文档基于《写作训练 AI 助手 PRD V7.2》及此前全部技术讨论成果，为 V1.0 MVP 版本提供完整的技术设计方案。

### 1.1 核心设计原则

| 原则 | 说明 |
| :--- | :--- |
| **确定性优先** | Lv.0-3 阶段评价以确定性规则检测为主，AI 不充当“文学评论家” |
| **渐进式扩展** | 单机起步，架构预留分布式扩展能力 |
| **工作区驱动** | Agent 的人格、知识、技能、记忆统一沉淀在结构化工作区中 |
| **混合编排** | Spring AI Alibaba Graph 负责确定性流程编排，AgentScope 负责复杂评价生成 |

### 1.2 V1.0 核心功能范围

| 功能模块 | V1.0 交付内容 | 优先级 |
| :--- | :--- | :--- |
| 用户体系 | 注册/登录、用户画像基础 | P0 |
| 专项训练 | Lv.0-1：感官编码力、情绪传递力、创意生成力（四要素）基础训练 | P0 |
| 引导式填空训练 | Lv.0-1 填空式训练任务 | P0 |
| 整合训练 | 完整微故事创作（200-500字） | P0 |
| AI 评价 | 规则检测 + 基础通顺度检测 + 改进建议 | P0 |
| 用户自评工具 | 写作能力自评清单 | P1 |
| 知识库 | 观察提示卡（冷启动预置） | P1 |
| 技能树 | 个人进度追踪（Lv.0-1 点亮） | P1 |

### 1.3 V1.0 明确不做（技术边界）

| 排除项 | 排除理由 |
| :--- | :--- |
| AI 模拟读者感受 | 主观审美判断，无可靠算法解 |
| “读者触动度”评估 | 无法算法化，AI 无法感知 |
| 风格标签生成（“你像海明威”） | 文学类比易引发争议，无客观标准 |
| 七要素 AI 评分（Lv.4+） | 推迟至 V2.0，V1.0 仅做七要素自检清单 |


## 二、总体架构

### 2.1 架构分层

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              接入层                                        │
│                    REST API（Spring Web MVC）                              │
│              WebSocket（流式评价反馈，可选）                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                              编排层                                        │
│              Spring AI Alibaba Graph（工作流引擎）                          │
│     StateGraph + Node + Edge + Conditional Routing                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                              Agent 层                                      │
│  ┌─────────────────────┐    ┌─────────────────────────────────────────┐   │
│  │  AgentScope Java    │    │      Spring AI Alibaba Agent Framework  │   │
│  │  HarnessAgent       │    │      ReactAgent / A2A 服务暴露          │   │
│  │  (评价/知识库检索)   │    │      (分布式协调)                       │   │
│  └─────────────────────┘    └─────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────────────────┤
│                              基础设施层                                    │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐               │
│  │  Nacos    │ │  Redis    │ │  MySQL    │ │  HanLP   │               │
│  │(服务注册) │ │(会话缓存) │ │(业务数据) │ │(规则检测)│               │
│  └───────────┘ └───────────┘ └───────────┘ └───────────┘               │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 模块划分

| 模块 | 技术选型 | 职责 |
| :--- | :--- | :--- |
| **写作训练核心服务** | Spring Boot 3.x + AgentScope HarnessAgent | Lv.0-1 训练任务执行、评价生成 |
| **Graph 工作流引擎** | Spring AI Alibaba Graph | 训练流程编排（出题→创作→评价→修改闭环） |
| **规则检测引擎** | HanLP 分词 + 词典匹配 | 心理词/行为词检测、通顺度检测 |
| **知识库服务** | AgentScope Workspace + 向量检索（可选） | 观察提示卡管理、语料检索 |
| **用户与画像服务** | Spring Data JPA + Redis | 用户管理、能力画像、进度追踪 |
| **A2A 通信服务** | Spring AI Alibaba A2A + Nacos | Agent 服务注册与远程调用 |


## 三、技术选型

### 3.1 核心框架版本

| 组件 | 版本 | 说明 |
| :--- | :--- | :--- |
| **Spring Boot** | 3.2.x+ | 基础应用框架，要求 JDK 17+ |
| **Spring AI Alibaba** | 1.1.2.0+ | Agent Framework + Graph + A2A 支持 |
| **AgentScope Java** | 2.0.0+ | HarnessAgent + 工作区驱动 + 记忆管理 |
| **Nacos** | 3.1.0+ | A2A Registry 支持 |
| **Redis** | 7.x+ | 会话缓存、分布式状态存储 |
| **MySQL** | 8.0+ | 业务数据持久化 |
| **HanLP** | portable-1.8.x+ | 中文分词、词性标注、词典匹配 |

### 3.2 Maven 依赖

```xml
<!-- Spring Boot -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.8</version>
</parent>

<!-- Spring AI Alibaba -->
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-alibaba-starter</artifactId>
    <version>1.1.2.0</version>
</dependency>

<!-- Spring AI Alibaba Agent Framework -->
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-alibaba-agent-framework</artifactId>
    <version>1.1.2.0</version>
</dependency>

<!-- Spring AI Alibaba Graph -->
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-alibaba-graph</artifactId>
    <version>1.1.2.0</version>
</dependency>

<!-- Spring AI Alibaba A2A + Nacos -->
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-alibaba-a2a-nacos</artifactId>
    <version>1.1.2.0</version>
</dependency>

<!-- AgentScope Java Harness -->
<dependency>
    <groupId>io.agentscope</groupId>
    <artifactId>agentscope-harness</artifactId>
    <version>2.0.0</version>
</dependency>

<!-- AgentScope Java Core -->
<dependency>
    <groupId>io.agentscope</groupId>
    <artifactId>agentscope-core</artifactId>
    <version>2.0.0</version>
</dependency>

<!-- HanLP NLP 工具包 -->
<dependency>
    <groupId>com.hankcs</groupId>
    <artifactId>hanlp</artifactId>
    <version>portable-1.8.4</version>
</dependency>

<!-- Spring Data JPA -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- Spring Data Redis -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<!-- Nacos Discovery -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```


## 四、规则检测引擎设计（无 NLP 研发人员方案）

### 4.1 技术方案：HanLP 分词 + 词典匹配

团队无专职 NLP 研发人员，采用 **开箱即用的开源 NLP 工具包 HanLP** 实现规则检测。HanLP 由北京大学开源，支持分词、词性标注、自定义词典等功能，对 Java 开发者友好。

### 4.2 核心实现

```java
import com.hankcs.hanlp.HanLP;
import com.hankcs.hanlp.seg.common.Term;
import org.springframework.stereotype.Component;

@Component
public class RuleDetector {
    
    // 心理词词典（产品核心资产，持续扩充）
    private static final Set<String> PSYCH_WORDS = new HashSet<>(Arrays.asList(
        "紧张", "悲伤", "愤怒", "恐惧", "喜悦", "羞耻", "焦虑", 
        "沮丧", "兴奋", "感动", "委屈", "愧疚", "得意", "失落"
    ));
    
    // 行为词词典（产品核心资产，持续扩充）
    private static final Set<String> ACTION_WORDS = new HashSet<>(Arrays.asList(
        "握拳", "转头", "吞咽", "搓手", "低头", "抬眼", "咬唇",
        "攥紧", "踱步", "叹息", "颤抖", "凝视", "转身", "停顿"
    ));
    
    public RuleDetectionResult detect(String text) {
        // 1. HanLP 分词 + 词性标注
        List<Term> terms = HanLP.segment(text);
        
        // 2. 心理词检测
        List<String> foundPsychWords = terms.stream()
            .map(Term::getWord)
            .filter(PSYCH_WORDS::contains)
            .collect(Collectors.toList());
        
        // 3. 行为词检测
        List<String> foundActionWords = terms.stream()
            .map(Term::getWord)
            .filter(ACTION_WORDS::contains)
            .collect(Collectors.toList());
        
        // 4. 构建检测结果
        return RuleDetectionResult.builder()
            .psychWordCount(foundPsychWords.size())
            .psychWords(foundPsychWords)
            .actionWordCount(foundActionWords.size())
            .actionWords(foundActionWords)
            .passed(foundPsychWords.isEmpty() && foundActionWords.size() >= 3)
            .build();
    }
}
```

### 4.3 自定义词典扩展

HanLP 支持自定义词典扩展，可将产品特有的心理词、行为词列表导入：

```java
// 在 resources 目录下创建 CustomDictionary.txt
// 格式：词语 词性 词频
// 示例：
// 攥紧 v 1000
// 喉结滑动 v 1000

// 加载自定义词典
HanLP.Config.CustomDictionaryPath = new String[]{
    "data/dictionary/custom/CustomDictionary.txt"
};
```

### 4.4 通顺度检测

通顺度检测采用 **困惑度（Perplexity）** 的简化替代方案：

```java
@Component
public class ReadabilityDetector {
    
    // 基于句长和标点分布的简易可读性评分
    public ReadabilityResult detect(String text) {
        // 1. 句子长度分布
        String[] sentences = text.split("[。！？；]");
        double avgLength = Arrays.stream(sentences)
            .mapToInt(String::length)
            .average()
            .orElse(0);
        
        // 2. 基础可读性判定
        boolean readable = avgLength > 5 && avgLength < 80;
        boolean hasPunctuation = text.matches(".*[，。！？；、].*");
        
        return ReadabilityResult.builder()
            .avgSentenceLength(avgLength)
            .sentenceCount(sentences.length)
            .readable(readable && hasPunctuation)
            .build();
    }
}
```


## 五、Agent 层设计

### 5.1 AgentScope HarnessAgent 配置

`HarnessAgent` 是 AgentScope Java 2.0 推荐的生产级 Agent 入口，它将工作区、长期记忆、会话持久化、子 Agent、沙箱等工程能力封装到一个 Builder 中。

```java
import io.agentscope.harness.agent.HarnessAgent;
import io.agentscope.harness.agent.memory.compaction.CompactionConfig;
import java.nio.file.Paths;

@Configuration
public class WritingTrainerAgentConfig {

    @Bean
    public HarnessAgent writingTrainerAgent(
            @Qualifier("dashscopeChatModel") ChatModel chatModel) {
        
        return HarnessAgent.builder()
                .name("writing-trainer")
                // 系统提示词：从工作区 PERSONA.md 加载
                .sysPrompt(loadSysPrompt())
                // 模型配置
                .model("dashscope:qwen-plus")
                // 工作区：结构化存储 Agent 人格、知识、技能
                .workspace(Paths.get(".agentscope/workspace"))
                // 上下文压缩配置
                .compaction(CompactionConfig.builder()
                        .triggerMessages(30)
                        .keepMessages(10)
                        .build())
                // 状态存储：Redis 分布式存储
                .stateStore(redisStateStore())
                .build();
    }
}
```

### 5.2 工作区结构

AgentScope Harness 采用 **工作区驱动（Workspace-driven）** 的设计理念：

```
.agentscope/workspace/
├── PERSONA.md                    # Agent 人格定义（系统提示词）
├── KNOWLEDGE.md                  # 领域知识库（写作训练方法论）
├── MEMORY.md                     # 长期记忆（由系统自动维护）
├── sessions/
│   └── <session-id>.log.jsonl    # 会话历史
├── memory/
│   └── YYYY-MM-DD.md             # 日常记忆日志
├── skills/
│   ├── sensory-training/
│   │   └── SKILL.md              # 感官编码力训练技能
│   ├── emotion-training/
│   │   └── SKILL.md              # 情绪传递力训练技能
│   └── creativity-training/
│       └── SKILL.md              # 创意生成力训练技能
├── subagents/
│   ├── evaluator.md              # 评价子 Agent 定义
│   └── knowledge-retriever.md    # 知识检索子 Agent
└── tools.json                    # 工具与 MCP 配置
```

### 5.3 评价子 Agent（EvaluatorAgent）

使用 AgentScope 构建评价子 Agent，集成规则检测工具：

```java
import io.agentscope.core.tool.Toolkit;
import io.agentscope.harness.agent.HarnessAgent;

@Configuration
public class EvaluatorAgentConfig {

    @Bean
    public HarnessAgent evaluatorAgent(
            @Qualifier("dashscopeChatModel") chatModel,
            RuleDetector ruleDetector,
            ReadabilityDetector readabilityDetector,
            KnowledgeService knowledgeService) {
        
        // 评价工具集
        Toolkit toolkit = new Toolkit();
        toolkit.registerTool(new RuleDetectionTool(ruleDetector));
        toolkit.registerTool(new ReadabilityTool(readabilityDetector));
        toolkit.registerTool(new KnowledgeRetrievalTool(knowledgeService));
        
        return HarnessAgent.builder()
                .name("evaluator")
                .sysPrompt("""
                    你是写作训练的评价助手。
                    
                    评价原则：
                    1. 只做规则检测和通顺度检测，不做审美判断
                    2. 先肯定用户"写出来了"这件事
                    3. 用对话式反馈，如"你这里用了三个形容词，试着换成一个动词？"
                    4. 明确标注"AI反馈仅供参考"
                    5. 优先使用工具进行检测，再生成评价
                    """)
                .model("dashscope:qwen-plus")
                .toolkit(toolkit)
                .workspace(Paths.get(".agentscope/workspace/evaluator"))
                .build();
    }
}
```


## 六、Graph 工作流编排

### 6.1 核心概念

Spring AI Alibaba Graph 是整个智能体编排背后的核心引擎：

- **StateGraph**：定义节点和边的状态图
- **Node**：封装具体操作或模型调用
- **Edge**：表示节点间的跳转关系
- **OverAllState**：全局状态，贯穿流程共享数据

### 6.2 Lv.0-1 训练闭环工作流

```java
import com.alibaba.cloud.ai.graph.StateGraph;
import com.alibaba.cloud.ai.graph.OverAllState;

@Configuration
public class TrainingWorkflowConfig {

    @Bean
    public StateGraph trainingWorkflow(
            HarnessAgent trainerAgent,
            HarnessAgent evaluatorAgent) {
        
        OverAllState state = new OverAllState();
        
        return StateGraph.builder(state)
                // 1. 出题节点
                .addNode("generate_question", new GenerateQuestionNode(trainerAgent))
                // 2. 用户创作节点
                .addNode("user_write", new UserInputNode())
                // 3. 评价节点（调用 AgentScope EvaluatorAgent）
                .addNode("evaluate", new EvaluateNode(evaluatorAgent))
                // 4. 条件分支：判断是否达标
                .addConditionalEdge("evaluate", state -> {
                    double score = state.get("score", Double.class);
                    int attempts = state.get("attempts", Integer.class);
                    if (score >= 70.0 || attempts >= 3) {
                        return "complete";
                    } else {
                        return "suggest_revise";
                    }
                })
                // 5. 修改建议节点
                .addNode("suggest_revise", new SuggestReviseNode(evaluatorAgent))
                // 6. 用户修改节点
                .addNode("user_revise", new UserInputNode())
                // 7. 再评价节点
                .addNode("re_evaluate", new EvaluateNode(evaluatorAgent))
                // 8. 完成节点
                .addNode("complete", new CompleteNode())
                // 定义边
                .addEdge("generate_question", "user_write")
                .addEdge("user_write", "evaluate")
                .addEdge("suggest_revise", "user_revise")
                .addEdge("user_revise", "re_evaluate")
                .addEdge("re_evaluate", "complete")
                .build();
    }
}
```

### 6.3 混合编排：Spring AI Alibaba + AgentScope

Spring AI Alibaba 与 AgentScope 可通过 **AgentScopeAgent** 进行混合编排：

```java
import com.alibaba.cloud.ai.agentscope.AgentScopeAgent;
import com.alibaba.cloud.ai.flow.SequentialAgent;

@Configuration
public class HybridOrchestrationConfig {

    @Bean
    public SequentialAgent trainingPipeline(
            HarnessAgent trainerAgent,
            HarnessAgent evaluatorAgent,
            HarnessAgent knowledgeAgent) {
        
        // 将 AgentScope HarnessAgent 包装为 Spring AI Alibaba 可调用的节点
        AgentScopeAgent trainerNode = new AgentScopeAgent(trainerAgent);
        AgentScopeAgent evaluatorNode = new AgentScopeAgent(evaluatorAgent);
        AgentScopeAgent knowledgeNode = new AgentScopeAgent(knowledgeAgent);
        
        return SequentialAgent.builder()
                .addAgent(knowledgeNode)   // 1. 知识检索
                .addAgent(trainerNode)     // 2. 训练出题
                .addAgent(evaluatorNode)   // 3. 评价反馈
                .build();
    }
}
```


## 七、A2A 分布式通信设计

### 7.1 A2A 协议集成

Nacos 从 3.1.0 版本开始提供 Agent 注册中心（A2A Registry），实现 Agent 的注册、发现、命名空间隔离、版本管理等功能。Spring AI Alibaba 从 1.0.0.4 版本开始支持通过 Agentic API 便捷定义和构建 Agent。

```yaml
# application.yml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: ${NACOS_SERVER:127.0.0.1:8848}
        namespace: ${NACOS_NAMESPACE:writing-training}
  ai:
    alibaba:
      a2a:
        server:
          enabled: true
          port: 8080
          card:
            name: writing-trainer
            description: 写作训练 AI 助手 - 帮助用户从不会写到能写出合格作品
            version: 1.0.0
```

### 7.2 A2A Server 配置

```java
import com.alibaba.cloud.ai.a2a.server.A2AServer;
import com.alibaba.cloud.ai.a2a.registry.NacosA2ARegistry;

@Configuration
public class A2AServerConfig {

    @Bean
    public A2AServer a2aServer(
            HarnessAgent trainerAgent,
            NacosA2ARegistry registry) {
        
        return A2AServer.builder()
                .agent(trainerAgent)
                .registry(registry)
                .port(8080)
                .build();
    }
}
```

### 7.3 A2A Client 配置

```java
import com.alibaba.cloud.ai.a2a.client.A2AClient;
import com.alibaba.cloud.ai.a2a.discovery.NacosAgentCardResolver;

@Configuration
public class A2AClientConfig {

    @Bean
    public A2AClient a2aClient() {
        NacosAgentCardResolver resolver = NacosAgentCardResolver.builder()
                .serverAddress("127.0.0.1:8848")
                .namespace("writing-training")
                .build();
        
        return A2AClient.builder()
                .resolver(resolver)
                .build();
    }
}
```


## 八、数据模型

### 8.1 核心实体

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;
    private String username;
    private String email;
    private String passwordHash;
    private Integer level;           // Lv.0-6
    private String currentFocus;     // 当前训练专项
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

@Entity
@Table(name = "training_sessions")
public class TrainingSession {
    @Id
    private String id;               // sessionId
    private String userId;
    private String skill;            // 专项名称
    private String level;            // Lv.0-1
    private String questionId;       // 题目ID
    private String userAnswer;       // 用户回答
    private Double score;            // 评分
    private String evaluation;       // AI 评价内容
    private Integer attempts;        // 修改次数
    private String status;           // PENDING/EVALUATED/REVISING/COMPLETED
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

@Entity
@Table(name = "user_progress")
public class UserProgress {
    @Id
    private String userId;
    // 五力专项得分
    private Double sensoryScore;
    private Double emotionScore;
    private Double logicScore;
    private Double expressionScore;
    private Double creativityScore;
    // 技能树点亮状态
    private String skillTree;        // JSON
    // 完成统计
    private Integer totalSessions;
    private Integer completedStories;
    private LocalDateTime lastActiveAt;
}
```

### 8.2 Redis 缓存结构

```
# 会话状态
writing:session:{sessionId} -> TrainingSession (JSON)

# 用户画像
writing:user:{userId}:profile -> UserProfile (JSON)

# 知识库词条
writing:knowledge:{entryId} -> KnowledgeEntry (JSON)

# 题目缓存
writing:question:{questionId} -> Question (JSON)
```


## 九、核心 API 设计

### 9.1 REST API

| 端点 | 方法 | 说明 |
| :--- | :--- | :--- |
| `/api/v1/training/question` | GET | 获取训练题目 |
| `/api/v1/training/submit` | POST | 提交训练回答 |
| `/api/v1/training/evaluate` | POST | 获取 AI 评价 |
| `/api/v1/training/revise` | POST | 提交修改稿 |
| `/api/v1/user/profile` | GET | 获取用户画像 |
| `/api/v1/user/progress` | GET | 获取进度追踪 |
| `/api/v1/knowledge/search` | GET | 搜索知识库 |
| `/api/v1/self-assessment` | POST | 提交自评结果 |

### 9.2 API 示例

**获取训练题目：**

```json
GET /api/v1/training/question?skill=emotion&level=Lv.0-1

{
    "code": 0,
    "data": {
        "questionId": "EMO-SHAME-001",
        "skill": "情绪传递力",
        "category": "羞耻",
        "scene": "深夜便利店，一个穿西装的中年男人在买烟。他刚被公司辞退...",
        "requirements": "请完成以下填空：\n他得知落选的消息...",
        "knowledgeHints": [
            "观察提示：人在羞耻时手部会有什么变化？",
            "观察提示：人在羞耻时目光会看向哪里？"
        ]
    }
}
```

**提交训练回答：**

```json
POST /api/v1/training/submit

{
    "sessionId": "sess-001",
    "questionId": "EMO-SHAME-001",
    "answer": "他吞咽了一下，目光停在价签上。手指在裤缝处蹭了一下。"
}
```


## 十、知识库设计

### 10.1 知识库存储结构

知识库采用 **观察提示卡** 形式，存储为结构化文档：

```json
{
    "id": "HINT-001",
    "category": "emotion",
    "subCategory": "shame",
    "type": "observation",
    "content": "观察提示：人在感到羞耻时，喉部和手部会有什么不自觉的动作？",
    "examples": [
        "喉结滑动、吞咽困难",
        "手指摩挲衣角、指甲掐掌心"
    ],
    "level": "Lv.0-1",
    "source": "expert",
    "tags": ["行为观察", "羞耻"]
}
```

### 10.2 知识库检索

```java
@Service
public class KnowledgeService {
    
    @Autowired
    private ElasticsearchRestTemplate esTemplate;
    
    public List<KnowledgeEntry> searchHints(String skill, String level, String category) {
        return esTemplate.search(
            NativeSearchQuery.builder()
                .withQuery(QueryBuilders.boolQuery()
                    .must(QueryBuilders.termQuery("skill", skill))
                    .must(QueryBuilders.termQuery("level", level))
                    .should(QueryBuilders.matchQuery("content", category))
                )
                .build(),
            KnowledgeEntry.class
        );
    }
}
```


## 十一、可观测性设计

### 11.1 指标采集

```java
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;

@Service
public class MetricsService {
    
    private final Timer trainingTimer;
    private final Counter evaluationCounter;
    
    public MetricsService(MeterRegistry registry) {
        this.trainingTimer = Timer.builder("training.duration")
                .description("训练任务耗时")
                .register(registry);
        this.evaluationCounter = Counter.builder("evaluation.count")
                .description("评价次数")
                .register(registry);
    }
}
```

### 11.2 日志配置

```yaml
# logback-spring.xml
logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"
  level:
    com.alibaba.cloud.ai: DEBUG
    io.agentscope: DEBUG
    com.hankcs.hanlp: INFO
```


## 十二、部署架构

### 12.1 V1.0 单机部署

```
┌─────────────────────────────────────────────────────────────────┐
│                        Docker Compose                           │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │
│  │ 写作训练服务 │  │   Nacos     │  │   Redis     │           │
│  │  (Spring    │  │  (注册中心)  │  │  (会话缓存)  │           │
│  │   Boot)     │  │  8848      │  │  6379      │           │
│  └─────────────┘  └─────────────┘  └─────────────┘           │
│  ┌─────────────┐  ┌─────────────┐                            │
│  │   MySQL     │  │   Admin     │                            │
│  │  (业务数据)  │  │  (可观测)   │                            │
│  │  3306       │  │  9090      │                            │
│  └─────────────┘  └─────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
```

### 12.2 Docker Compose 配置

```yaml
version: '3.8'
services:
  writing-training-service:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - NACOS_SERVER=127.0.0.1:8848
      - REDIS_HOST=redis
      - DB_HOST=mysql
      - DASHSCOPE_API_KEY=${DASHSCOPE_API_KEY}
    depends_on:
      - nacos
      - redis
      - mysql

  nacos:
    image: nacos/nacos-server:3.2.0
    ports:
      - "8848:8848"
    environment:
      - MODE=standalone

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  mysql:
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=writing
      - MYSQL_DATABASE=writing_training
```


## 十三、性能目标与扩展路径

### 13.1 性能目标

| 指标 | 目标值 |
| :--- | :--- |
| API 响应时间（P95） | < 200ms |
| 训练评价生成时间 | < 5s |
| 规则检测处理时间 | < 100ms |
| 并发用户支持 | 100+ |
| 系统可用性 | 99.9% |

### 13.2 扩展路径

| 阶段 | 扩展策略 |
| :--- | :--- |
| **V1.0** | 单机部署，Redis 会话缓存，HanLP 本地词典 |
| **V1.5** | 水平扩展（多副本），Nacos 服务发现 |
| **V2.0** | 分布式部署，Agent 状态外置存储 |
| **V2.5** | 异步消息队列，多租户隔离 |


## 十四、关键技术决策汇总

| 决策点 | 选择 | 理由 |
| :--- | :--- | :--- |
| **规则检测** | HanLP 分词 + 词典匹配 | 无需 NLP 研发人员，开箱即用 |
| **Agent 框架** | AgentScope Java 2.0 | 生产级 HarnessAgent，工作区驱动 |
| **工作流编排** | Spring AI Alibaba Graph | 状态管理、条件路由、循环控制 |
| **服务注册发现** | Nacos 3.1.0+ | A2A Registry 原生支持 |
| **混合编排** | AgentScopeAgent 包装 | 官方支持，无缝集成 |
| **上下文管理** | AgentScope Compaction | 6 种渐进式压缩策略 |
| **会话持久化** | Redis + AgentStateStore | 分布式会话状态共享 |


## 十五、总结

本技术设计文档基于 **Spring AI Alibaba 1.1.2.0+** 和 **AgentScope Java 2.0+**，为“写作训练 AI 助手”V1.0 MVP 提供了完整的技术方案：

| 维度 | 说明 |
| :--- | :--- |
| **规则检测** | HanLP 分词 + 词典匹配，无需 NLP 研发人员 |
| **Agent 运行时** | AgentScope HarnessAgent（工作区驱动、记忆管理） |
| **工作流编排** | Spring AI Alibaba Graph（状态管理、条件路由） |
| **分布式通信** | Spring AI Alibaba A2A + Nacos |
| **混合编排** | AgentScopeAgent 包装，Spring AI Alibaba Graph 编排 |
| **部署模式** | Docker Compose（单机）→ Kubernetes（分布式） |

**核心优势**：

1. **无 NLP 人员即可落地**：HanLP 开箱即用，词典驱动
2. **统一 Java 技术栈**：全栈基于 JVM，降低集成复杂度
3. **工作区驱动**：Agent 人格、知识、技能配置化管理
4. **混合编排**：Spring AI Alibaba 管流程，AgentScope 管评价
5. **渐进式扩展**：从单机到分布式，无需重写业务代码

**V1.0 交付物**：
- Lv.0-1 核心训练闭环（出题→创作→评价→修改）
- 引导式填空训练
- 基础 AI 评价（规则检测 + 通顺度检测）
- 用户画像与进度追踪
- 知识库（冷启动观察提示卡）
- 技能树（个人进度可视化）
