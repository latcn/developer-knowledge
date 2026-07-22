
### 第一阶段： 战略定型 给项目画好边界

####  1.  开发者承担关键决策： 问题边界 核心场景 优先级排序
####  2. 需求定型
   是一项工程思维训练，将模糊的构想或感性冲动，转化为边界清晰、可执行、可验证的工程问题。    
#####   架构四问：
   *  用户画像决定UI交互设计
   *  核心链路确立系统的主干流程，聚焦MVP的核心实现，
   *  阶段目标（什么是当前阶段的MVP，什么状态算完成，哪些功能可以延后）
   *  负向约束  哪些事情是现阶段坚决不做
#####    四个关键决策
  *  问题定义权： 究竟解决什么问题， 不要构建无人使用的需求
  *  边界控制权： 哪些做，哪些不做，做到什么程度？
  *  架构主导权： 数据流向，模块划分，通信协议由谁来决定？
  *  节奏掌控权： 什么时候推进，什么时候暂停
#####  架构师武器
 未来个体开发者的核心心智模型，三种关键能力
 *  最小问题集拆解：学会把做一个微信拆成如何实现两个客户端的文本实时同步 
 *  利用SubAgent建立防火墙： 
    + 思考态AgentA：只负责质疑、提问、补充逻辑，不写代码
    + 执行态AgentB:  在AgentA 定型后， 严格按图索骥
 *  决策点识别
      什么时候应该停止写代码

#####  拆分需求
1.  价值锚点在哪？ 核心需求目标，具体受众， 衡量标准
2.  用户第一动作？ 关键动作是什么，不要让用户去停顿思考，要聚焦
3.  明确的边界在哪？ 明确不做清单

### 第二阶段 工程定型 可落地蓝图
 1.   全栈架构设计： 明确系统边界和数据流转
 2.   领域建模： 逻辑的稳固源于定义的清晰
 3.   数据模型落地： 从领域模型到数据库表
 4.   API设计： 定好接口规范和接口

#####   架构设计的通用范式：
   1.  需求技术化： 不要停留“用户点击按钮” 表象，要找决定系统生死的核心命题
   2.  职责模块化： 根据执行环境（端vs云）和逻辑属性（view、业务、存储）划分边界。
        ***好的架构像乐高，每一块都独立存在，却能严丝合缝地合作***
   3.  流程闭环化： 通过梳理全链路，发现潜在的瓶颈、安全漏洞或逻辑断层
    
####  领域建模
1.  统一语言
    + ***消除词不达意***
    + ***将业务模型编码成AI能理解的结构化上下文，从而能生成可用的代码***
2. 领域建模四步法
    +  ***提炼名词，识别核心实体***
    +  ***定义属性，即值对象***
    + ***梳理动词， 实体之间如何互动，相互之间的关系即业务逻辑*** 
    + ***定义状态流转, 状态机***
      
***领域层应可脱离框架独立运行，仅表达业务规则包含实体、值对象、聚合根、领域服务。
应用层： 处理用例协调与技术适配，CQE, DTO，ApplicationService
基础设施层： 实现技术细节，存储、配置、工具，可依赖领域层，但领域层必须完全独立基础设施层***

####  代码契约 Rules驱动开发
1.  第一类Rule , 解决怎么写。定好统一的风格，ai生成的代码要看起来是一个人写的
2.  第二类Rule,  抵抗一团乱麻，清晰统一的划分，控制项目不变形
3.  第三类Rule,  数据安全、性能，避免致命风险
4.  第四类Rule,  AI 学会自检， 保证能把产品交付
  ***Rule 写的不是建议而是具体明确的施工要求***
  三类内容：
  *  红线： 不遵守就算失败
  *  默认做法： 不写就容易乱
  *  冲突处理： 避免AI自作主张
    
###  第三阶段 工程落地 AI开发可控可调试可交付
#### 小步快跑， AI修改BUG
 *  在不重构系统的前提下，把某一条完整路径跑通
 *  明确边界，修改什么，不能修改什么
 *  明确目标状态，满足什么条件才算成功
 *  验收方式， 例如返回示例，数据状态变化
 *  禁止顺手优化， 除非我明确提出，否则不要进行任何结构调整、命名优化或重构行为，仅限最小必要修改以修复当前问题。
    ```
 例子： 把「必须控制的 4 件事」写进 Prompt
 现在需要修复 TRAE IM 中“好友申请”相关问题。​    
 修复范围（改动边界）：​   
  •  仅处理好友申请与好友关系相关的前后端逻辑​    
  •  不涉及消息系统、鉴权流程、数据库表结构调整​    
  •  不进行任何代码风格优化或结构重构​   
   目标状态（修复完成后应满足）：​    
    发起好友申请后，request 表中成功新增对应记录​  
   验收方式：​    
   •  发起好友申请后，可在数据库中查询到对应 request 记录​  
   额外约束：​    
   •  除非明确要求，不要进行任何重构、命名调整或抽象优化​    
   •  优先保证数据与状态流转正确，其次再考虑前端体验问题​    
   请基于现有代码，在最小修改范围内完成上述修复。
    ```
 

### 全链路开发框架
***具体方法论：***
*  规范驱动开发SDD 
*  三问裁剪法  拆解边界 做什么、不做什么、做到什么程度
*  三步检查法  意图(是不是让AI做的)、质量（风格/规范/一致性）、边界（错误/异常/潜在风险）




***claude.md 应该放的是比如：项目简介和架构、技术栈版本、目录结构说明、开发规范（命名、格式）、常用命令、核心业务概念等信息。
/docs 或 /specs 放的是详细的 API 设计文档、数据库 schema、业务流程文档等信息
skilll 放的应该是固定的流程说明等信息，比如CRUD。***


### ***Trae具体实施步骤：***
#### 1. 需求分析和架构设计
1.  **提示词：** 创建一个SDD驱动的空项目，不包含任何代码。
2.  **生成需求：** PM/风险评估/MVP专家几个Agent
      * 需求简单讨论
      * 技术栈选择
      * 根据我们的讨论，帮我把 Hify 的项目概述写进 01-project-overview.md 。包括产品定位、做什么、不做什么、技术栈、部署与运维预期。格式简洁明了。
      * 评估下风险，给出改进建议，更新project-overview
3.  **代码架构：**
     * Hify 是一个 Spring Boot 单体应用，功能包括模型提供商管理、Agent 配置、对话引擎、知识库 RAG、简版工作流、MCP 工具接入。一个人开发，一期 50 人使用，但后续可能要扩到几千人。代码内部怎么组织？给我方案对比。
     * 基于  Hify  的功能，帮我梳理这些模块之间的依赖关系。谁依赖谁？有没有循环依赖的风险？
     * Hify 是模块化单体，用 Spring Boot + MyBatis-Plus。帮我定义代码组织规范，覆盖：每个模块内部的分层结构、每一层的职责边界、跨模块调用的规则。要求具体到 AI 能直接执行，不要模糊的描述。***最终生成一份架构设计文档***
    * Hify 要调用多个外部 LLM API（OpenAI、Claude、Gemini、Ollama），这些调用慢且不稳定。从线程管理、容错、超时、重试四个维度，给出完整的技术方案
4.    **架构设计 部署性能预判与数据设计**
      * Hify 是模块化单体，技术栈 Spring Boot + Vue + MySQL + Redis + pgvector。目标 50 人内部使用，生产环境用 Docker + K8s 部署。帮我设计当前阶段的部署架构：有哪些组件、请求怎么流转、每个组件的职责是什么。
      * 如果 Hify 要从 50 人扩展到几千人，当前架构需要怎么演进？帮我设计一个分阶段的扩展路径，每一步的触发条件是什么、改什么、不改什么。
5.    **数据模型定义 数据模型和数据库规范**
      *  基于 Hify 的功能范围（模型管理、Agent、对话、工作流、知识库、MCP 工具），帮我梳理核心数据表和它们之间的关系。只要表名和关系，不展开字段。
      * Hify 用 MySQL 8.x + pgvector。帮我定义数据库层面的性能规范，覆盖：索引设计原则、大表预判和应对策略、分页查询注意事项、通用字段约定。要求具体到 AI 建表时能直接执行。
6.  **更新项目规则**
     * 根据 01-project-overview.md 03-architecture.md 04-llm-client-spec.md 总结项目简介和架构、技术栈版本、目录结构说明、开发规范（命名、格式）、常用命令、核心业务概念等信息。更新项目规则 agents.md  
     * Agents.md 结构：
         +  项目上下文： 什么项目，做什么不做什么
         +  架构规范： 模块划分、依赖关系、外部调用处理
         +  代码组织规范： 分层结构（controller/service/mapper）
         +  部署与数据库规范： 部署架构、索引规则、分页规范、pgvector
         +  接口规范与行为指令
     *  以上步骤最终生成：
         *  一份完整的产品定义 做什么、不做什么、做到什么程度
         *  一套应用架构 模块划分、代码组织规范、外部调用处理
         *  一套系统架构  部署形态、性能瓶颈地图、三阶段扩展路径（预判）、数据库规范
         *  一份CLAUDE.md
        
#### ***2.  工程搭建***
1.  后端骨架和公共基础设施
    ***a.*** 按照 CLAUDE.md 中的项目结构和技术栈，创建 Hify 的 Maven 多模块工程骨架。父 pom 声明所有子模块，统一管理 Spring Boot、MyBatis-Plus、Redis 等版本号。子模块之间的依赖关系按 CLAUDE.md 中定义的架构来。只创建 pom 和目录结构，不需要写 Java 代码。
    ***b.*** 在hify-common 中创建统一响应类。按照 CLAUDE.md 接口规范：
     Result\<T\>包含code、message、data 三个字段，提供 ok() 和 fail() 静态方法。
     PageResult\<T\> 继承  Result，额外包含 total、page、size。
    ***c.***  在 hify-common 中创建错误码枚举 ErrorCode 和业务异常类 BizException。ErrorCode 包含 code 和 message，覆盖通用错误（参数错误、未授权、系统内部错误等）。BizException 持有 ErrorCode，支持自定义 message 覆盖。
    ***d.*** 在 hify-common 中创建全局异常处理器 GlobalExceptionHandler，使用 @RestControllerAdvice。捕获 BizException 返回对应错误码，捕获 MethodArgumentNotValidException 返回参数校验错误，兜底捕获 Exception 返回系统内部错误。所有异常响应必须使用 Result.fail() 和 ErrorCode 枚举。
    ***e.*** 在 hify-common 中创建 MyBatis-Plus 配置类。包含：分页插件、自动填充（createTime、updateTime）、逻辑删除配置。
    ***f.***  在 hify-common 中创建 Redis 配置类。包含：RedisTemplate 序列化配置（key 用 String，value 用 JSON）、基础的 RedisUtil 工具类（get/set/delete/expire）。  
    ***g.***  为 hify-provider、hify-agent、hify-chat、hify-mcp 等业务模块创建标准的 package 结构。按照 CLAUDE.md 代码组织规范，每个模块包含 controller/service/service-impl/mapper/entity/dto/config 目录。每个模块只创建 package 和一个空的占位类，不需要写业务代码。
    ***h.***  在 hify-app 中创建 Spring Boot 启动类 HifyApplication，以及 application.yml 配置文件。配置项包括：数据库连接、Redis 连接、MyBatis-Plus 配置、服务端口 8080。
     ***大量代码怎么review 影响范围越大越先查
     结构性问题，模块依赖；公共模块的核心代码； 配置文件；***    
2.  前端功能
     ***a.***  **项目骨架:** 初始化 Hify 前端项目 hify-web。Vue 3 + TypeScript + Vite + Element Plus。目录结构按 CLAUDE.md 中定义的前端结构来。
     Vite 开发服务器配置代理：/api 请求转发到 localhost:8080。
     ***b.***  **axios 统一请求层** 在 hify-web/src/utils/ 下创建 request.ts，封装 axios 实例。baseURL 设为 /api。响应拦截器里判断 code：200 直接返回 data  字段（自动解包），非 200 用 Element Plus 的 ElMessage.error 提示 message，然后 reject。导出 get、post、put、del 四个方法。
     ***c.*** 在 hify-web/src/api/ 下创建 health.ts，用封装好的 request 调用 GET /api/v1/health。导出 getHealth 方法。
     ***d.*** **路由和页面空壳**  在 hify-web 中配置 Vue Router，创建以下路由和对应的空壳页面组件：模型管理、Agent 管理、对话。每个空壳页面只显示页面名称，比如 ProviderList.vue 里就一行"模型提供商管理"。再创建一个 App.vue 布局：左侧 Element Plus 菜单栏（三个菜单项对应三个路由），右侧内容区用 router-view。
     ***e.***  修改 ProviderList.vue，在页面加载时调用 getHealth()，把返回结果显示在页面上。如果调用成功显示绿色的"后端已连接：Hify is running"，失败显示红色的"后端未连接"。
      
3.  后端业务基础组件
     * Hify 项目工程骨架已经搭好（Maven 多模块、hify-common 的 Result / 异常处理 / MyBatis-Plus 配置 / Redis 配置、前端 Vue 工程）。现在要开始做业务功能了。在写业务代码之前，还需要准备哪些基础组件？从数据库层、接口层、外部调用、缓存、可观测性几个角度帮我梳理，每个组件说明它解决什么问题。
     * 按照 CLAUDE.md 的数据库规范和数据模型，生成所有业务表的建表 DDL。放在 hify-app/src/main/resources/db/schema.sql。表名小写下划线、主键 id bigint 自增、时间字段 created_at/updated_at datetime、逻辑删除 deleted tinyint 默认  0、字符集 utf8mb4。包含：provider、model_config、agent、agent_tool、mcp_server、chat_session、chat_message。
     * 在 HifyApplication 启动类上加 @MapperScan(“com.hify.**.mapper”)，扫描所有子模块的 Mapper 包。
     * 在 hify-common 中创建 ThreadPoolConfig（com.hify.common.config）。定义两个线程池：llmExecutor（核心 10，最大 50，队列 100，线程名前缀 llm-，拒绝策略 CallerRunsPolicy）用于 LLM 调用；asyncExecutor（核心 5，最大 20，队列 200，线程名前缀 async-）用于日志异步写入等非关键任务。用 @Bean + @Qualifier 注册。
     * 在 hify-common 中创建 BaseEntity 类。字段：id（Long，@TableId 自增）、createdAt（LocalDateTime，插入时自动填充）、updatedAt（LocalDateTime，插入和更新时自动填充）、deleted（Integer，@TableLogic，默认 0）。后面所有业务实体继承这个类。
     * 在 hify-common 中创建 PageHelper 工具类。提供两个静态方法：toPage(page, pageSize) 把前端参数转成 MyBatis-Plus 的 Page 对象（page 从 1 开始，pageSize 默认 20 最大 100）；toPageResult(IPage) 把查询结果转成我们的 PageResult。
     * 在 hify-common 中配置 Jackson 的全局时间序列化。LocalDateTime 统一用 ISO 8601 格式（yyyy-MM-dd’T’HH:mm:ss），LocalDate 用 yyyy-MM-dd。配置 JavaTimeModule，关掉 WRITE_DATES_AS_TIMESTAMPS。
     * 在 hify-common 中启用 Spring Cache。@EnableCaching，配置 RedisCacheManager，默认 TTL 30 分钟，key 前缀 hify:。配置不同缓存名的 TTL：provider-cache 30 分钟、agent-cache 30 分钟、session-cache 2 小时。
     * 实现一个最简单的 CRUD 演示。实体：DemoItem，只有 name（String）和 status（Integer）两个字段，继承 BaseEntity。完整实现 Controller → Service → ServiceImpl → Mapper → Entity → CreateReq/UpdateReq/Resp DTO。Controller 使用 RESTful 路径  /api/v1/demo-items，返回统一 Result，列表接口返回 PageResult，创建和更新接口使用 @Valid 校验，Service 层处理业务逻辑，Mapper 继承 BaseMapper。
     * 在 hify-common 中创建 LlmHttpClient  类（com.hify.common.http）。内部持有 RestTemplate（连接超时 5s，读超时 60s）和  OkHttpClient（连接超时 5s，读超时 120s）。提供  post(url, headers, body)  方法返回 String，提供 stream(url, headers, body, callback)  方法通过回调逐行返回。所有请求记录日志（URL、耗时、状态码），异常统一转为 LlmApiException（区分  TIMEOUT、AUTH_FAILED、RATE_LIMITED）。
     * 在 hify-common 中配置 Resilience4j 熔断器（com.hify.common.resilience）。application.yml 配置：slidingWindowSize 10，failureRateThreshold 50%，waitDurationInOpenState 30s，permittedNumberOfCallsInHalfOpenState 3。创建 CircuitBreakerService，按 providerName 获取或创建独立熔断器实例。重试逻辑：网络超时重试 2 次（间隔 1s），限流退避重试（2s、4s），认证失败不重试。
     * 在 hify-common 中配置统一日志（com.hify.common.log）。logback-spring.xml 区分环境：开发环境控制台彩色输出，生产环境 JSON 格式输出到文件（按天滚动，保留 30 天）。创建 RequestLogInterceptor（HandlerInterceptor），记录每个请求的 method、path、status、耗时，慢请求（>1s）标 WARN。请求进入时生成 traceId 放入 MDC，请求结束时清理。日志格式：%d{HH:mm:ss} [%thread] [%X{traceId}] %-5level %logger{20} - %msg%n。


   验收：
   ```
   # 入参校验生效吗？
curl -X POST http://localhost:8080/api/v1/demo-items \
  -H "Content-Type: application/json" \
  -d '{"name": "", "status": 1}'
# 期望：Result.fail，提示 name 不能为空

# 正常创建
curl -X POST http://localhost:8080/api/v1/demo-items \
  -H "Content-Type: application/json" \
  -d '{"name": "测试项", "status": 1}'
# 期望：Result.ok(id)，数据库 created_at 自动填充

# 分页列表
curl "http://localhost:8080/api/v1/demo-items?page=1&pageSize=10"
# 期望：PageResult，时间字段是 ISO 8601 格式

# 逻辑删除
curl -X DELETE http://localhost:8080/api/v1/demo-items/1
# 期望：数据库 deleted=1，再查列表看不到了
   ```

4.  前端UI设计和基础组件
    * Hify 是一个 AI Agent 开发平台，面向技术团队内部使用，主要用户是开发者和技术管理者。界面以管理后台为主——大量的表格、表单、配置页面，加上一个对话交互页面。我想要的视觉风格：浅底  +  科技感点缀。整体用浅色背景保持信息可读性（管理后台表格多，深色底长时间看眼睛累）。但不要太素——侧边栏用深色底，按钮和关键交互元素用亮色，制造科技感和品牌感。色调方向：主色用蓝紫系（科技感强），辅色用青色或薄荷绿（数据 / 状态指示）。参考 Linear、Supabase 的视觉风格——干净但不无聊，有设计感但不花哨。帮我设计一套完整的设计系统：主色 / 辅色 / 背景色阶 / 文字色阶 / 圆角 / 阴影 / 过渡动效，用 CSS 变量输出。
    * 改造 Hify 的侧边栏。要求：背景用深色（接近纯黑但不是纯黑，用 --color-bg-dark），和浅色内容区形成对比顶部 Logo 区域：显示"Hify"品牌名，用主色渐变文字，下面一行小字  “AI Agent Platform”菜单项：默认白色文字  +  透明背景，hover 时背景微亮（rgba 白色 10% 透明度），选中态左边一条 3px 的主色竖线  +  背景微亮菜单图标：每个菜单项前加 Element Plus 图标（模型管理用 Setting，Agent 管理用 User，对话用 ChatDotRound）底部：折叠 / 展开按钮，版本号整体要炫酷，有科技感，但不能花哨到影响使用
    * 优化 Hify 的页面整体布局：顶栏：左侧面包屑导航（显示当前页面路径），右侧用户信息区域（头像  +  用户名 placeholder）内容区：背景用  --color-bg-secondary（浅灰），内容卡片用白色背景  +  轻阴影  +  圆角，和背景有层次感页面标题区域：每个页面顶部有标题  +  描述文字  +  操作按钮区（右侧）间距统一：页面边距 24px，卡片内边距 20px，元素间距 16px按钮样式：主要操作用主色渐变按钮（蓝紫渐变），次要操作用白色边框按钮，危险操作用红色
    * 在 hify-web 中创建以下前端公共组件，放在 src/components/ 目录下。所有组件使用 Vue 3 Composition API + TypeScript + Element Plus：HifyTable.vue：通用列表页表格组件。Props 接收 columns 配置（label/prop/width/slot）、api 方法（返回 PageResult 格式）、是否显示分页。内部自动管理 loading 状态、分页参数、数据请求。暴露 refresh() 方法供外部调用刷新。空状态显 Element Plus 的 el-empty。HifyFormDialog.vue：通用表单弹窗组件。Props 接收  title、width、表单 rules。v-model 控制显示隐藏。内部管理提交 loading、关闭时自动重置表单。暴露 open(data?) 方法，传 data 为编辑模式，不传为新增模式。提交时触发 submi 事件，由父组件处理 API 调用。src/composables/useConfirm.ts：删除确认 composable。接收确认文案和 API 方法，弹出 ElMessageBox 确认框，确认后调用 API，成功后显示 ElMessage 成功提示，返回 Promise。一行代码完成  “确认删除→调接口→提示成功”  全流程。src/composables/useRequest.ts：请求状态管理 composable。接收 API 方法，返回 { data, loading, error, execute }。自动管理三态，避免每个页面写 try-catch-finally 样板代码。src/utils/notify.ts：统一通知封装。导出 notifySuccess/notifyError/notifyWarning 三个方法，底层调用 ElMessage，统一配置 duration 和样式。每个组件要有 TypeScript 类型定义，用泛型支持不同数据类型。组件风格和第一步定的设计系统一致。
    * 用 HifyTable 和 HifyFormDialog 实现 Provider 列表页面（src/views/provider/ProviderList.vue）。用 mock 数据，不调真实 API。 列表展示：名称、类型（OpenAI/Claude/Gemini/Ollama）、Base URL、状态（启用 / 禁用，用 el-tag 显示）、创建时间、操作（编辑 / 删除）。 页面顶部有标题  “模型提供商管理”、描述文字、右侧  “新增提供商”  按钮。 点新增弹出 HifyFormDialog，表单字段：名称、类型（下拉）、API Key、Base URL。 删除用 useConfirm。 mock 数据写 5 条，类型分布开。
    * ProviderList 页面的表格和顶部标题区之间间距太大了，从 24px 改成 16px。表格的行高偏高，把 el-table 的 row-style 行高从默认改成 52px。操作列的两个按钮间距太小，加 8px 的 margin-left。
    * 状态列的 el-tag 颜色不对。启用状态用绿色（type=“success”），禁用状态用灰色（type=“info”），不要用红色——禁用不是错误，不应该用 danger 色。
    * 新增提供商的弹窗宽度太宽了，600px 改成 520px。API Key 输入框改成 password 类型，加一个  “显示 / 隐藏”  切换按钮。Base URL 输入框加 placeholder 示例：“https://api.openai.com/v1”。表单 label 宽度统一 100px，从左对齐改成右对齐。
    * 整体看一下 ProviderList 页面，做以下微调： 1.  表格头部背景色太深了，换成  --color-bg-secondary（浅灰）  2.  分页器居右对齐，上方加一条细分割线  3.  “新增提供商”  按钮加一个 Plus 图标  4.  鼠标悬停表格行时背景微微变色（hover 效果）  5.  操作列的  “编辑”  按钮用 text 类型蓝色，“删除”  用 text 类型红色，不要默认的 primary 按钮样式
    * 关键认知：你不需要会写 CSS，但你需要能说清楚“哪里不对”和“应该怎样”。间距太大改成 16px、禁用不应该用红色用灰色、弹窗太宽改成 520px，这些描述不需要前端技能，只需要你看着屏幕，说出你的判断，Claude Code 负责把你的判断翻译成代码。

 5.   ***经验变成SKill***
      * 按照 CLAUDE.md 规范和上面的表结构，在  hify-provider 中创建 Provider、ModelConfig、ProviderHealth 的 Entity 和 Mapper。Entity 继承 BaseEntity（ProviderHealth 除外，它有自己的字段结构），Mapper 继承 BaseMapper。auth_config 和 extra_params 字段用 MyBatis-Plus 的 JSON TypeHandler。
      * ***领域快速理解四问***
          + 是什么，解决什么问题？建立认知框架
          + 用在哪，什么场景？ 理解优先级
          + 由什么组成，哪些是必须的？ 支撑功能取舍
          + 技术架构是什么？ 支撑架构决策
       * ***咨询→设计→拆解→执行→前端对接→验收
       * 我刚完成了 Hify 项目 Provider 模块的开发，流程是这样的： 
         1. 先用咨询模式梳理了供应商选型、数据模型设计、边界问题
         2. 数据模型确定后更新了 schema.sql
         3. 后端按 MVC 分层拆解：Entity+Mapper → DTO → Service（CRUD+ 连通性测试 + 模型同步 + 健康检查）→ Controller
         4. 每步编译或 curl 验证通过再进下一步
         5. 前端对接：创建 API 文件，把 mock 数据源换成真实 API完整
         6. 验收：后端 curl +  浏览器全流程
          帮我把这个流程沉淀成一个 Skill 文件，放在  .claude/skills/module-delivery.md。要求：每一步有明确的产出物和验证方式，关键决策点标注“等待用户确认”，把我踩过的坑写成注意事项。Claude Code 生成第一版
          
 6.   复杂前端交互
      * 对话页面的发送交互，按时间线实现：用户点发送后输入框立刻清空，消息区域底部追加用户气泡；紧接着出现空的 AI 气泡加载动画；前端用 fetch 手动处理 SSE 流（不要用 EventSource，接口是 POST）；每收到 delta chunk 就追加内容到 AI 气泡并滚动到底部；收到 done 事件后移除加载动画，恢复发送按钮。
      *  ***复杂页面拆解有三层：***
          + 结构层： 页面长什么样，组件怎么排
             +  参考截图
          + 行为层： 用户做了什么，页面发生什么
             +  静态交互：四维度描述
                 Provider 管理页。
                 布局：顶部操作栏（标题 + 新增按钮），主体是 Provider 列表表格。
                 数据：展示 name、type、status，status 用标签区分启用 / 禁用。
                 交互：点新增弹出表单弹窗，字段包括名称、类型（下拉选 OpenAI/Anthropic/Ollama）、Base URL、API Key；保存后列表刷新；支持编辑和删除，删除需二次确认。
                 接口：GET /api/v1/providers 查列表，POST 新增，PUT 编辑，DELETE 删除。
             +  动态交互：时间线描述
                  发送消息的完整时间线：
                   用户在输入框输入内容，点发送（或按 Enter）
                   输入框立刻清空，发送按钮变为不可点击
                   消息区域底部出现用户消息气泡（靠右，深色背景）
                   紧接着出现 AI 消息气泡（靠左，浅色背景），内容区域为空，显示加载动画前端用 fetch 手动处理 SSE 流（不要用 EventSource，接口是 POST）
                   每收到一个 delta chunk，把 content 追加到 AI 气泡，同时滚动到底部
                   收到 done 事件，加载动画消失，发送按钮恢复可用
                   如果请求失败或收到 error 事件，AI 气泡显示红色错误提示，发送按钮恢复
              *  ***技术约束要在描述里显式说明。*** SSE over POST、Markdown 渲染、滚动策略，不说 Claude Code 就会用默认方式处理，大概率出错 
          + 细节层： 动画速度、样式微调、边界处理
              + 第三步：增量调整
                  每次只改一个维度，描述说清楚 delta。
                  
          +  ***先对齐结构，再描述行为，最后打磨细节***
 7.   RAG 向量数据库
      *  -- 启用扩展（每个数据库执行一次） CREATE EXTENSION IF NOT EXISTS vector; -- 建表，1536 维对应 OpenAI text-embedding-3-small CREATE TABLE document_chunk (     id        BIGSERIAL PRIMARY KEY,     content   TEXT NOT NULL,     embedding vector(1536) ); -- 相似度查询：<=> 是余弦距离，越小越相似，取最近 3 条 SELECT content,        1 - (embedding <=> '[0.011, -0.033, ...]') AS similarity FROM document_chunk ORDER BY embedding <=> '[0.011, -0.033, ...]' LIMIT 3;
 8.  项目的质量
      *  代码检查：
          你是一位有十年经验的 Java 后端工程师，正在做代码 review。
			以下是 Hify 对话链路的核心代码，请从这几个维度给出意见：
			    安全性：权限校验是否完整，有没有越权访问的可能
			    静默失败：有没有 catch 块吞掉了异常，调用方感知不到失败
			    边界条件：null 处理、空列表、并发场景有没有遗漏
			    性能隐患：N+1 查询、不必要的阻塞调用、大对象
			不需要夸代码写得好。直接说问题，每个问题给出具体位置和修复建议。
      *  核心的链路识别
         ***哪些是系统的核心路径，哪些是风险集中的地方***
         **帮我分析这个系统，输出三个清单：**
			核心链路清单（3-5条）
			   每条链路：名称、涉及的模块和类、为什么是核心链路
			风险集中区域
			   哪些模块/方法最容易出问题、出了问题影响最大
			   每个风险点说明：风险类型（安全/并发/性能/数据一致性）、可能的失败场景
			测试重心建议
			   基于前两条，测试精力应该往哪放
			   哪些地方必须有测试覆盖，哪些可以先跳过
			输出格式：结构化的 CLAUDE.md 片段，我直接复制进去。
		**测试用例判断**
          有没有 IO（DB/HTTP/Redis/文件）？    有 → 考虑集成测试 
         去掉外部依赖，剩下的逻辑有复杂度吗？    没有 → 不值得测    有 → 可以单测 
         改动频率高？出错影响大？    是 → 值得做    否 → 优先级放低
        **生成单测规范**
        基于 CLAUDE.md 中的核心链路和风险地图，帮我生成 Hify 项目的单元测试规范。
			规范需要覆盖：
			1. 哪些代码必须写单测（结合核心链路判断）
			2. 哪些代码不写单测、用集成测试替代（结合 Hify 的外部依赖特点）
			3. 测试命名规范：should_[期望结果]_when_[输入条件]
			4. 测试结构：Given-When-Then
			5. mock 使用规范：什么时候 mock，什么时候不 mock
			  只 mock 跨越本类边界的依赖，不 mock 标准库、不 mock 被测类自身。
			6. 断言规范：用 AssertJ，断言要有意义
			7. 禁止事项：哪些写法不允许出现
			   禁止在测试里写业务逻辑，不要重复计算期望值，直接写字符串字面量。
			输出格式：直接输出 CLAUDE.md 片段，我复制进去就能用。
      *  单元测试
          覆盖纯逻辑，不依赖外部系统。AI执行
          提示：
          深度分析 ProviderService 的 createProvider 方法。
			告诉我：
			1. 这个方法有哪些执行路径（正常路径 + 异常路径）
			2. 每条路径的关键变量是什么
			3. 哪些边界条件最容易出错
			基于分析，给我一份测试计划：测哪些场景、每个场景验证什么断言。
			先给计划，不要写代码
      *   集成测试
          定mock策略，AI执行，一个场景一个
      *    混沌测试   
      *   固化为skill
		   我刚才做了 ProviderService 的单元测试，完整流程是：
			1. 读代码，分析执行路径和边界条件
			2. 输出测试计划（不写代码）
			3. 我确认计划，CC 主动指出计划里的技术问题，调整后再执行
			4. Service 业务逻辑和 DTO 约束规则分两个测试文件
			5. 跑测试，失败了分析是测试写错还是实现有 bug
			请把这个流程固化成一个 SKILL.md 文件。
			让我以后对任何 Service 方法用 /单测 命令触发，CC 自动按这个流程走。
			SKILL.md 要包含：
			- 触发方式
			- 读代码的步骤（先读被测类，再读依赖的 DTO 和 ErrorCode）
			- 测试计划的输出格式（执行路径树 + 边界条件 + 分优先级的场景表）
			- 写代码前的技术确认清单（mock 方式、断言库、是否有 Bean Validation 相关场景）
			- 测试文件拆分原则（Service 逻辑和 DTO 约束分开）
			- 跑测试和处理失败的步骤
 9.   部署
     * 帮我写 Hify 后端的 Dockerfile。
		情况说明：
		- Maven 多模块项目，主模块是 hify-app
		- JDK 17
		- 不需要打包 MySQL、Redis、pgvector，它们是外部服务
		要求：
		- 多阶段构建，减小镜像体积
		- 非 root 用户运行（安全最佳实践）
		- 健康检查用 /api/v1/health 接口
	  *  帮我写 Hify 前端的 Dockerfile。 
	      情况说明：
	       - Vue 3 项目，npm run build 打包 
	       +  用 Nginx 托管静态文件 
	       + 前端需要把 /api 请求反向代理到后端 特别注意： - Hify 有流式响应（SSE），Nginx 需要关闭缓冲
	       +  LLM 调用可能很慢，超时时间要够长
	  *  帮我生成 Hify 的 docker-compose.yml。
			要求：
			- 包含前端和后端两个服务
			- MySQL、Redis、pgvector 是外部服务，通过环境变量配置连接地址
			- 后端健康检查用 /api/v1/health 接口
			- 前端依赖后端健康检查通过后才启动
			- 敏感配置（密码、API Key）从 .env 文件读取攒批


   