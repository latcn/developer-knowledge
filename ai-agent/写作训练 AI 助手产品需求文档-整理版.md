# WriteTrainer 数据模型与存储设计

> 版本：v1.2（合并版）
> 状态：已评审（含两轮修订：文本/评价分离 + AI 评价互动、作者眼列表详情、作品集版本历史）
> 关联文档：`doc/写作训练AI助手产品需求文档.md`

---

## 1. 设计目标与核心原则

本应用为「本地优先（local-first）」架构：所有用户数据先在本地持久化，后续接入远端存储时无需改动业务代码，只需替换存储实现并补充同步逻辑。

### 1.1 核心原则

| #   | 原则                   | 说明                                                            |
| --- | -------------------- | ------------------------------------------------------------- |
| P1  | **原始文本与评价反馈分离存储**    | 文本快照（Revision/Snapshot）是不可变实体；评价反馈是追加式实体；两者只通过 id 外键关联，永不互相嵌套 |
| P2  | **改后再评 = 新快照 + 新评价** | 每次评价冻结一份不可变文本快照；每次修改产生新快照；旧快照与旧评价全部保留，形成完整进化轨迹                |
| P3  | **追加式日志**            | 评价/异议/裁决只追加、不覆盖、不物理删除（会话级墓碑删除，子表级联标记）                         |
| P4  | **派生值不落库**           | 可由日志推导的数据（matchHistory、firstEvaluation、采纳数等）一律计算得出，避免双写不一致    |
| P5  | **同步友好**             | 客户端 UUID 主键、version 乐观锁、deleted 墓碑、_dirty 待同步标记——本地与远端共用同一结构  |
| P6  | **列表轻量化**            | 列表只读投影字段，全文仅在详情/展开时读取                                         |

### 1.2 存储抽象（未来可插拔）

业务层只依赖统一接口，本地提供两个实现，未来新增远端实现即可切换：

- `IDataStore<T>`：`get / put / query / delete / watch`（本地实现：LocalStorageStore、IndexedDBStore；未来：RemoteStore）
- 介质分配原则：**轻量元数据/会话态 → localStorage；全文/大对象/高频写/追加日志 → IndexedDB**

---

## 2. 通用字段约定

所有用户数据实体统一携带以下字段（本地与远端共用，未来同步无需改表）：

| 字段                      | 类型                  | 说明                                       |
| ------------------------- | --------------------- | ------------------------------------------ |
| `id`                      | string（客户端 UUID） | 全局唯一主键；离线创建不冲突，同步天然幂等 |
| `ownerId`                 | string                | 归属用户（对应 user.id），远端按用户隔离   |
| `createdAt` / `updatedAt` | ISO 时间字符串        | 审计 + 增量同步游标                        |
| `version`                 | number                | 乐观并发版本号，冲突时以高版本为准         |
| `deleted`                 | boolean               | 墓碑标记：本地删除=标记，同步后再物理清除  |
| `_dirty`                  | boolean（仅本地）     | 待同步标记（本地存储中用于未来同步队列）   |

---

## 3. 实体关系总览

```
UserAccount ──(mock 期)── Session
UserProfile（用户资料/能力/统计）
MaterialCard（灵感素材）
LearningProgress（学习路径进度）
GrowthRecord（成长记录）

骨架工坊：
SkeletonSession 1──n DraftRevision 1──n SkeletonEvaluation
                                         1──n EvaluationReply（用户异议）
作者眼：
JurySession 1──n StorySnapshot（微小说快照）
            1──n CriticismRevision（批判修订）1──n Adjudication（裁决，round=initial/rebuttal）
作品集：
PortfolioWork 1──n WorkVersion（版本链，含修改点与关联评价）
```

---

## 4. 实体定义

### 4.1 账号与会话（mock 期专用）

#### UserAccount（本地 mock 账号）

| 字段                                           | 类型           | 说明                                           |
| ---------------------------------------------- | -------------- | ---------------------------------------------- |
| id / createdAt / updatedAt / version / deleted | 通用           |                                                |
| username                                       | string         | 昵称                                           |
| email                                          | string         | 登录名（唯一；`admin@sina.cn` 为保留演示邮箱） |
| salt                                           | string         | 密码盐                                         |
| passwordHash                                   | string         | SHA-256（非安全上下文降级 FNV-1a）             |
| radar / stats / isPremium                      | 同 UserProfile | 冗余同步                                       |

> 仅 mock 期使用；接入真实后端后整表废弃，认证改由服务端 httpOnly cookie 承担。

#### Session（会话）

| 字段      | 类型   | 说明                                      |
| --------- | ------ | ----------------------------------------- |
| token     | string | 会话令牌（`wt_` 前缀标识 mock 会话）      |
| userId    | string | 关联用户                                  |
| expiresAt | number | 过期时间戳（记住我 30 天 / 默认 12 小时） |

### 4.2 用户与素材

#### UserProfile（用户资料与能力）

| 字段                  | 类型    | 说明                                                                                    |
| --------------------- | ------- | --------------------------------------------------------------------------------------- |
| id / username / email | string  | 基本资料                                                                                |
| radar                 | Radar4D | structureControl / revisionSensitivity / textEconomy / aestheticDiscrimination（0-100） |
| stats                 | Stats   | totalStories / publishedCount / totalSessions                                           |
| isPremium             | boolean | 会员标记                                                                                |

#### MaterialCard（灵感素材）

| 字段                                                     | 类型     | 说明       |
| -------------------------------------------------------- | -------- | ---------- |
| id / ownerId / createdAt / updatedAt / version / deleted | 通用     |            |
| what / why / how                                         | string   | 素材三要素 |
| tags                                                     | string[] | 标签       |

#### LearningProgress（学习路径进度，预留）

| 字段                                         | 类型                    | 说明                                            |
| -------------------------------------------- | ----------------------- | ----------------------------------------------- |
| id / ownerId / updatedAt / version / deleted | 通用                    | 每用户一条                                      |
| completedPhases                              | string[]（phaseId）     | 已完成阶段（observe/imitate/critique/diagnose） |
| milestones                                   | Record<phaseId, string> | 各阶段里程碑产出索引（关联作品 id）             |

### 4.3 骨架工坊（三表 + 异议表）

#### SkeletonSession（练习会话）

| 字段                                                     | 类型         | 说明                                          |
| -------------------------------------------------------- | ------------ | --------------------------------------------- |
| id / ownerId / createdAt / updatedAt / version / deleted | 通用         |                                               |
| skeletonId                                               | string       | null                                          | 所选名篇骨架 |
| currentDraft                                             | string       | **可变工作草稿**（防抖 500ms 落库，只存最新） |
| stage                                                    | `'selecting' | 'writing'                                     | 'evaluated'  | 'archived'` | 最近活动阶段，**可回退** |

#### DraftRevision（文本快照——不可变）

| 字段                                         | 类型   | 说明                                   |
| -------------------------------------------- | ------ | -------------------------------------- |
| id / ownerId / createdAt / version / deleted | 通用   |                                        |
| sessionId                                    | string | 所属会话（索引）                       |
| seq                                          | number | 会话内自增版本号（v1, v2, v3…）        |
| content                                      | string | **评价发生瞬间冻结的全文**（永远不改） |
| wordCount                                    | number | 冻结时计算                             |
| parentRevisionId                             | string | null                                   | 上一版快照（diff/进化对比） |
| changedNote                                  | string | null                                   | 用户本轮修改说明（可选）    |

#### SkeletonEvaluation（AI 评价——追加式）

| 字段                                         | 类型                      | 说明                                                                                           |
| -------------------------------------------- | ------------------------- | ---------------------------------------------------------------------------------------------- |
| id / ownerId / createdAt / version / deleted | 通用                      |                                                                                                |
| sessionId                                    | string                    | 所属会话                                                                                       |
| revisionId                                   | string                    | 指向被评价的 DraftRevision（外键）                                                             |
| round                                        | number                    | 同一 revision 内第几轮评价（1, 2, 3…）                                                         |
| basedOnReplyIds                              | string[]                  | 本轮评价回应了哪些用户异议                                                                     |
| changeSummary                                | string                    | null                                                                                           | 相比上一轮"修正了什么"（AI 自述） |
| author                                       | `'ai'`                    | 评价者（预留 `'user'` 支持互评）                                                               |
| payload                                      | SkeletonEvaluationPayload | matchRate / isElasticVariation / nodeAlignment[] / sevenElementDiff[] / suggestions[] / passed |

#### EvaluationReply（用户异议——追加式）

| 字段                                         | 类型     | 说明                      |
| -------------------------------------------- | -------- | ------------------------- |
| id / ownerId / createdAt / version / deleted | 通用     |                           |
| sessionId                                    | string   | 所属会话                  |
| revisionId                                   | string   | 针对哪版文本              |
| evaluationId                                 | string   | 针对哪次 AI 评价          |
| suggestionIndex                              | number   | null                      | 针对第几条建议（null = 针对整体） |
| content                                      | string   | 异议内容（不同建议/理由） |
| stance                                       | `'agree' | 'disagree'                | 'partial'`                        | 态度标记 |

> **派生（不落库）**：`firstEvaluation` = seq 最小快照的评价；`currentEvaluation` = 最新快照的评价；`matchHistory` = 按 seq 排序的 matchRate 序列；`effectiveEvaluation` = 同 revision 下 round 最大的评价；`acceptedCount` = 某评价下 stance=agree 的异议数。

### 4.4 作者眼（四表）

#### JurySession（练习会话）

| 字段                                                       | 类型       | 说明                     |               |              |                                     |
| -------------------------------------------------------- | -------- | ---------------------- | ------------- | ------------ | ----------------------------------- |
| id / ownerId / createdAt / updatedAt / version / deleted | 通用       |                        |               |              |                                     |
| seed                                                     | string   | 初始种子                   |               |              |                                     |
| stage                                                    | `'story' | 'criticized'           | 'adjudicated' | 'finalized'` | 最近活动阶段，**允许回退**（改批判→回退到 criticized） |
| preview                                                  | string   | 最新微小说前 80 字符（**列表投影**） |               |              |                                     |
| storyCount / criticismCount / adjudicationCount          | number   | 迭代计数（**投影**，写时更新）      |               |              |                                     |
| lastActivityAt                                           | number   | 列表倒序排序游标（**投影**）       |               |              |                                     |

> 投影字段随三表写入同步更新；远端同步后可由服务端聚合重建，**投影不是事实来源**，详情页永远以三表为准。

#### StorySnapshot（微小说快照——不可变）

| 字段                                         | 类型   | 说明                    |
| -------------------------------------------- | ------ | ----------------------- |
| id / ownerId / createdAt / version / deleted | 通用   |                         |
| sessionId                                    | string | 所属会话                |
| seq                                          | number | 版本号（重新生成即 +1） |
| content                                      | string | 微小说全文（冻结）      |
| seed                                         | string | 生成该版所用的种子      |

#### CriticismRevision（批判修订——不可变）

| 字段                                         | 类型   | 说明                        |
| -------------------------------------------- | ------ | --------------------------- |
| id / ownerId / createdAt / version / deleted | 通用   |                             |
| sessionId                                    | string | 所属会话                    |
| storySnapshotId                              | string | 针对哪一版微小说（外键）    |
| seq                                          | number | 批判版本号（修改重提即 +1） |
| content                                      | string | 批判全文（冻结）            |

#### Adjudication（裁决——追加式）

| 字段                                         | 类型               | 说明                                                                                                  |
| -------------------------------------------- | ------------------ | ----------------------------------------------------------------------------------------------------- |
| id / ownerId / createdAt / version / deleted | 通用               |                                                                                                       |
| sessionId                                    | string             | 所属会话                                                                                              |
| criticismId                                  | string             | 指向被裁决的 CriticismRevision（外键）                                                                |
| round                                        | `'initial'         | 'rebuttal'`                                                                                           | 首轮 / 反驳后二次仲裁               |
| rebuttal                                     | string             | null                                                                                                  | 用户反驳文本（round='rebuttal' 时） |
| payload                                      | JuryVerdictPayload | verdict(valid/invalid/undetermined) / confidence / intuition / evidence / reasoning / counterQuestion |

### 4.5 作品集（版本链）

#### PortfolioWork

| 字段                                                                      | 类型                     | 说明                   |
| ------------------------------------------------------------------------- | ------------------------ | ---------------------- |
| id / ownerId / createdAt / updatedAt / version / deleted                  | 通用                     |                        |
| title / currentContent / wordCount                                        | string / string / number | 标题 / 最新正文 / 字数 |
| status                                                                    | `'draft'                 | 'published'            | 'revising'` | 作品状态 |
| matchRate / publishableRubric? / sourceSkeletonId? / sourceSkeletonTitle? | —                        | 沿用现有结构           |
| submittedAt                                                               | ISO                      | 提交时间               |

#### WorkVersion（版本历史——增量）

| 字段                                                          | 类型     | 说明                                 |
| ------------------------------------------------------------- | -------- | ------------------------------------ |
| id / content / wordCount / matchRate? / userNote? / createdAt | 现有字段 | 版本全文与元数据                     |
| **parentVersionId**                                           | string   | null                                 | 上一版本（v1 为 null），构成版本链 |
| **changeSummary**                                             | string[] | 修改点摘要（列表展示用，归档时生成） |
| **relatedEvaluationIds**                                      | string[] | 关联的评价 id（建议→修改的可追溯）   |

---

## 5. 骨架工坊：AI 评价互动设计

### 5.1 交互模型（评价线程）

把"评价"从一次性事件升级为**评价线程（Evaluation Thread）**：同一版文本快照下，AI 评价与用户回应构成一条可追加的对话链。

```
文本 v2 ── AI 评价(round 1)
              ├─ 用户：采纳建议 ②③
              ├─ 用户：对建议 ① 提出异议「开头不需要补环境描写，因为……」
              │      └─ AI 重新评价(round 2，仅针对异议点，changeSummary 注明修正)
              └─ 用户：再次异议 / 采纳 / 进入修改
```

**关键规则**：

1. 评价与异议都挂在**同一条 DraftRevision** 上；只有"修改草稿"才产生新 Revision；
2. 新评价必须引用它所回应的异议（`basedOnReplyIds`），形成可追溯链；
3. 每次重新评价都是追加，不覆盖旧评价。

### 5.2 状态机

```
writing ──点击评价──▶ evaluated
   ▲                     │
   │ 修改草稿             ├─ 用户异议 ──▶ AI 重新评价（round+1，仍 evaluated）
   └─────────────────────┘
evaluated ──点击归档──▶ archived（快照→WorkVersion，带上采纳清单）
```

### 5.3 UI 布局（评价区）

- **评价卡（AI）**：matchRate + 建议列表；每条建议带「采纳」「有异议」按钮；
- **异议输入**：点击"有异议"展开文本域（预填定位信息），提交后显示用户气泡；
- **重新评价**：出现异议后显示「带着我的意见重新评价」；新评价顶部显示 changeSummary；
- **历史折叠**：旧轮次评价默认折叠，可展开。

---

## 6. 作者眼：会话列表与详情

### 6.1 列表（`/jury` 模块下方）

- 列表项：种子（标题）+ 阶段徽标（写作中/已批判/已裁决/已完成）+ 最近更新时间 + 迭代次数 + 微小说首行预览（`preview`）；
- 点击 → `/jury/:sessionId` 详情；
- 「开始新练习」创建新会话进入练习区；
- 空态引导文案；游标分页（每页 10 条）。

### 6.2 详情（`/jury/:sessionId`）——迭代时间线

- **最新状态卡**（默认展开）：最新微小说 + 最新批判 → 最新裁决；
- **迭代时间线**（倒序）：微小说 v1 → 批判 v1 → 裁决 initial → 反驳 → 二次仲裁 → 批判 v2 → 裁决 initial …；
- 每个节点**点击展开**显示具体文本（微小说全文 / 批判全文 / 反驳全文 / 裁决四要素）；
- 数据源：session + 三表按 `(sessionId, seq/createdAt)` 一次聚合。

---

## 7. 作品集：最新正文 + 版本历史

### 7.1 展示结构（`/portfolio/:id`）

- **正文区**：展示最新版本全文（标注版本号、日期、字数）；
- **修改历史**（倒序列表）：每项显示"版本号 + 日期 + 修改点摘要"，点击展开**版本详情面板**：
  - 该版本全文；
  - 与上一版差异视图（新增绿/删除红/改写黄）；
  - 关联建议（本版改动对应哪些评价的哪几条建议，可跳转回顾评价原文）。

### 7.2 修改点生成策略（归档时一次性生成，列表零成本）

1. **采纳清单**：用户点过"采纳"的建议 → 修改点文案（"采纳：补全'转折'节点"）；
2. **用户说明**：归档时手填一句话（可选）；
3. **文本 diff 兜底**：新增/删除/改写统计 → 机器摘要（"3 处新增 / 2 处删除"）。

> diff 只用于详情面板的**实时计算**（客户端 diff，不落库）；落库的只有 changeSummary 摘要。

---

## 8. 索引与查询设计

| 表                                                | 索引                                  | 用途                         |
| ------------------------------------------------- | ------------------------------------- | ---------------------------- |
| DraftRevision / StorySnapshot / CriticismRevision | `(sessionId, seq)`                    | 会话内按版本顺序取进化轨迹   |
| SkeletonEvaluation                                | `(sessionId, revisionId)`             | 按快照取评价；按会话取评价链 |
| EvaluationReply                                   | `(sessionId, evaluationId)`           | 按评价取异议                 |
| Adjudication                                      | `(sessionId, criticismId, round)`     | 按批判取裁决                 |
| JurySession（列表）                               | `(ownerId, lastActivityAt)`           | 会话列表倒序 + 游标分页      |
| WorkVersion                                       | 嵌入 PortfolioWork，按 createdAt 倒序 | 版本历史                     |
| 全部实体                                          | `ownerId`、`updatedAt`                | 用户隔离 + 增量同步游标      |

---

## 9. 存储介质分配

| 实体                                                                                                                                           | 介质         | 理由                         |
| ---------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | ---------------------------- |
| UserAccount / Session / UserProfile / MaterialCard / GrowthRecord / LearningProgress / SkeletonSession·JurySession（元数据+currentDraft 若短） | localStorage | 轻量、高频读                 |
| DraftRevision / SkeletonEvaluation / EvaluationReply / StorySnapshot / CriticismRevision / Adjudication / PortfolioWork·WorkVersion            | IndexedDB    | 全文、不可变快照、追加式日志 |

> 推荐终极形态：所有"文本类 + 评价类"实体全部入 IndexedDB，localStorage 只留账号/会话/偏好；同步层未来只对接 IndexedDB 数据源。

---

## 10. 边界与性能策略

| 场景             | 处理                                                             |
| ---------------- | ---------------------------------------------------------------- |
| 列表空态         | 引导文案（"完成第一次批判练习后…"/"暂无历史版本"）               |
| 无历史版本的初稿 | 版本区显示"暂无历史版本"                                         |
| 会话/版本删除    | 会话级墓碑删除，子表级联标记；快照与评价永不物理删除             |
| diff 缺父版本    | 无 parent 时降级为纯全文展示，不做 diff                          |
| 版本过多         | 历史区默认折叠"最近 5 版 + 查看全部"，分页加载全文               |
| 会话过多         | 游标分页；本地 >500 条提示归档                                   |
| 长文本性能       | 列表一律用投影字段；全文仅在详情/展开时读取；diff 仅详情按需计算 |
| 输入中断电       | 草稿防抖 500ms 落库；评价/异议提交即持久化（追加式不可变原则）   |

---

## 11. 现有代码/数据迁移映射

| 现有状态                                                             | 迁移目标                                                                                                            |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| jury store：seed/microStory/criticism/verdict/rebuttal/secondVerdict | 拆入 JurySession + StorySnapshot(seq=1) + CriticismRevision(seq=1) + Adjudication×2                                 |
| skeleton store：draft/firstEvaluation/currentEvaluation/matchHistory | draft→SkeletonSession.currentDraft；评价槽位→DraftRevision + SkeletonEvaluation 链（matchHistory 由链推导，不迁移） |
| `wt_user / wt_material / wt_progress`、IndexedDB `works`             | 补 ownerId/version/deleted 默认值，不动结构                                                                         |
| 登录/会话（mock）                                                    | 维持现状，接后端后废弃 UserAccount/Session                                                                          |

---

## 12. 分阶段实施建议

| 阶段 | 内容                                                      | 风险                       |
| ---- | --------------------------------------------------------- | -------------------------- |
| P0   | 骨架工坊评价互动（EvaluationReply + 多轮评价 + 重新评价） | 中（状态机与 UI 改动最大） |
| P0   | 作者眼会话列表 + 详情时间线（投影 + 路由拆分）            | 中                         |
| P1   | 作品集版本历史（changeSummary 生成 + 列表 + diff 详情）   | 低（依赖 P0 的采纳清单）   |
| P2   | 列表分页、懒加载、删除墓碑、远端同步接口预留              | 低                         |

---

## 13. 变更记录

| 版本 | 日期    | 内容                                                                                                                                                                                                                                                                                         |
| ---- | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| v1.0 | 2026-08 | 初版：通用字段、实体定义、介质分配                                                                                                                                                                                                                                                           |
| v1.1 | 2026-08 | 评审修订：文本/评价分表、版本化快照、改后再评迭代链、派生字段约定                                                                                                                                                                                                                            |
| v1.2 | 2026-08 | 合并：骨架工坊 AI 评价互动（EvaluationReply/多轮评价）、作者眼会话列表与详情时间线（JurySession 投影）、作品集版本历史（WorkVersion 版本链/修改点/diff）                                                                                                                                     |
| v1.3 | 2026-08 | **已实现**：存储抽象层（src/utils/store/，IDataStore + localStorage/IndexedDB 双实现 + RemoteStore 占位）、骨架工坊/作者眼四表落库、草稿防抖持久化、评价互动（采纳/异议/重新评价）、作者眼会话列表与 /jury/:id 详情时间线、作品集版本历史与 diff、墓碑删除与分页。执行进度见 doc/开发计划.md |
