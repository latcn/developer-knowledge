
---
## 一、背景与目标

### 1.1 背景

随着 AI Agent 应用从原型验证走向企业级生产落地，单智能体模式逐渐暴露出工具选择困难、上下文膨胀、能力边界模糊等问题。多智能体（Multi-Agent）系统通过将复杂任务分解给多个专业化 Agent 协同完成，已成为必然趋势。

**AgentScope Java** 是阿里巴巴开源的多智能体开发框架的 Java 实现，面向 Java 开发者提供企业级 Agentic 应用构建能力。它采用领先的 ReAct（推理-行动）模式，支持高效的工具调用，并允许开发者对 Agent 执行过程进行实时介入。**Spring AI Alibaba** 是面向 Java 开发者的 Agentic AI 框架，在 1.1.2.2 版本中全面升级了对 AgentScope 框架的支持，以 AgentScope ReActAgent 为核心，全面支持基于 AgentScope 的多智能体编排。

两个框架均基于 JVM，天然契合企业级 Java 技术栈，便于集成到现有微服务体系中。

### 1.2 设计目标

本系统以 **Spring AI Alibaba** 和 **AgentScope Java** 为核心技术底座，设计一套完整的分布式 A2A 智能体系统，具体目标如下：

1. **纯 Java 技术栈**：全栈基于 JVM，Agent 开发、编排、通信、部署均使用 Java，无需跨语言集成
2. **双通信模式支持**：同时支持基于 Nacos 的同步 RPC 调用模式和基于 RocketMQ Lite 的异步消息队列模式
3. **配置化驱动**：Agent 的人格、知识、技能、记忆、子 Agent 规格统一沉淀在结构化工作区中，通过配置文件驱动，无需修改代码即可升级 Agent
4. **完整功能覆盖**：涵盖 Prompt 管理、记忆管理、技能系统、工具调用、MCP 协议集成、上下文管理与压缩摘要、自动化技能生成与知识沉淀、多种 Agent 协作模式
5. **生产级可观测性**：提供服务注册发现、负载均衡、健康检查、分布式追踪等企业级能力

---

## 二、技术底座：AgentScope Java 核心能力

### 2.1 框架概述

AgentScope Java 是一个面向智能体的编程框架，用于构建 LLM 驱动的应用程序。它提供了创建智能体所需的一切：ReAct 推理、工具调用、内存管理、多智能体协作等。AgentScope Java 2.0 在 1.x 的 ReAct 循环基础上，抽出独立的 HarnessAgent 层，同时把模块拆成清晰的核心与扩展。

**核心能力矩阵**：

| 能力维度             | 说明                                                       |
| ---------------- | -------------------------------------------------------- |
| **ReActAgent**   | 采用 ReAct（推理-行动）范式，Agent 自主规划并执行复杂任务                      |
| **HarnessAgent** | 生产级运行时入口，封装工作区、长期记忆、会话持久化、子 Agent、沙箱等工程能力                |
| **多智能体编排**       | 支持 Pipeline、Routing、Handoffs、Supervisor、Subagent 等多种协作模式 |
| **工具系统**         | 统一的 Toolkit 抽象，支持本地工具与远程 MCP 工具                          |
| **MCP 协议**       | 完整支持 Model Context Protocol，通过 stdio/sse/ws 连接外部工具服务器    |
| **记忆管理**         | 双层记忆模型（日常日志 + 长期记忆）+ AutoContextMemory 智能压缩              |
| **技能系统**         | 结构化技能生命周期管理，支持从草稿到生产工具的演进                                |
| **Prompt 管理**    | 所有提示词可修改，支持模板变量与版本管理                                     |

### 2.2 HarnessAgent：企业级运行时入口

`HarnessAgent` 是 AgentScope Java 推荐的入口点，它将长期运行的 Agent 所需的工程能力——工作区驱动的人格、长期记忆、子 Agent 编排、沙箱隔离、技能组合、计划模式——封装到单个 Builder 中。

```
HarnessAgent
├── Workspace（工作区）
│   ├── PERSONA.md          # Agent 人格定义
│   ├── KNOWLEDGE.md        # 领域知识
│   ├── MEMORY.md           # 长期记忆（自动维护）
│   ├── skills/             # 技能目录
│   │   └── */SKILL.md      # 每个技能的定义
│   ├── subagents/          # 子 Agent 定义
│   │   └── *.md
│   └── tools.json          # 工具与 MCP 配置
├── Memory（记忆系统）
│   ├── 日常日志（Daily Logs）
│   └── 长期记忆（Long-term Memory）+ FTS5 全文检索
├── Session（会话管理）
│   └── 跨请求状态恢复
└── Sandbox（沙箱隔离）
    └── Shell/Code/Skill 执行隔离
```

---

## 三、系统整体架构

### 3.1 架构分层（纯 Java 版）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           接入层（API Gateway / WebSocket）                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │              AgentScope Java Agent 集群（全 Java）                    │  │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐    │  │
│  │  │ ReActAgent  │ │ HarnessAgent│ │ Supervisor │ │ Subagent    │    │  │
│  │  │ (单智能体)   │ │ (生产级入口) │ │ (监督者)    │ │ (子智能体)   │    │  │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘    │  │
│  │                                                                       │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │              Spring AI Alibaba Flow Agent 编排层                │  │  │
│  │  │    SequentialAgent / ParallelAgent / RoutingAgent / LoopAgent   │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                    A2A 通信层（双模式）                             │  │
│  │  ┌─────────────────────────┐    ┌─────────────────────────────┐   │  │
│  │  │  模式一：Nacos RPC      │    │  模式二：RocketMQ Lite      │   │  │
│  │  │  • 同步调用             │    │  • 异步消息                  │   │  │
│  │  │  • 实时响应             │    │  • 高可靠队列                │   │  │
│  │  │  • 负载均衡             │    │  • 会话持久化                │   │  │
│  │  │  • 服务发现             │    │  • 断点续传                  │   │  │
│  │  └─────────────────────────┘    └─────────────────────────────┘   │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                      │                                    │
│                                      ▼                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │              基础设施层（Nacos / RocketMQ / Redis / 可观测）        │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 核心组件清单（Java 实现）

| 组件 | 所属框架 | 职责 |
|------|----------|------|
| ReActAgent | AgentScope Java | 本地 Agent 核心实现，ReAct 推理循环 |
| HarnessAgent | AgentScope Java | 生产级运行时入口，封装工作区/记忆/会话/沙箱 |
| AutoContextMemory | AgentScope Java | 智能上下文内存管理，6 种渐进式压缩策略 |
| Skill System | AgentScope Java | 技能生命周期管理，自动化生成与沉淀 |
| Toolkit / MCP | AgentScope Java | 工具注册与 MCP 协议集成 |
| Pipeline / Supervisor | AgentScope Java | 多智能体协作编排 |
| Spring AI Alibaba Flow | Spring AI Alibaba | 工作流编排（Sequential/Parallel/Routing/Loop） |
| A2A Server/Registry | Spring AI Alibaba | Agent 服务暴露与注册发现 |

---

## 四、配置化设计

### 4.1 设计原则

AgentScope Java 2.0 的核心设计理念是 **“配置即集成”** ——Agent 的人格、知识、技能、记忆、子 Agent 规格统一沉淀在结构化工作区中，每次运行自动从工作区加载上下文、结束后自动回写记忆。**调整 Agent 的人格、知识、技能不需要动代码，只需要编辑工作区里的文件——改一个文件就等于升级一次 Agent**。

### 4.2 工作区结构

```
workspace/
├── PERSONA.md                    # Agent 人格定义（系统提示词）
├── KNOWLEDGE.md                  # 领域知识库
├── MEMORY.md                     # 长期记忆（由系统自动维护）
├── sessions/
│   └── <session-id>.log.jsonl    # 会话历史
├── memory/
│   └── YYYY-MM-DD.md             # 日常记忆日志
├── skills/
│   ├── data-analyzer/
│   │   └── SKILL.md              # 数据分析技能定义
│   ├── report-generator/
│   │   └── SKILL.md              # 报告生成技能定义
│   └── ...
├── subagents/
│   ├── researcher.md             # 研究子 Agent 定义
│   └── reviewer.md               # 评审子 Agent 定义
└── tools.json                    # 工具与 MCP Server 配置
```

### 4.3 配置示例（Java 环境）

**tools.json —— 工具与 MCP 声明式配置**：

```json
{
  "tools": [
    {
      "name": "web_search",
      "type": "local",
      "className": "com.example.tools.WebSearchTool"
    }
  ],
  "mcpServers": {
    "github": {
      "transport": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${env:GITHUB_TOKEN}"
      }
    },
    "filesystem": {
      "transport": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "./data"]
    },
    "remote-knowledge": {
      "transport": "sse",
      "url": "https://mcp.example.com/sse",
      "headers": {
        "Authorization": "Bearer ${env:MCP_TOKEN}"
      }
    }
  }
}
```

AgentScope Java 把 MCP server 作为 Agent 工具的一种“来源”——在 tools.json 里声明一个 MCP server，Agent 启动时通过 stdio 或 sse 协议连上它，自动把 server 暴露的工具当作 Agent 自己的 tool。MCP 支持三种连接方式：stdio（本地进程）、sse（远程 HTTP SSE）、ws（双向 WebSocket）。

**PERSONA.md —— Agent 人格定义**：

```markdown
# Agent Persona

你是「智能数据分析助手」，一个专业的数据分析专家。

## 核心能力
- 数据清洗与预处理
- 统计分析与建模
- 数据可视化建议
- 分析报告生成

## 行为准则
- 始终基于数据进行推理，不做无依据的猜测
- 对分析结果提供置信度说明
- 用清晰的语言解释分析方法和结论
```

### 4.4 代码级配置（Java）

通过 Java 代码动态配置 HarnessAgent：

```java
import com.alibaba.agentscope.agent.HarnessAgent;
import com.alibaba.agentscope.memory.AutoContextMemory;
import com.alibaba.agentscope.store.RedisStateStore;

HarnessAgent agent = HarnessAgent.builder()
    .name("data_analyst")
    .workspace(Path.of("./workspace"))
    .model(DashScopeChatModel.builder()
        .apiKey(System.getenv("DASHSCOPE_API_KEY"))
        .modelName("qwen-max")
        .build())
    .memoryConfig(MemoryConfig.builder()
        .triggerMessages(50)
        .triggerTokens(80000)
        .keepMessages(20)
        .build())
    .stateStore(new RedisStateStore(redisConfig))  // 分布式会话持久化
    .sandbox(new DockerSandbox())                   // 沙箱隔离
    .build();
```

---

## 五、Prompt 管理

### 5.1 设计原则

AgentScope 的核心设计原则是对开发者完全透明：**所有提示工程（Prompt Engineering）均显式暴露**，用户可以自己修改所有提示词相关的内容。每一次 API 调用都能够被定位，所有 Agent 的配置来自用户确定的设置。

### 5.2 Prompt 模板管理

AgentScope Java 支持通过 **Nacos Prompt 管理** 实现 Prompt 的集中化、版本化管理：

- **模板变量**：Prompt 模板支援 `{{变量名}}` 文法定義可替換參數
- **版本管理**：每个 Prompt 模板支持多版本，可发布新版本并回滚
- **动态加载**：Agent 运行时从 Nacos 动态获取最新 Prompt 模板

### 5.3 Prompt 模板示例（Java 环境）

```markdown
# 数据分析 Prompt 模板 (v1.0.1)

你是一个专业的数据分析专家，擅长处理 {{data_type}} 类型的数据分析任务。

## 用户需求
{{user_query}}

## 可用数据
{{data_context}}

## 分析要求
1. 首先理解数据结构和字段含义
2. 根据用户需求选择合适的分析方法
3. 提供清晰的分析结果和可视化建议
4. 给出数据驱动的结论和建议

## 输出格式
请按以下结构输出分析报告：
- 数据概览
- 分析方法
- 关键发现
- 结论与建议
```

### 5.4 系统提示词分层

AgentScope Java 的 Prompt 体系分为多层：

| 层级        | 内容               | 来源                  |
| --------- | ---------------- | ------------------- |
| **系统人格层** | Agent 角色、能力、行为准则 | `PERSONA.md`        |
| **知识层**   | 领域知识、业务规则        | `KNOWLEDGE.md`      |
| **记忆层**   | 长期记忆、历史事实        | `MEMORY.md`（自动维护）   |
| **任务层**   | 当前任务指令           | 用户输入 + 模板渲染         |
| **工具层**   | 工具描述与调用规范        | `tools.json` + 自动生成 |
|           |                  |                     |

---

## 六、记忆管理系统

### 6.1 双层记忆模型

AgentScope Java 的 Harness 将记忆分为两层：

| 层级                 | 说明                                               | 维护方式                    |
| ------------------ | ------------------------------------------------ | ----------------------- |
| **Layer 1 — 日常日志** | `memory/YYYY-MM-DD.md`，高频率、低精选，仅追加模式，记录“刚刚讨论了什么” | MemoryFlushManager 自动追加 |
| **Layer 2 — 长期记忆** | `MEMORY.md`，低频率、高精选，完整重写，每个推理轮次注入系统提示词           | MemoryConsolidator 后台整理 |

此外，系统维护 **SQLite FTS5 全文检索索引**，支持对记忆的语义检索。

### 6.2 触发时机

记忆系统的触发点：

| 触发点 | 动作 |
|--------|------|
| **推理前**（PreReasoningEvent） | CompactionHook 检查对话阈值，超限则触发 ConversationCompactor |
| **调用结束**（PostCallEvent） | MemoryFlushHook 提取事实并卸载 |
| **上下文溢出**（ContextLengthExceeded） | forceCompactAndRetry 强制压缩并重试 |
| **超大工具结果**（PostActingEvent） | ToolResultEvictionHook 卸载到文件系统 |
| **后台调度** | MemoryMaintenanceScheduler 每 6 小时运行一次维护周期 |

### 6.3 上下文压缩与摘要（AutoContextMemory）

AgentScope Java 推出 **AutoContextMemory**，智能管理长对话上下文，通过 6 种渐进式压缩策略，在降低 70% token 成本的同时保障信息完整性。

**压缩触发条件**：消息数量阈值或 Token 数量阈值，满足任一即触发压缩。

**6 种渐进式压缩策略**：

| 策略 | 说明 | 成本 |
|------|------|------|
| **策略 1** | 压缩历史工具调用（连续 >6 条），保留工具名称、参数和关键结果 | 轻量级 |
| **策略 2** | 卸载大型消息（带保护），保护最新助手响应和最后 N 条消息 | 中 |
| **策略 3** | 卸载大型消息（无保护） | 中 |
| **策略 4** | 摘要历史对话轮次 | 较重 |
| **策略 5** | 摘要当前轮次大型消息 | 较重 |
| **策略 6** | 压缩当前轮次消息 | 最重 |

**压缩原则**：
- **当前轮次优先**：优先保护当前轮次的完整信息
- **用户交互优先**：用户输入和 Agent 回复的重要性高于工具调用的中间结果
- **可回溯性**：所有压缩的原文都可以通过 UUID 回溯，确保信息不丢失

**多存储架构**：
- **工作内存存储**：存储压缩后的消息，直接参与模型推理
- **原始内存存储**：存储完整、未压缩的消息历史，仅追加模式
- **卸载上下文存储**：以 UUID 为键存储卸载的消息内容，支持按需重载
- **压缩事件存储**：记录所有压缩操作的详细信息

### 6.4 会话持久化

AgentScope Java 支持跨请求、跨进程重启的状态恢复。每次 `call()` 结束后，两个输出自动持久化：
- 会话历史持久化到 `sessions/<id>.log.jsonl`
- 会话索引更新到 `sessions/sessions.json`

分布式场景下，可通过 `RedisStateStore` 实现跨实例的会话状态共享。

---

## 七、技能系统（Skill System）

### 7.1 设计理念

AgentScope Java 的 Harness Skill System 是核心技能机制的 Production-Grade 扩展。它提供了技能的**结构化生命周期管理**——从 Agent 生成的草稿到经过筛选的生产工具——在工作区内管理。它实现了一个 **“自我学习循环”** ，Agent 可以提议、精炼并最终推广新的能力。

### 7.2 技能生命周期

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Skill 生命周期                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐ │
│  │ 草稿阶段 │ → │ 评审阶段 │ → │ 测试阶段 │ → │ 发布阶段 │ → │ 归档阶段 │ │
│  │ (Draft) │    │ (Review)│    │ (Test)  │    │ (Prod)  │    │(Archive)│ │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘ │
│       ↑              │              │              │                       │
│       └──────────────┴──────────────┴──────────────┘                       │
│                        自动化生成与迭代优化                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.3 技能定义格式

每个技能在 `workspace/skills/<skill-name>/SKILL.md` 中定义：

```markdown
---
name: data-analyzer
version: 1.0.0
status: production
author: agent-system
created: 2026-07-01
---

# 数据分析技能

## 描述
对结构化数据进行统计分析、趋势检测和异常发现。

## 输入参数
- `dataset`: 数据集（JSON 数组或 CSV 字符串）
- `analysis_type`: 分析类型（summary / trend / anomaly）
- `options`: 可选配置（JSON 对象）

## 输出
分析结果（JSON 对象），包含统计指标、图表数据和异常列表。

## 使用示例
```json
{
  "dataset": "[...]",
  "analysis_type": "trend",
  "options": {"time_column": "date", "value_column": "sales"}
}
```

## 依赖工具
- Java 类: com.example.analytics.StatisticsTool
- 外部库: Apache Commons Math

## 沙箱要求
- 内存限制: 512MB
- 超时: 30s
- 网络: 禁止
```

### 7.4 技能安装源

Harness 支持从多种来源安装技能：
- **Skill Marketplaces**：Git 仓库、Nacos、MySQL、classpath、自定义存储
- **Workspace**：`workspace/skills/` 对所有用户共享
- **用户隔离**：`<userId>/skills/` 按用户隔离

---

## 八、工具与 MCP 协议集成

### 8.1 工具系统架构

AgentScope Java 提供统一的工具抽象，支持本地工具与远程 MCP 工具的无差别调用：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Toolkit（工具注册中心）                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────┐    ┌─────────────────────────────────────┐   │
│  │     本地工具             │    │          MCP 工具                   │   │
│  │  (Local Tools)          │    │    (Model Context Protocol)         │   │
│  │                         │    │                                     │   │
│  │  • @Tool 注解方法       │    │  • stdio 连接（本地进程）           │   │
│  │  • Java 类方法          │    │  • sse 连接（远程 HTTP SSE）        │   │
│  │  • 脚本工具（Groovy等） │    │  • ws 连接（双向 WebSocket）        │   │
│  └─────────────────────────┘    └─────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 MCP 协议完整支持

AgentScope Java 提供对 Model Context Protocol（MCP）的完整支持，使 Agent 能够连接到外部工具服务器并使用 MCP 生态系统中的工具。

**MCP 工具接入方式**：

| 协议 | 适用场景 | 声明方式 |
|------|----------|----------|
| **stdio** | 本地进程，最常见 | `command` + `args` |
| **sse** | 远程 HTTP SSE 服务器 | `url` + `headers` |
| **ws** | 双向 WebSocket | `url` + `headers` |

**声明式接入（推荐）** ：通过 `workspace/tools.json` 中的 `mcpServers` 段声明 MCP Server。

**编程式接入**：通过 Java 代码动态配置 `ToolsConfig` + `McpServerConfig`。

### 8.3 工具分组与过滤

AgentScope Java 支持通过 `workspace/tools.json` + `toolFilter` 实现工具分组配置化：**新增一个角色只需要在 JSON 里加一段，不需要改 Java 代码**。

```json
{
  "toolGroups": {
    "data_analyst": {
      "tools": ["web_search", "data_query", "statistics"],
      "mcpServers": ["filesystem"]
    },
    "developer": {
      "tools": ["code_search", "git_operations"],
      "mcpServers": ["github"]
    }
  }
}
```

---

## 九、多智能体协作模式

AgentScope Java 生态沉淀了多种适用于不同业务场景的多智能体模式，用户可根据业务选型直接映射到对应的智能体实现。

### 9.1 模式分类

AgentScope Java 将多智能体模式分为两大类：

| 模式类型 | 说明 | 包含模式 |
|----------|------|----------|
| **工作流模式（Workflow Mode）** | 流程在 Agent 或节点间流转，每个节点可与用户交互 | Pipeline、Routing、Handoffs、Custom Workflow |
| **对话模式（Conversational Mode）** | Agent 决策在连续对话上下文中发生，通常只有主 Agent 与用户交互 | Supervisor、Subagents、Skills |

### 9.2 核心多智能体模式（Java 实现）

**1. Pipeline（流水线）**

按预定义顺序逐个执行 Agent。Pipeline 示例使用 Spring AI Alibaba Flow Agent（SequentialAgent、ParallelAgent、LoopAgent）与 AgentScope Agent 子 Agent 协同工作。

```java
// 使用 Spring AI Alibaba Flow Agent 编排 AgentScope Agent
SequentialAgent pipeline = SequentialAgent.builder()
    .addAgent(new AgentScopeAgent(researcherAgent))
    .addAgent(new AgentScopeAgent(analyzerAgent))
    .addAgent(new AgentScopeAgent(reporterAgent))
    .build();
```

**2. Supervisor（监督者）**

一个 Supervisor Agent 作为入口，使用 AutoContextMemory 进行上下文压缩，并将任务路由给多个 Worker Agent。一个 HarnessAgent 搭载多个 SubagentDeclaration，由 LLM 主持群聊与协作。

**3. Handoffs（交接）**

Agent 在完成任务后将控制权交接给下一个专业 Agent，适用于需要角色切换的场景。

**4. Routing（路由）**

根据输入内容动态路由到不同的专家 Agent。

**5. Subagent（子智能体）**

支持声明式定义子 Agent、同步或异步委派子任务，工具执行可配置在隔离沙箱内完成。

### 9.3 Orchestrator + Workers 模式

AgentScope Java 2.0 引入 **orchestrator + workers** 模式替代 1.x 的 MsgHub：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Orchestrator Agent（主协调者）                          │
│                              │                                              │
│              ┌───────────────┼───────────────┐                              │
│              ▼               ▼               ▼                              │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐              │
│  │  Worker Agent 1 │ │  Worker Agent 2 │ │  Worker Agent 3 │              │
│  │   (Subagent)    │ │   (Subagent)    │ │   (Subagent)    │              │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘              │
│                                                                             │
│  特点：主 Agent 主持 Subagent 群聊，LLM 动态决定任务分配与协作方式          │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 十、自动化技能生成与知识沉淀

### 10.1 自我学习循环

AgentScope Java 的 Harness Skill System 支持 **“自我学习循环”** ——Agent 可以提议、精炼并最终推广新的能力：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         自我学习循环                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 任务执行 ──► 2. 经验提取 ──► 3. 技能草稿生成 ──► 4. 技能评审           │
│       ↑                          │                                         │
│       └──────────────────────────┘                                         │
│                                                                             │
│  5. 技能测试 ──► 6. 技能发布 ──► 7. 知识沉淀 ──► 8. 持续优化              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 10.2 工作区驱动的持续演化

Agent 的人格、知识、技能、记忆、子 Agent 规格统一沉淀在结构化工作区里，每次运行自动从工作区加载上下文、结束后自动回写记忆，Agent 的能力随时间持续演化。

**关键特性**：
- **无需重新编译部署**：修改工作区文件即可升级 Agent
- **自动加载**：每次推理前自动从工作区加载最新配置
- **自动回写**：每次调用结束后自动回写记忆和状态
- **版本可追溯**：工作区文件可纳入版本控制

### 10.3 知识沉淀机制（Java 实现）

| 机制 | 说明 |
|------|------|
| **对话压缩摘要** | AutoContextMemory 自动生成对话摘要，沉淀为结构化知识 |
| **长期记忆整理** | MemoryConsolidator 后台将日常日志整理为长期记忆 |
| **技能提取** | 从成功的任务执行中提取可复用的技能模板 |
| **知识库更新** | 新发现的事实和规则自动更新到 KNOWLEDGE.md |

---

## 十一、A2A 通信层设计

### 11.1 双模式架构

系统提供两种 A2A 通信模式，通过配置切换：

| 模式 | 传输层 | 适用场景 |
|------|--------|----------|
| **Nacos RPC** | HTTP + Nacos 服务发现 | 低延迟、实时交互、同步调用 |
| **RocketMQ Lite** | 消息队列（LiteTopic） | 高吞吐、长时任务、高可靠、异步解耦 |

### 11.2 AgentScope Java 的 A2A 支持

AgentScope Java 内置对 A2A 协议的支持：
- **A2A Server**：通过 AgentApp 或 HarnessAgent 将 Agent 暴露为 A2A 服务
- **A2A Registry**：集成 Nacos 实现 Agent 注册与发现
- **A2A Client**：通过 A2AAgent 代理调用远程 Agent

### 11.3 通信统一抽象

```java
public interface A2ATransport {
    CompletableFuture<A2AResponse> send(A2ARequest request);
    void register(AgentCard card);
    AgentCard discover(String agentName);
    boolean healthCheck(String agentName);
}

@Component
@ConditionalOnProperty(name = "a2a.transport", havingValue = "nacos")
public class NacosRpcTransport implements A2ATransport { ... }

@Component
@ConditionalOnProperty(name = "a2a.transport", havingValue = "rocketmq")
public class RocketMqTransport implements A2ATransport { ... }
```

### 11.4 跨 Agent 调用示例（Java）

```java
// Provider 端：暴露 HarnessAgent 为 A2A 服务
HarnessAgent agent = HarnessAgent.builder()
    .name("data_analyst")
    .workspace(Path.of("./workspace"))
    .build();

A2AServer server = new A2AServer(agent);
server.start(8080);

// Consumer 端：通过 A2A Client 调用远程 Agent
A2AClient client = new A2AClient(
    NacosAgentCardResolver.builder()
        .serverAddress("127.0.0.1:8848")
        .namespace("a2a-agents")
        .build()
);
A2AAgent remoteAgent = client.resolve("data_analyst");
String result = remoteAgent.call("分析这份销售数据");
```

---

## 十二、可观测性设计

### 12.1 可观测性维度

| 维度 | 指标示例 | 采集方式 |
|------|----------|----------|
| **Agent 调用** | 调用次数、延迟、错误率、Token 消耗 | Micrometer |
| **A2A 通信** | 请求量、队列深度、重试次数 | 自定义 Metrics |
| **记忆系统** | 压缩次数、Token 节省率、记忆条目数 | AutoContextMemory 事件 |
| **技能系统** | 技能调用次数、成功率、生成数量 | 自定义 Metrics |
| **资源使用** | CPU、内存、GC 频率 | Prometheus + JMX |
| **链路追踪** | 跨 Agent 调用链 | OpenTelemetry |

### 12.2 AgentScope Java 可观测性内置支持

AgentScope Java 通过 **Hook 系统** 提供细粒度的可观测性：
- **PreReasoningEvent**：推理前 Hook，可记录输入上下文
- **PostActingEvent**：行动后 Hook，可记录工具调用结果
- **PostCallEvent**：调用结束 Hook，可记录完整执行轨迹
- **ContextLengthExceeded**：上下文溢出事件，可触发告警

### 12.3 Spring AI Alibaba Admin 集成

Spring AI Alibaba Admin 提供一站式 Agent 平台，支持可视化 Agent 开发、可观测性、评估和 MCP 管理等，可与 AgentScope Java Agent 无缝集成。

---

## 十三、安全设计

### 13.1 安全措施

| 层面 | 措施 |
|------|------|
| **沙箱隔离** | Shell/Code/Skill 的执行通过沙箱后端隔离，用户输入驱动的命令不再直接在宿主机上运行 |
| **工具权限控制** | 通过 toolFilter 实现工具按角色动态供给，避免工具过载 |
| **多租户隔离** | 工作区按用户隔离（`<userId>/skills/`），会话与用户维度隔离 |
| **命名空间隔离** | Nacos 命名空间实现多环境/多租户隔离 |
| **A2A 通信安全** | A2A 协议支持认证机制（如 JWT 或 mTLS） |

---

## 十四、技术栈选型（全 Java）

| 组件 | 选型 | 版本要求 | 说明 |
|------|------|----------|------|
| **Agent 框架** | AgentScope Java | 2.0+ | Agent 开发与运行时框架 |
| **工作流编排** | Spring AI Alibaba | 1.1.2.2+ | Flow Agent 编排与 A2A 能力 |
| **注册中心** | Nacos | 3.1.0+ | A2A Registry 原生支持 |
| **消息队列** | Apache RocketMQ | 5.x+ | LiteTopic 支持 |
| **会话缓存** | Redis | 7.x+ | 会话状态持久化 |
| **可观测** | Micrometer + OpenTelemetry | - | 指标、链路追踪 |
| **部署** | Kubernetes | 1.24+ | 容器编排 |
| **JDK** | JDK 17+ | - | 必要 |
| **构建工具** | Maven 3.9+ / Gradle 8.x | - |  |

---

## 十五、部署拓扑（纯 Java 集群）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            用户 / 客户端                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        API Gateway (Spring Cloud Gateway)                  │
│                   • 路由转发 • 认证鉴权 • 限流熔断                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│              AgentScope Java Agent 集群（多个 Pod / JVM）                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │ HarnessAgent│  │ ReActAgent  │  │ Supervisor  │  │ Subagent    │      │
│  │ (数据专家)   │  │ (天气查询)   │  │ (协调者)    │  │ (专业子Agent)│      │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘      │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                   基础设施层（Nacos / RocketMQ / Redis）                    │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐                 │
│  │ Nacos 集群    │  │ RocketMQ 集群 │  │ Redis 集群    │                 │
│  │ (服务注册发现) │  │ (消息队列)    │  │ (会话缓存)    │                 │
│  └───────────────┘  └───────────────┘  └───────────────┘                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

所有 Agent 均以 Spring Boot 应用形式部署，通过 Nacos 实现服务发现，通过 RocketMQ 实现异步通信，通过 Redis 实现会话共享。

---

## 十六、总结

本设计文档提出了一套完全基于 **Java** 技术栈的分布式 A2A 智能体系统，依托 **Spring AI Alibaba** 和 **AgentScope Java** 两大框架，实现企业级多智能体协作。

**核心优势**：

1. **全栈 Java**：统一技术栈，降低学习成本和集成复杂度，便于与现有 Java 微服务生态融合
2. **配置化驱动**：Agent 人格、知识、技能、记忆全部通过工作区文件管理，改文件即升级，无需重新编译部署
3. **双 A2A 通信**：Nacos RPC 满足低延迟同步场景，RocketMQ Lite 支撑高吞吐异步长任务
4. **完整能力覆盖**：Prompt 管理、双层记忆 + 智能压缩、技能生命周期、MCP 工具生态、多种多智能体协作模式、自动化知识沉淀
5. **生产级特性**：服务注册发现、健康检查、沙箱隔离、多租户、可观测性、分布式会话

该系统可作为企业构建 AI Agent 中台的标准参考架构，支持从简单对话助手到复杂多智能体协同任务的各类场景。