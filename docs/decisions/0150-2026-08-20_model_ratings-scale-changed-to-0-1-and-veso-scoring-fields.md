# ADR 0150 — `metadata.ratings` 评分尺度改为 0-1 连续量表，新增 VESO 四问打分字段

## 状态

✅ 已实施（规范本身；`models/papers/s1/` 与 `models/plan/` 下具体模型文件的评分迁移为后续单独任务，不在本次范围内）

## 日期

2026-08-20

## 背景

本仓库此前有两套互相独立、刻度不一致的模型评分表达：

1. `models/plan/lm_model_library_plan.md` 候选总览表的 var/equ/sim/opt 四列，0-2 三档，建模前打，回答"这个方向有没有证据支撑、值不值得投入"。
2. `docs/authoring/ratings.md` 的 `metadata.ratings` 字段，1-5 五档，建模后打，回答"这个已建成模型的重要性/需求强度/证据质量/验证置信度有多高"。

2026-08-19/20 一次会话里，为 S1 论文找母婴案例（Pareto 前沿退化）的替补时，对四个新候选和一个既有案例（CKD）额外做了"外部文献核验"——检索有没有已发表的同类联合定量研究、结论是否相当——这类判断本质上就是在回答 `ratings.md` 已有的 `validation_confidence` 字段，但当时口头打成了"高/中/低"，没有对齐到量表本身。复盘时进一步发现，用户设计候选表 var/equ/sim/opt 时其实是有意把刻度压到 0-2（三档），而 `ratings.md` 是 1-5（五档），两者刻度不一致，且候选表的四个字母 V/E/S/O 与 `life-matters-home/process/veso_debug_checklist.md` 定义的 VESO debug 归因框架共享同一套四层分解（Variables/Equations/Simulation/Optimizer），是同一个核心框架的两种应用，不是巧合重名。

讨论评分尺度本身时，用户提出两点设计要求：

- **N 分之几不自解释上限**：无论是 0-2 还是 1-5，一个孤立的数字（比如"2"或"4"）不能自己说明满分是多少，读者要先确认量表定义才知道这个数字接近满还是刚及格；0-1 天然自带上限，"0.7"不需要额外上下文。
- **打分的思考负担要低，但不能牺牲精度到无法解释的地步**：最初讨论过"AI 内部用 1-5、展示给用户用 0-2"的双层方案，用户否决，理由是引入两套并行数字反而增加认知负担、且有两份记录长期跑偏的风险；最终定为单一的 0-1 连续量表，允许任意精度（不强制卡在离散档位上），但每个字段仍锚定五个参考点（0/0.25/0.5/0.75/1，与原 1-5 的五档定义一一对应），要求偏离锚点时在理由里说明相对哪个锚点、为什么偏离，防止连续精度演变成无法解释的假精度。

## 决策

### 1. `metadata.ratings` 全部字段尺度从 1-5 整数改为 0-1 连续小数

五个锚点 0/0.25/0.5/0.75/1，语义定义直接从原 1-5 的五档搬过来，只换算数字，不重新定义。允许锚点间任意精度，理由文本须说明相对最近锚点的偏离依据。面向读者的界面可选择性展示为五星（星数 = 分数 × 5，半星对应 0.1 精度），底层数值不因展示折算而改变精度。

### 2. 字段按"技术类（VESO）/非技术类（自由裁量）"两分法重组，字段数从原有的六个通用+三个类型专属精简为四个通用技术类+三个非技术类

同一天的后续讨论中，用户提出评价标准本质上分两类：VESO（模型完备性）判断有客观锚点可查，是技术类；重要性、信心、新颖度这类判断没有客观清单、本质是自由裁量，是非技术类。据此把字段重新分组、合并：

- **技术类（VESO，四个通用字段）**：`variable`（吸收原 `evidence_quality`）、`equation`、`simulation`、`optimization`（即 tradeoff 是否成立，吸收候选表 var/equ/sim/opt 四问与最初设计的 `o_tradeoff_predicted`），与 debug 用途的 VESO 框架（`veso_debug_checklist.md`）互相点对方一次，明确是同一个框架的两种应用，不重复定义标准。`optimization` 的数字可随建模进展覆写更新，但已经写进 `metadata.description.method` 的定性预判文字本身不得因为看到数值结果而悄悄改写——数字迭代和文字预判冻结是两件不同的事，最初设计的"`o_tradeoff_predicted` 打分后冻结、验证后读数另开 `validation_confidence`"方案在这轮讨论中被否决，因为把"数字不能改"和"文字预判不能悄悄改写"混为一谈，人为增加了字段数量。
- **非技术类（三个字段）**：`importance`（合并原 `topic_importance`/`framework_demand`/`paper_value`/`popularity`/`social_value` 五个"值不值得做"的判断为一个字段，一句话理由里指明具体是哪种价值驱动分数，不强制拆分子类型）、`innovation`（原有字段不变，仅尺度改为 0-1，判断新颖度而非重要性，两者是独立的轴，保留为单独字段不并入 `importance`）、`confidence`（原 `validation_confidence` 改名而来，判断依据不变，与 `optimization` 的区别是前者判断实测结果对不对得上文献、后者判断权衡结构本身成不成立；改名理由是"validation"在这套字段体系里不再是唯一语境，直接叫 `confidence` 更短、更贴合"非技术类自由裁量"的定位）。

`ratings.md` 适用范围新增 `models/plan/`（原来只写 `papers/`、`scenarios/`、`references/`），候选材料评分与建成模型评分共用同一套字段和量表。

### 3. 外部文献核验的产出正式落在 `confidence`，agent 文档不再自定义标准

`lm-modeling-design.md`"外部文献核验"一节此前用"高/中/低"口头描述可信度，改为直接按 `ratings.md` 当前定义的 `confidence` 量表打分并附理由；`lm-paper-case-writer.md` 涉及打分/引用评分的部分同步改为读取 `ratings.md`，不在各自文档内复制量表定义。以后调整评分标准只改 `ratings.md` 一处，agent 下次执行时自动读到新版本。

### 4. 本次改动范围不追溯迁移已有评分

`models/papers/s2/`、`s3/`、`s4/`、`models/scenarios/`、`models/references/` 下已有模型文件里写好的 1-5 评分本次不迁移，保留原样、按各自文件当时的定义解读，留待后续单独任务处理；`lm_model_library_plan.md` 候选总览表已写好的历史 0-2 评分同样不改写，按该表自身表头标注的旧刻度解读。`docs/authoring/ratings.md` 与 `LM_format_1.0.md` 一样处于未对外发布的 Draft 状态（参照 ADR 0144 的判断标准，无已冻结版本、无已依赖旧刻度的外部消费者），本次调整不构成对已发布契约的破坏性变更。

## 结果

- `docs/authoring/ratings.md`：评分尺度改为 0-1，五锚点；字段重组为技术类（`variable`/`equation`/`simulation`/`optimization`，四个通用）与非技术类（`importance`/`innovation`/`confidence`，`confidence` 由 `validation_confidence` 改名而来）两组，原六个通用字段+三个类型专属字段合并精简为四个通用技术类+三个非技术类共七个字段；适用范围扩展到 `models/plan/`。
- `agents/lm-modeling-design.md`："外部文献核验"改为直接引用 `ratings.md` 的 `confidence` 定义打分；"定性预判 Pareto 结构"一节新增给 `optimization` 打分的要求，数字可覆写更新，预判文字本身不得悄悄改写。
- `agents/lm-paper-case-writer.md`：`## Plan` 小节的模型评分快照、"引言"步骤均改为引用 `ratings.md` 当前定义（技术类四项+非技术类三项）。
- `life-matters-home/process/veso_debug_checklist.md`：开头新增一句话交叉引用 `ratings.md`，说明 VESO 框架的双重用途。
- 本次未改动任何具体模型文件的 `metadata.ratings` 数值，S1 论文与 `models/plan/` 下模型的实际评分迁移作为后续任务单独执行。
