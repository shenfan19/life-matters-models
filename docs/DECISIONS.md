# Model 决议汇总

> 本文件是 `docs/decisions/` 中 YAML 模型格式和模型库结构相关 ADR 的**主题分类摘要**。  
> 仿真引擎、优化器、UI 相关 ADR 见 `life-matters-reference-engine/docs/DECISIONS.md`。  
> 完整时序索引见 [decisions/README.md](decisions/README.md)。

**重要程度**：⭐⭐ = 核心约束，影响格式规范或架构，不可随意更改；⭐ = 重要实现决策；无标注 = 已实施，历史记录

---

## 一、YAML 模型格式（→ model.md）

| ADR | 标题 | 重要程度 | 状态 |
|-----|------|---------|------|
| [0040](decisions/0040-2026-04-22_sim_医学证据类型与变量映射.md) | **医学证据 8 子类型（evidence vs parameter 区分）** | ⭐⭐ | ✅ |
| [0044](decisions/0044-2026-04-30_sim_schedule作为simulation-input子类型.md) | **schedule 归属 simulation 块；pulse 模式；离散 input 不写零值点** | ⭐⭐ | ✅ |
| [0046](decisions/0046-2026-04-30_sim_步长设计-step_size元数据与step公式符号.md) | ~~step_size 元数据~~（已由 ADR 0104 替代） | | 🔴 已废弃 |
| [0104](decisions/0104-2026-06-16_model_step-unit-per-formula-and-sim-step-size.md) | **per-equation `step_unit`（必填）+ `simulation.step_size`；移除 `metadata.step_size`** | ⭐⭐ | ✅ |
| [0053](decisions/0053-2026-05-03_sim_date_range调度字段与YAML-schedule优先级修复.md) | date_range 字段；YAML Schedule 优先于 GUI Regimen | ⭐ | ✅ |
| [0063](decisions/0063-2026-05-07_sim_resolved-imports-and-output-selection.md) | **Resolved imports 与输出变量选择规则（已由 0107 修订）** | ⭐⭐ | ✅ |
| [0107](decisions/0107-2026-06-16_model_output-variables-import-overwrite.md) | **`output_variables` / `output_types` import 行为统一为覆盖（取代并集）** | ⭐⭐ | ✅ |
| [0065](decisions/0065-2026-05-08_sim_structured-description.md) | metadata.description 支持结构化写法（brief/need/method 等字段） | ⭐ | ✅ |
| [0075](decisions/0075-2026-05-17_model_remove-type-standalone-fields.md) | **删除 YAML 顶层 `type` 和 `standalone` 字段** | ⭐⭐ | ✅ |
| [0092](decisions/0092-2026-06-05_model_input-variable-bare-unit-rule.md) | **`type: input` 变量裸单位规范（事件量，禁止速率单位）** | ⭐⭐ | ✅ |
| [0096](decisions/0096-2026-06-06_model_filename-quality-markers.md) | **文件名质量标记：`_nosim` / `_noopt` / `_noref` 后缀约定** | ⭐⭐ | ✅ |
| [0098](decisions/0098-2026-06-11_sim_optimizer-schedule-sustained-mode.md) | optimization.schedules 新增 `mode: sustained`（子日步长持续输入） | ⭐ | ✅（旧格式，由 0100 取代但仍受支持） |
| [0099](decisions/0099-2026-06-11_sim_sustained-value-step-invariance.md) | sustained 模式 `value` 语义修正：窗口总量 / N_steps（step-size 不变性） | ⭐⭐ | ✅ |
| [0100](decisions/0100-2026-06-11_sim_unify-pulse-sustained-time-interval.md) | **统一 pulse/sustained 为时间区间 `time_start`/`time_end`；GUI 取消 full day/time/sustained 三态** | ⭐⭐ | 🟡 部分实施（papers/s5 术语未实施） |

---

## 二、模型库结构与命名

| ADR | 标题 | 重要程度 | 状态 |
|-----|------|---------|------|
| [0022](decisions/0022-models-three-level-taxonomy.md) | **Models 三层分类体系（medical/social → 学科 → 细分）** | ⭐⭐ | ✅ |
| [0041](decisions/0041-2026-04-22_project_命名规范下划线优先.md) | **命名规范：snake_case 下划线优先** | ⭐⭐ | ✅ |
| [0042](decisions/0042-2026-04-23_project_mod-to-model-rename.md) | mod → model 全面重命名 | | ✅ |
| [0057](decisions/0057-2026-05-04_project_models-paper-directory.md) | models/papers/ 论文专用目录约定 | ⭐ | ✅ |
| [0062](decisions/0062-2026-05-06_project_models-directory-rename.md) | models 顶层目录与文件命名规范更新 | | ✅ |
| [0086](decisions/0086-2026-05-26_project_lmml-rename-from-lmf.md) | **格式命名：LMF → LMML（Life Matters Model Language）** | ⭐ | ✅ |

---

## 维护规则

- 新 ADR：写入 `decisions/` 并在 `decisions/README.md` 添加行，同时在本文件对应分类中新增一行
- ⭐⭐ 决议有改动时：同步更新 `model.md` 对应章节
- YAML 格式类 ADR 归入第一节；模型库组织类归入第二节
