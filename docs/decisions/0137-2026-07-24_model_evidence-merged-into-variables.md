# 0137 — `evidence:` 顶层节并入 `variables:`，改用 `evidence_type` 字段

**日期**：2026-07-24
**状态**：✅ 已接受

---

## 背景

ADR 0040 把文献效应量（OR/HR/RR/Cohen's d 等）设计成独立的顶层 YAML 节 `evidence:`，与
`variables:` 平级：`variables:` 下的条目声明 `type: state|input|parameter`，`evidence:` 下的
条目声明 `type: rr|or|hr|ard|cohens_d|ir|beta|pk`，Loader 加载阶段把 `evidence:` 条目换算后
写进 `self.variables`（`parameter` 类型），两个 YAML 节最终汇合进同一个运行时命名空间——
`evidence:` 条目本质上就是变量，只是多了一步换算。

用户在复核 LM format 顶层结构时提出：项目希望 YAML 顶层类目保持"Unix 风格"的最少化、同级别
清晰（`metadata`/`variables`/`formulas`/`sim`/`opt` 这条线简洁，每类只做一件事，留给未来扩展
的空间也要求"强行归类"到已有的同级结构里，而不是像 Windows 那样不断在顶层堆砌平行概念）。
`evidence:` 独立成第 6 个顶层节，正是这种堆砌的开端：它和 `variables:` 描述的是同一种东西
（模型里的命名量），只是多了"数据来源需要换算"这一个属性，却被提升成了一个完全独立的顶层
命名空间，需要额外一条"不能与 `variables:` 重名"的校验规则来维持两个命名空间不冲突——这条
校验规则本身就是"不该有两个命名空间"的证据。

讨论中比较了两种合并方案：

1. 把 8 个 evidence 子类型编码进 `variables.type` 本身（如 `type: evidence-ard`），`type`
   枚举从 3 值扩展到 11 值。
2. `variables.type` 保持 3 值不变（`state`/`input`/`parameter`），新增一个正交字段
   `evidence_type` 表达文献效应量子类型，声明了 `evidence_type` 的条目 `type` 必须是
   `parameter`。

方案 1 会把"变量在动力学里的角色"（角色维度：state/input/parameter）和"数据来源的统计效应量
类型"（来源维度：rr/or/hr/...）两个正交维度绑进同一个字符串字段，且直接推翻了 ADR 0040「实施
记录」里已经论证过的"不为 evidence 建第 4 种类型，避免为尚不存在的 Modeller 优化器预留类型
膨胀"的决定——方案 1 不是不建第 4 种类型，而是建了 8 种，膨胀方向正好相反。方案 2 保持
`type` 三值不变，`evidence_type` 只是把 Loader 换算后本来就会自动挂上的两个溯源字段
（`evidence_type`/`evidence_raw_value`，ADR 0040 已定义）中的第一个，从"仅供查询的运行时
产物"提前变成"建模者声明时就填的输入字段"，本身不是新概念。

## 决策

**采纳方案 2**：删除顶层 `evidence:` 节，evidence 声明整体挪进 `variables:`，用
`type: parameter` + `evidence_type: <8 种子类型之一>` 表达。

- `variables:` 条目新增可选字段 `evidence_type`（`rr`/`or`/`hr`/`ard`/`cohens_d`/`ir`/`beta`/`pk`）。
  声明了该字段的条目，`value` 填**原始文献值**，`type` 必须是 `parameter`（否则 Loader 报错
  拒绝），Loader 在加载阶段把 `value` 原地换算为可进公式的系数（同名覆盖，不加后缀），换算前
  的原始值保留在运行时的 `evidence_raw_value` 字段。换算公式、8 种子类型详解、`applies_to`
  自动接入 dynamics 的机制完全不变，只是数据源从 `data['evidence']` 换成
  `data['variables']`（详见 `life-matters-reference-engine` 仓库 `docs/reference_engine/evidence/{conversion,applies_to}.md`）。
- `evidence:` 与 `variables:` 两个命名空间的重名校验（ADR 0040 引入）随之删除——两者合一后，
  这类冲突已经是"同一个 YAML 映射内重复键"，属于 YAML 语法层面的问题，不再是本层需要单独
  校验的语义冲突。
- 新增一条此前不存在、由本次合并催生的校验：`applies_to` 只在声明了 `evidence_type` 的条目上
  有意义（合并前，`applies_to` 只可能出现在 `evidence:` 节里，不存在"写在普通 parameter 上"
  这种可能性；合并后这种误用变得可能，Loader 显式拒绝）。
- `VariableType` 仍然只有 3 个值，不新增、不扩展——这正是 ADR 0040「实施记录」里"不为 evidence
  建第 4 种类型"这条决定的延续，不是推翻。

## 影响范围

**`life-matters-reference-engine` 仓库**：
- `reference_engine/src/model_structure/loader.py` `_apply_model_data`：`variables:` 与
  `evidence:` 两个处理循环合并为一个（新增 `evidence_type`/角色校验分支），`applies_to`
  自动接入循环的数据源从 `data['evidence']` 改为 `data['variables']`，新增
  "`applies_to` 需要 `evidence_type`" 校验。`base.py`（`Variable.evidence_type`/
  `evidence_raw_value` 字段定义）、`routes/models.py`（API 响应）不变——这两处此前就已经
  按"换算结果并入 `variables`"的方式工作，不受此次改动影响。
- `docs/model.md`「变量类型（3 种）+ evidence 顶层换算」章节整体重写为「... + evidence_type
  原地换算」，含 YAML 示例、`baseline_ref`/`applies_to` 字段表、完整 Schema 示例。
- `docs/evidence/conversion.md`、`docs/evidence/applies_to.md`：换算/接入逻辑本身（8 种
  子类型公式、校验顺序、生成表达式模板）不变，仅更新"YAML 里怎么声明"相关表述和代码引用
  （`data['evidence']` → `variables_data`），新增 `applies_to` 校验流程图中的
  "是否声明 `evidence_type`"前置分支。
- `docs/design.md`、`docs/impl.md`：evidence 相关小节同步措辞。
- `test_verification/errors/test_evidence_errors.py`：原 `test_evidence_name_colliding_with_variable_is_rejected`
  测试的场景（两个命名空间重名）合并后结构性消失，改为
  `test_evidence_type_on_non_parameter_role_is_rejected`，验证新增的"`evidence_type` 要求
  `type: parameter`"校验。

**`life-matters-models` 仓库**：
- `models/test_fixtures/valid/test_valid_evidence_{rr,or,hr,ard,cohens_d,ir,beta,pk,types}.yaml`
  （9 个）、`invalid/test_invalid_evidence_missing_baseline_ref.yaml`：`evidence:` 节整体
  挪进 `variables:`。
- `invalid/test_invalid_evidence_name_collision.yaml`：原测试场景结构性消失，改为测试
  "`evidence_type` 声明在非 `parameter` 角色变量上会被拒绝"。
- `models/test_fixtures/fixture_catalog.md`：§1 说明文字与该 fixture 描述同步更新。
- `models/references/**` 下 13 个真实模型 YAML（`class_size_2026`、`tobacco_elasticity_2026`、
  `mindfulness_mood_2026`、`mindfulness_anxiety_2026`、`inactivity_shortsleep_mortality_2026`、
  `cbt_depression_2026`、`processed_meat_mortality_2026`、`statin_ldl_doseresponse_2026`、
  `caffeine_pk_2026`、`waist_circumference_lungcancer_2026`、`smoking_lungcancer_2026`、
  `processed_meat_crc_2026`、`hepb_hcc_2026`）：`evidence:` 节挪进 `variables:`，均为单条或
  两条 evidence，均不涉及 `applies_to`/`baseline_ref`，迁移后逐一重新加载验证换算值与迁移前
  完全一致。

**历史 ADR 正文不动**：ADR 0040 描述的是它被接受时的设计（顶层节）及两次「实施记录」，按
"历史记录不做追溯性改写"惯例保留原文，不回填本次改动；本 ADR 是它的后续演进记录。

## 结果

- 顶层 YAML 节从 `metadata`/`imports`/`variables`/`formulas`/`simulation`/`optimization` +
  `evidence` 收敛回不含 `evidence` 的这条主线，evidence 声明是 `variables:` 条目的一个可选
  维度，不再是独立命名空间。
- `VariableType` 保持 3 值，未引入方案 1 那种 11 值的角色×来源复合枚举。
- 9 个 test_fixtures + 13 个真实模型 YAML 迁移后重新加载，换算值（含 `applies_to` 自动生成的
  5 条 dynamics 表达式）与迁移前逐一核对完全一致。

## 未决

- GUI（sim_gui）暂无 `evidence_type` 字段的专属可视化标注（Overview 表格里 evidence 换算出的
  parameter 目前和普通 parameter 显示方式相同，无来源徽章），留待 variables/formulas 通用
  编辑器实现后一并补齐。
- paper（S1–S4）章节及 `LM_FORMAT_1.0.md` 若引用了旧版 `evidence:` 顶层节语法，需要单独排查
  同步（不属于本 ADR 的引擎/模型改动范围，跟踪见 `paper/c_paper_plan.md` 待更新备忘）。
