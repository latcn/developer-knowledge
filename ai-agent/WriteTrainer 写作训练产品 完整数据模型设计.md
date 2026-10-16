
## 一、数据模型设计原则与命名规范

### 命名规范

| 规范项      | 规则                               | 示例                                        |
| :------- | :------------------------------- | :---------------------------------------- |
| **主键**   | 统一使用 `id`，类型为 UUID               | `id UUID PRIMARY KEY`                     |
| **业务编码** | 使用 `_code` 后缀，表示业务唯一标识           | `skill_code`, `level_code`, `rank_code`   |
| **外键**   | 直接引用目标表的字段名                      | `user_id`, `rank_code`                    |
| **时间戳**  | 统一使用 `created_at` / `updated_at` | TIMESTAMP WITH DEFAULT NOW()              |
| **布尔字段** | 使用 `is_` 前缀                      | `is_active`, `is_favorite`, `is_verified` |
| **索引命名** | `idx_表名_字段名`                     | `idx_training_records_user_id`            |

### 数据类型映射

| 类型 | 数据库类型 | 说明 |
| :--- | :--- | :--- |
| `id` / 主键 / 外键 | UUID | 通用唯一标识 |
| `_code` 字段 | VARCHAR(20-50) | 业务编码 |
| 短文本 | VARCHAR(30-200) | 标题、名称、标签 |
| 长文本 | TEXT | 内容、建议、反思 |
| 评分字段 | DECIMAL(5,2) / INT | 支持0-10或0-100 |
| 时间戳 | TIMESTAMP WITH TIME ZONE | 所有时间字段 |
| JSON 字段 | JSONB | 仅在必须存储半结构化数据时使用 |
| 向量 | VECTOR(384) | pgvector 扩展，用于 RAG |


## 二、核心用户与进度数据（3张表）


### 2.1 用户表（`users`）

| 字段名                    | 类型           | 约束                    | 说明                                                     |
| :--------------------- | :----------- | :-------------------- | :----------------------------------------------------- |
| `id`                   | UUID         | PRIMARY KEY           | 用户唯一标识                                                 |
| `username`             | VARCHAR(50)  | NOT NULL, UNIQUE      | 用户名，用于登录和展示                                            |
| `email`                | VARCHAR(100) | UNIQUE                | 邮箱（云端版使用，单机版可空）                                        |
| `password`             | VARCHAR(255) |                       | 密码（单机版可空）                                              |
| `current_rank_code`    | VARCHAR(50)  | FK→`rank_config.code` | 当前最高段位编码                                               |
| `total_trainings`      | INT          | DEFAULT 0             | 总训练次数（专项+段位）                                           |
| `rank_works_count`     | INT          | DEFAULT 0             | 段位练习完成作品数                                              |
| `consecutive_days`     | INT          | DEFAULT 0             | 连续训练天数                                                 |
| `last_training_at`     | TIMESTAMP    |                       | 最后训练时间                                                 |
| `style_intent`         | VARCHAR(20)  |                       | 用户风格意图：`minimalist`/`detailed`/`neutral`，AI评价时据此跳过对应维度 |
| `skip_dimensions`      | VARCHAR(200) |                       | 跳过的检测维度，逗号分隔：`sensory,emotion`                         |
| `notification_enabled` | BOOLEAN      | DEFAULT TRUE          | 通知开关                                                   |
| `daily_goal`           | INT          | DEFAULT 1             | 每日训练目标                                                 |
| `created_at`           | TIMESTAMP    | DEFAULT NOW()         | 创建时间                                                   |
| `updated_at`           | TIMESTAMP    | DEFAULT NOW()         | 更新时间                                                   |


### 2.2 用户专项进度表（`user_skill_progress`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `user_id` | UUID | NOT NULL, FK→`users.id` | 用户 ID |
| `skill_code` | VARCHAR(20) | NOT NULL | 专项编码：`logic`/`sensory`/`emotion`/`expression`/`creativity` |
| `current_level_code` | VARCHAR(10) | NOT NULL | 当前等级编码：`Lv.0-1`/`Lv.2-3`/`Lv.4+` |
| `current_score` | DECIMAL(5,2) | DEFAULT 0 | 当前综合得分（滑动窗口平均，0-100） |
| `total_trainings` | INT | DEFAULT 0 | 该专项总训练次数 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | 创建时间 |
| `updated_at` | TIMESTAMP | DEFAULT NOW() | 更新时间 |
| `UNIQUE KEY` | (`user_id`, `skill_code`) | | 联合唯一索引 |
| `INDEX` | (`user_id`) | | 用户查询索引 |


### 2.3 专项得分历史表（`skill_score_history`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `user_id` | UUID | NOT NULL, FK→`users.id` | 用户 ID |
| `skill_code` | VARCHAR(20) | NOT NULL | 专项编码 |
| `record_id` | UUID | FK→`training_records.id` | 关联的训练记录 |
| `score` | DECIMAL(5,2) | NOT NULL | 本次训练得分（0-10） |
| `level_code_at_training` | VARCHAR(10) | NOT NULL | 训练时的等级编码快照 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | 创建时间 |
| `INDEX` | (`user_id`, `skill_code`, `created_at` DESC) | | 组合索引，支持趋势图查询 |


## 三、配置与主数据（2张表）


### 3.1 专项等级配置表（`skill_level_config`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `skill_code` | VARCHAR(20) | NOT NULL | 专项编码 |
| `level_code` | VARCHAR(10) | NOT NULL | 等级编码：`Lv.0-1`/`Lv.2-3`/`Lv.4+` |
| `level_order` | INT | NOT NULL | 排序：0/1/2 |
| `level_name` | VARCHAR(20) | NOT NULL | 显示名称：`初阶`/`进阶`/`高阶` |
| `min_score` | INT | NOT NULL | 等级映射最低分（0-100），当前得分 ≥ `min_score` 且 < `max_score` 时映射到该等级 |
| `max_score` | INT | NOT NULL | 等级映射最高分（0-100），当前得分 ≥ `min_score` 且 < `max_score` 时映射到该等级 |
| `description` | TEXT | | 能力描述 |
| `min_training_count` | INT | NOT NULL | **晋升最少训练次数**：在该等级下至少完成 N 次训练才能晋升 |
| `min_avg_score` | INT | NOT NULL | **晋升平均分门槛**：最近 N 次训练的平均分（0-100）≥ 该值 |
| `stability_check_count` | INT | NOT NULL | **稳定性校验次数**：检查最近 N 次训练是否均达标 |
| `stability_min_score` | INT | NOT NULL | **稳定性最低分**：最近 N 次训练的单次得分（0-100）必须 ≥ 该值 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | 创建时间 |
| `updated_at` | TIMESTAMP | DEFAULT NOW() | 更新时间 |
| `UNIQUE KEY` | (`skill_code`, `level_code`) | | 联合唯一索引 |


### 3.2 段位配置表（`rank_config`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `code` | VARCHAR(50) | NOT NULL, UNIQUE | 段位编码：`rank_story_newbie` |
| `name` | VARCHAR(30) | NOT NULL | 段位名称：`故事新手` |
| `icon` | VARCHAR(10) | NOT NULL | 图标 Emoji：`🌟` |
| `rank_order` | INT | NOT NULL | 排序（决定晋升路径） |
| `required_works` | INT | NOT NULL | 晋升所需作品数 |
| `skill_thresholds` | JSONB | | **技能门槛**：`{"logic":"Lv.2-3", "sensory":"Lv.0-1"}` |
| `unlocks` | JSONB | | **解锁内容**：`["微故事证书", "矛盾放大训练"]` |
| `estimated_time` | VARCHAR(20) | | 预计耗时：`3-5天` |
| `enabled` | BOOLEAN | DEFAULT TRUE | 是否启用 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | 创建时间 |
| `updated_at` | TIMESTAMP | DEFAULT NOW() | 更新时间 |

**注**：`skill_thresholds` 和 `unlocks` 使用 JSONB 存储，因其结构固定且无需独立查询，可避免多表关联带来的复杂性。


## 四、训练相关数据（3张表）


### 4.1 题目表（`questions`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `skill_code` | VARCHAR(20) | | 关联专项编码（专项训练题必填） |
| `title` | VARCHAR(100) | NOT NULL | 题目标题 |
| `scene` | TEXT | NOT NULL | 灵魂场景（题目主体描述） |
| `type` | VARCHAR(10) | NOT NULL | `skill`（专项训练）/`rank`（段位练习） |
| `is_active` | BOOLEAN | DEFAULT TRUE | 是否启用 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | 创建时间 |
| `updated_at` | TIMESTAMP | DEFAULT NOW() | 更新时间 |
| `INDEX` | (`skill_code`, `type`) | | 组合索引 |


### 4.2 题目等级配置表（`question_level_config`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `question_id` | UUID | NOT NULL, FK→`questions.id` | 题目 ID |
| `level_code` | VARCHAR(10) | NOT NULL | 等级编码 |
| `requirements` | TEXT | NOT NULL | 该等级的训练要求 |
| `max_length` | INT | NOT NULL | 字数上限 |
| `difficulty` | INT | DEFAULT 1 | 难度系数 1-5 |
| `UNIQUE KEY` | (`question_id`, `level_code`) | | 联合唯一索引 |


### 4.3 训练记录表（`training_records`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `user_id` | UUID | NOT NULL, FK→`users.id` | 用户 ID |
| `type` | VARCHAR(10) | NOT NULL | `skill`/`rank` |
| `skill_code` | VARCHAR(20) | NOT NULL | 关联专项编码 |
| `question_id` | UUID | FK→`questions.id` | 关联题目 ID |
| `level_code_at_training` | VARCHAR(10) | NOT NULL | 训练时等级编码快照 |
| `title` | VARCHAR(100) | | 作品标题（段位练习使用） |
| `original_content` | TEXT | NOT NULL | 原始提交内容（Markdown） |
| `revised_content` | TEXT | | 当前最新修改内容 |
| `word_count` | INT | DEFAULT 0 | 字数 |
| `final_score` | DECIMAL(5,2) | | 综合得分（0-10） |
| `passed` | BOOLEAN | | 是否通过（≥ 7 分） |
| `status` | VARCHAR(20) | DEFAULT 'pending' | `pending`/`evaluated`/`passed`/`failed`/`revising`/`cooldown`/`completed` |
| `user_note` | TEXT | | 用户反思 |
| `submitted_at` | TIMESTAMP | DEFAULT NOW() | 首次提交时间 |
| `updated_at` | TIMESTAMP | DEFAULT NOW() | 最后更新时间 |
| `INDEX` | (`user_id`, `type`, `status`) | | 组合索引 |
| `INDEX` | (`user_id`, `skill_code`) | | 组合索引 |
| `INDEX` | (`submitted_at` DESC) | | 时间倒序索引 |


## 五、评价相关数据（3张表）


### 5.1 评价报告表（`evaluation_reports`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `record_id` | UUID | NOT NULL, UNIQUE, FK→`training_records.id` | 训练记录 ID |
| `rule_score` | DECIMAL(5,2) | | 规则层得分（0-10） |
| `semantic_score` | DECIMAL(5,2) | | 语义层得分（0-10） |
| `penalty` | DECIMAL(4,2) | DEFAULT 0 | 减分项（0-3） |
| `rule_sensory_coverage` | INT | | 五感覆盖数（0-5） |
| `rule_emotion_density` | DECIMAL(3,2) | | 情绪词密度（每百字出现次数） |
| `rule_blank_complete` | BOOLEAN | | 填空是否完整 |
| `rule_readability` | VARCHAR(10) | | `good`/`ok`/`poor` |
| `rule_dialogue_diversity` | DECIMAL(3,2) | | 对话标签多样性 |
| `semantic_fluency` | INT | | 流畅度（0-10） |
| `semantic_emotion` | INT | | 情感传递（0-10） |
| `semantic_specificity` | INT | | 具体度（0-10） |
| `semantic_coherence` | INT | | 连贯性（0-10） |
| `semantic_subtext` | INT | | 潜台词（0-10） |
| `penalty_reason` | VARCHAR(200) | | 扣分原因 |
| `penalty_evidence` | TEXT | | 扣分证据原文片段 |
| `penalty_evidence_type` | VARCHAR(30) | | `sensory_overload`/`emotion_empty`/`logic_break`/`expression_repetitive`/`creativity_template` |
| `penalty_verified` | BOOLEAN | | 是否通过系统二次验证 |
| `encouragement` | TEXT | | 鼓励语 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | 创建时间 |
| `INDEX` | (`record_id`) | | 唯一索引 |


### 5.2 评价建议表（`evaluation_suggestions`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `report_id` | UUID | NOT NULL, FK→`evaluation_reports.id` | 评价报告 ID |
| `aspect` | VARCHAR(50) | | 维度名称 |
| `suggestion_text` | TEXT | NOT NULL | 建议内容 |
| `quote_text` | TEXT | | 原文引用 |
| `priority` | INT | DEFAULT 1 | 优先级（1-3） |
| `display_order` | INT | DEFAULT 0 | 显示顺序 |
| `INDEX` | (`report_id`, `display_order`) | | 组合索引 |


### 5.3 七要素自检报告表（`seven_element_reports`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `record_id` | UUID | NOT NULL, UNIQUE, FK→`training_records.id` | 训练记录 ID |
| `plot_score` | INT | | 情节（0-10） |
| `plot_note` | TEXT | | 情节备注 |
| `structure_score` | INT | | 结构（0-10） |
| `structure_note` | TEXT | | 结构备注 |
| `character_score` | INT | | 人物（0-10） |
| `character_note` | TEXT | | 人物备注 |
| `scene_score` | INT | | 场景（0-10） |
| `scene_note` | TEXT | | 场景备注 |
| `dialogue_score` | INT | | 对话（0-10） |
| `dialogue_note` | TEXT | | 对话备注 |
| `style_score` | INT | | 风格（0-10） |
| `style_note` | TEXT | | 风格备注 |
| `theme_score` | INT | | 主题（0-10） |
| `theme_note` | TEXT | | 主题备注 |
| `overall_score` | DECIMAL(3,1) | | 综合自评分（0-10） |
| `completed_at` | TIMESTAMP | DEFAULT NOW() | 完成时间 |
| `INDEX` | (`record_id`) | | 唯一索引 |


## 六、修改迭代数据（1张表）


### 6.1 训练迭代表（`training_iterations`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `record_id` | UUID | NOT NULL, FK→`training_records.id` | 关联训练记录 |
| `iteration_number` | INT | NOT NULL | 第几次修改（从 1 开始） |
| `content` | TEXT | NOT NULL | 本次修改的内容 |
| `word_count` | INT | DEFAULT 0 | 字数 |
| `user_note` | TEXT | | 用户修改反思 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | 创建时间 |
| `UNIQUE KEY` | (`record_id`, `iteration_number`) | | 联合唯一索引 |


## 七、晋升相关数据（2张表）


### 7.1 统一晋升记录表（`promotion_records`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `user_id` | UUID | NOT NULL, FK→`users.id` | 用户 ID |
| `promotion_type` | VARCHAR(20) | NOT NULL | `skill_level`/`rank` |
| `skill_code` | VARCHAR(20) | | 专项编码（专项晋升时使用） |
| `rank_code` | VARCHAR(50) | FK→`rank_config.code` | 段位编码（段位晋升时使用） |
| `old_level_code` | VARCHAR(10) | | 原等级编码（专项晋升时使用） |
| `new_level_code` | VARCHAR(10) | | 新等级编码（专项晋升时使用） |
| `trigger_record_id` | UUID | FK→`training_records.id` | 触发训练记录 ID |
| `trigger_type` | VARCHAR(20) | NOT NULL | `auto`/`manual_confirm` |
| `required_works_snapshot` | INT | | 晋升时作品数量要求快照 |
| `skill_thresholds_snapshot` | VARCHAR(200) | | 技能门槛快照，格式：`logic:Lv.2-3,sensory:Lv.0-1` |
| `user_confirmed_at` | TIMESTAMP | | 用户确认时间（手动晋升时） |
| `created_at` | TIMESTAMP | DEFAULT NOW() | 创建时间 |
| `INDEX` | (`user_id`, `promotion_type`, `created_at` DESC) | | 组合索引 |


### 7.2 用户段位记录表（`user_ranks`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `user_id` | UUID | NOT NULL, FK→`users.id` | 用户 ID |
| `rank_code` | VARCHAR(50) | NOT NULL, FK→`rank_config.code` | 段位编码 |
| `promotion_record_id` | UUID | FK→`promotion_records.id` | 关联晋升记录 |
| `promoted_at` | TIMESTAMP | DEFAULT NOW() | 晋升时间 |
| `UNIQUE KEY` | (`user_id`, `rank_code`) | | 联合唯一索引 |


## 八、素材库与知识库数据（6张表）


### 8.1 知识卡片表（`knowledge_cards`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `title` | VARCHAR(100) | NOT NULL | 卡片标题 |
| `content` | TEXT | NOT NULL | 卡片内容（Markdown） |
| `skill_code` | VARCHAR(20) | NOT NULL | 专项编码：`logic`/`sensory`/`emotion`/`expression`/`creativity` |
| `level_code` | VARCHAR(10) | | 适用等级编码（可为空，表示通用） |
| `type` | VARCHAR(20) | | `observation`/`example`/`theory` |
| `teaching_goal` | VARCHAR(200) | | 教学目标 |
| `applicable_scene` | VARCHAR(100) | | 适用场景：`training_stuck`/`pre_writing`/`revision_guide` |
| `example_ref` | VARCHAR(200) | | 经典参照 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | 创建时间 |
| `updated_at` | TIMESTAMP | DEFAULT NOW() | 更新时间 |
| `INDEX` | (`skill_code`, `level_code`) | | 组合索引 |


### 8.2 题目-知识卡片关联表（`question_knowledge`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `question_id` | UUID | NOT NULL, FK→`questions.id` | 题目 ID |
| `level_code` | VARCHAR(10) | NOT NULL | 等级编码 |
| `knowledge_card_id` | UUID | NOT NULL, FK→`knowledge_cards.id` | 知识卡片 ID |
| `display_order` | INT | DEFAULT 0 | 显示顺序 |
| `UNIQUE KEY` | (`question_id`, `level_code`, `knowledge_card_id`) | | 联合唯一索引 |


### 8.3 系统素材库表（`material_library`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `dimension` | VARCHAR(20) | NOT NULL | `character`/`conflict`/`sensory`/`world`/`narrative`/`dialogue` |
| `title` | VARCHAR(100) | NOT NULL | 素材标题 |
| `content` | TEXT | NOT NULL | 素材正文 |
| `source_type` | VARCHAR(20) | NOT NULL | `canon`/`classic`/`expert`/`ai` |
| `source_name` | VARCHAR(100) | | 来源名称 |
| `source_author` | VARCHAR(50) | | 来源作者 |
| `source_ref_book` | VARCHAR(100) | | 出处书籍/作品 |
| `source_verified` | BOOLEAN | DEFAULT FALSE | 是否核实 |
| `source_verified_by` | VARCHAR(50) | | 核实人 |
| `difficulty` | VARCHAR(20) | | `beginner`/`intermediate`/`advanced` |
| `level_code` | VARCHAR(10) | | 适用等级编码 |
| `usage_count` | INT | DEFAULT 0 | 使用次数 |
| `embedding` | VECTOR(384) | | 向量（pgvector），用于语义检索 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | 创建时间 |
| `updated_at` | TIMESTAMP | DEFAULT NOW() | 更新时间 |
| `INDEX` | (`dimension`, `difficulty`) | | 组合索引 |


### 8.4 标签主数据表（`material_tags`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `tag` | VARCHAR(30) | NOT NULL, UNIQUE | 标签名，全局唯一 |
| `usage_count` | INT | DEFAULT 0 | 被引用次数 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | 创建时间 |
| `updated_at` | TIMESTAMP | DEFAULT NOW() | 更新时间 |
| `INDEX` | (`tag`) | | 标签查询索引 |


### 8.5 系统素材-标签关联表（`material_library_tags`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `material_id` | UUID | NOT NULL, FK→`material_library.id` | 素材 ID |
| `tag_id` | UUID | NOT NULL, FK→`material_tags.id` | 标签 ID |
| `UNIQUE KEY` | (`material_id`, `tag_id`) | | 联合唯一索引 |


### 8.6 用户素材表（`user_materials`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `user_id` | UUID | NOT NULL, FK→`users.id` | 用户 ID |
| `source_material_id` | UUID | FK→`material_library.id` | 来源素材 ID |
| `title` | VARCHAR(100) | NOT NULL | 用户自定义标题 |
| `content` | TEXT | NOT NULL | 素材内容 |
| `blanks` | VARCHAR(500) | | 留白位置标注，逗号分隔 |
| `dimension` | VARCHAR(20) | NOT NULL | 所属维度 |
| `genre` | VARCHAR(20) | | 题材 |
| `source_type` | VARCHAR(20) | NOT NULL | `search`/`generate`/`generate_combined`/`user_defined` |
| `source_dialogue_id` | UUID | FK→`dialogue_sessions.id` | 来源对话会话 ID |
| `usage_count` | INT | DEFAULT 0 | 在训练中被使用次数 |
| `is_favorite` | BOOLEAN | DEFAULT FALSE | 是否收藏 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | 创建时间 |
| `updated_at` | TIMESTAMP | DEFAULT NOW() | 更新时间 |
| `INDEX` | (`user_id`, `dimension`) | | 组合索引 |


### 8.7 用户素材-标签关联表（`user_material_tags`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `user_material_id` | UUID | NOT NULL, FK→`user_materials.id` | 用户素材 ID |
| `tag_id` | UUID | NOT NULL, FK→`material_tags.id` | 标签 ID |
| `UNIQUE KEY` | (`user_material_id`, `tag_id`) | | 联合唯一索引 |


## 九、对话与反馈数据（3张表）


### 9.1 对话会话表（`dialogue_sessions`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `user_id` | UUID | NOT NULL, FK→`users.id` | 用户 ID |
| `status` | VARCHAR(20) | DEFAULT 'collecting' | `collecting`/`searching`/`generating`/`confirming`/`completed` |
| `genre` | VARCHAR(20) | | 题材 |
| `dimension` | VARCHAR(20) | | 素材维度 |
| `theme` | VARCHAR(100) | | 核心主题 |
| `key_elements` | VARCHAR(200) | | 核心元素，逗号分隔 |
| `rejected_directions` | VARCHAR(200) | | 已拒绝方向，逗号分隔 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | 创建时间 |
| `updated_at` | TIMESTAMP | DEFAULT NOW() | 更新时间 |
| `INDEX` | (`user_id`, `status`) | | 组合索引 |


### 9.2 对话消息表（`dialogue_messages`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `session_id` | UUID | NOT NULL, FK→`dialogue_sessions.id` | 关联会话 |
| `role` | VARCHAR(10) | NOT NULL | `user`/`assistant` |
| `content` | TEXT | NOT NULL | 消息内容 |
| `intent` | VARCHAR(20) | | 解析出的意图 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | 创建时间 |
| `INDEX` | (`session_id`, `created_at`) | | 组合索引 |


### 9.3 评价反馈表（`evaluation_feedback`）

| 字段名 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PRIMARY KEY | 唯一标识 |
| `record_id` | UUID | NOT NULL, FK→`training_records.id` | 关联训练记录 |
| `user_id` | UUID | NOT NULL, FK→`users.id` | 用户 ID |
| `suggestion_id` | UUID | FK→`evaluation_suggestions.id` | 关联的建议 ID |
| `feedback_type` | VARCHAR(10) | NOT NULL | `helpful`/`unhelpful` |
| `user_comment` | TEXT | | 用户附加评论 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | 创建时间 |
| `UNIQUE KEY` | (`record_id`, `suggestion_id`) | | 联合唯一索引 |
| `INDEX` | (`user_id`) | | 用户查询索引 |


## 十、表间关系总览

| # | 源表 | 目标表 | 关系 | 关联字段 |
| :--- | :--- | :--- | :--- | :--- |
| 1 | `users` | `user_skill_progress` | 1:N | `users.id` → `user_skill_progress.user_id` |
| 2 | `users` | `user_ranks` | 1:N | `users.id` → `user_ranks.user_id` |
| 3 | `users` | `training_records` | 1:N | `users.id` → `training_records.user_id` |
| 4 | `users` | `promotion_records` | 1:N | `users.id` → `promotion_records.user_id` |
| 5 | `users` | `user_materials` | 1:N | `users.id` → `user_materials.user_id` |
| 6 | `users` | `dialogue_sessions` | 1:N | `users.id` → `dialogue_sessions.user_id` |
| 7 | `users` | `evaluation_feedback` | 1:N | `users.id` → `evaluation_feedback.user_id` |
| 8 | `users` | `skill_score_history` | 1:N | `users.id` → `skill_score_history.user_id` |
| 9 | `rank_config` | `user_ranks` | 1:N | `rank_config.code` → `user_ranks.rank_code` |
| 10 | `rank_config` | `promotion_records` | 1:N | `rank_config.code` → `promotion_records.rank_code` |
| 11 | `questions` | `question_level_config` | 1:N | `questions.id` → `question_level_config.question_id` |
| 12 | `questions` | `question_knowledge` | 1:N | `questions.id` → `question_knowledge.question_id` |
| 13 | `questions` | `training_records` | 1:N | `questions.id` → `training_records.question_id` |
| 14 | `training_records` | `evaluation_reports` | 1:1 | `training_records.id` → `evaluation_reports.record_id` |
| 15 | `training_records` | `training_iterations` | 1:N | `training_records.id` → `training_iterations.record_id` |
| 16 | `training_records` | `seven_element_reports` | 1:1 | `training_records.id` → `seven_element_reports.record_id` |
| 17 | `training_records` | `skill_score_history` | 1:N | `training_records.id` → `skill_score_history.record_id` |
| 18 | `training_records` | `promotion_records` | 1:N | `training_records.id` → `promotion_records.trigger_record_id` |
| 19 | `training_records` | `evaluation_feedback` | 1:N | `training_records.id` → `evaluation_feedback.record_id` |
| 20 | `evaluation_reports` | `evaluation_suggestions` | 1:N | `evaluation_reports.id` → `evaluation_suggestions.report_id` |
| 21 | `knowledge_cards` | `question_knowledge` | 1:N | `knowledge_cards.id` → `question_knowledge.knowledge_card_id` |
| 22 | `material_library` | `material_library_tags` | 1:N | `material_library.id` → `material_library_tags.material_id` |
| 23 | `material_library` | `user_materials` | 1:N | `material_library.id` → `user_materials.source_material_id` |
| 24 | `material_tags` | `material_library_tags` | 1:N | `material_tags.id` → `material_library_tags.tag_id` |
| 25 | `material_tags` | `user_material_tags` | 1:N | `material_tags.id` → `user_material_tags.tag_id` |
| 26 | `user_materials` | `user_material_tags` | 1:N | `user_materials.id` → `user_material_tags.user_material_id` |
| 27 | `user_materials` | `dialogue_sessions` | N:1 | `user_materials.source_dialogue_id` → `dialogue_sessions.id` |
| 28 | `dialogue_sessions` | `dialogue_messages` | 1:N | `dialogue_sessions.id` → `dialogue_messages.session_id` |


## 十一、完整数据模型 ER 图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                  完整数据模型 ER 图                                 │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌──────────────┐          ┌──────────────────────┐          ┌──────────────────┐ │
│  │    users     │ 1      N │  user_skill_progress │          │ skill_level_config│ │
│  └──────────────┘ ──────── └──────────────────────┘          └──────────────────┘ │
│         │                        │                              │                   │
│         │1                       │                              │                   │
│         ▼                        │                              │                   │
│  ┌──────────────┐          ┌──────────────────────┐           │                   │
│  │  user_ranks  │          │   training_records   │───────────┘                   │
│  └──────────────┘          └──────────────────────┘                               │
│         │                        │1                                                 │
│         │                        ▼                                                 │
│         │                 ┌──────────────────────┐                                 │
│         │                 │ training_iterations  │                                 │
│         │                 └──────────────────────┘                                 │
│         │                        │                                                 │
│         │                        │1                                                │
│         │                        ▼                                                 │
│         │                 ┌──────────────────────┐                                 │
│         │                 │  evaluation_reports  │                                 │
│         │                 └──────────────────────┘                                 │
│         │                        │1                                                │
│         │                        ▼                                                 │
│         │                 ┌──────────────────────┐                                 │
│         │                 │evaluation_suggestions│                                 │
│         │                 └──────────────────────┘                                 │
│         │                                                                           │
│         ├──┐          ┌──────────────────────┐          ┌──────────────────────┐  │
│         │  │          │    questions         │──────────│ question_level_config│  │
│         │  │          └──────────────────────┘          └──────────────────────┘  │
│         ▼  ▼                    │                                                 │
│  ┌──────────────────────┐       │                                                  │
│  │  promotion_records   │       ▼                                                  │
│  └──────────────────────┘  ┌──────────────────────┐          ┌──────────────────┐ │
│                                    │              │          │  knowledge_cards │ │
│                                    │              └──────────└──────────────────┘ │
│                                    ▼                  │          │                  │
│                             ┌──────────────────────┐  │          ▼                  │
│                             │  question_knowledge  │  │   ┌──────────────────────┐ │
│                             └──────────────────────┘  │   │  material_library   │ │
│                                                        │   └──────────────────────┘ │
│  ┌──────────────┐          ┌──────────────────────┐  │          │                  │
│  │  rank_config │          │ seven_element_reports│  │          │                  │
│  └──────────────┘          └──────────────────────┘  │          ▼                  │
│                                                       │   ┌──────────────────────┐ │
│                                                       │   │ material_library_tags│ │
│                                                       │   └──────────────────────┘ │
│                                                       │          │                  │
│                                                       │          ▼                  │
│                                                       │   ┌──────────────────────┐ │
│                                                       │   │    material_tags    │ │
│                                                       │   └──────────────────────┘ │
│                                                       │                            │
│  ┌──────────────┐          ┌──────────────────────┐  │                            │
│  │user_materials│──────────│ user_material_tags   │  │                            │
│  └──────────────┘          └──────────────────────┘  │                            │
│         │                                             │                            │
│         │                                             │                            │
│  ┌──────┴──────┐          ┌──────────────────────┐  │                            │
│  │             │          │  dialogue_sessions   │  │                            │
│  │             │          └──────────────────────┘  │                            │
│  │             │                    │               │                            │
│  │             │                    ▼               │                            │
│  │             │          ┌──────────────────────┐  │                            │
│  │             │          │  dialogue_messages   │  │                            │
│  │             │          └──────────────────────┘  │                            │
│  │             │                                    │                            │
│  │             └────────────────────────────────────┘                            │
│  │                                                                                │
│  └──────────────────────────────┐                                                 │
│                                  │                                                 │
│                           ┌──────┴──────┐                                          │
│                           │             │                                          │
│                           ▼             ▼                                          │
│                   ┌────────────┐ ┌────────────┐                                   │
│                   │skill_score │ │evaluation  │                                   │
│                   │ _history   │ │ _feedback  │                                   │
│                   └────────────┘ └────────────┘                                   │
└─────────────────────────────────────────────────────────────────────────────────────┘
```


## 十二、核心数据流转


### 专项训练完成数据写入

```
用户提交训练
    ↓
training_records 新增 (status='pending', type='skill')
    ↓
联合评分 → evaluation_reports 新增 → evaluation_suggestions 批量新增
    ↓
training_records 更新 (status='evaluated', final_score, passed)
    ↓
skill_score_history 新增 (score, level_code_at_training)
    ↓
user_skill_progress 更新 (current_score, total_trainings)
    ↓
检查晋升条件：min_training_count + min_avg_score + stability_check
    ↓
条件满足 → promotion_records 新增 (type='skill_level')
    ↓
user_skill_progress 更新 current_level_code
    ↓
skill_level_history 已合并至 promotion_records
```


### 段位晋升数据写入

```
段位练习完成 → rank_works_count + 1
    ↓
用户查看进度页 或 完成训练后 → 自动触发条件检查
    ↓
查询 rank_config 中下一个未解锁段位
    ↓
校验：rank_works_count ≥ required_works
    ↓
校验：user_skill_progress 各专项等级 ≥ skill_thresholds
    ↓
全部满足 → 前端展示晋升提示弹窗
    ↓
用户确认晋升
    ↓
promotion_records 新增 (type='rank', trigger_type='manual_confirm')
    ↓
user_ranks 新增 → users.current_rank_code 更新
```


### 素材检索与保存

```
用户输入查询
    ↓
dialogue_sessions 新增/更新 + dialogue_messages 新增 (role='user')
    ↓
多路检索：material_library 向量检索 + 标签过滤 + 维度过滤
    ↓
dialogue_messages 新增 (role='assistant', content=检索结果)
    ↓
用户确认保存
    ↓
user_materials 新增 (source_type='search' 或 'generate')
    ↓
user_material_tags 批量新增（关联已有 tag_id 或创建新 tag）
    ↓
material_library.usage_count + 1（如引用系统素材）
```