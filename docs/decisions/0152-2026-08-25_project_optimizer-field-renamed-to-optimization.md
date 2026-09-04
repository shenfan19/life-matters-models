# ADR 0152 — `optimizer:` 顶层字段更名为 `optimization:`

## 状态

✅ 已实施

## 日期

2026-08-25

## 背景

LM format 的四个核心结构块合起来读作 VESO，对应 `variables`、`equations`、`simulation`、`optimizer`。ADR 0144（2026-08-08，`formulas:` 更名为 `equations:`）已经把 VESO 的助记符正式定为 Variables, Equations, Simulation, **Optimization**，但当时只改了字段名对应的助记符文字，没有同步把第四个字段本身从 `optimizer:` 改成 `optimization:`——`simulation` 是活动/过程名词，`optimizer` 却是施事者名词，VESO 四个字母里唯独 O 和其余三个的构词方式不一致。

这一不一致不是文字风格问题：`simulation:` 块装的是一次仿真实验/场景配置，不是"仿真器"这个工具本身，这正是当初命名时特意避开 `simulator` 一词的理由（`self.simulator` 作为引擎内部属性名保留至今，但对外的 YAML 字段早已是 `simulation:`）。`optimizer:` 块同样装的是一次优化实验的配置与结果（`optimizer.results`），不是"优化器"这个工具本身，理应适用同一条命名逻辑，命名成 `optimization:` 才对。

`LM_format_1.0.md` 版本历史里 v1.0 的每一条记录均标注为 Draft，规范至今没有对外冻结发布，S1 论文是这个项目第一次面向外部读者的正式发布节点。据此，本次改名不构成对已发布契约的破坏性变更，不触发 major version bump，`LM_format_1.0.md` 仍为 v1.0。

## 决策

### 1. `optimizer:` 顶层字段更名为 `optimization:`

所有 LM file（`life-matters-models`、`life-matters-game` 两仓库 `models/` 下共 175 个文件）的顶层 `optimizer:` 键改为 `optimization:`，子字段路径（`optimizer.results` → `optimization.results`、`optimizer.startpoint.regimens` → `optimization.startpoint.regimens`、`optimizer.mc` → `optimization.mc`、`optimizer.algorithm` → `optimization.algorithm` 等）随顶层键改名自动变为 `optimization.*`，无需单独处理。

### 2. 只改外部可见的字段名，engine 内部实现细节保持不变

范围边界经用户明确裁定：只改 YAML 里的字段名、规范文档、论文、GUI 展示文字、API 返回的 JSON 字段名——这些是读者/用户会直接看到的东西。engine 内部的模块文件名（`optimizer_engine.py`、`optimizer_backends.py`、`optimizer_parsing.py`、`optimizer_eval.py`、`routes/optimizer.py`）、类名、函数名、Python 属性名（`self.optimizer`）、局部变量名、API 路由路径（`/api/optimizer/*`）一律保留 `optimizer` 不变——这些是引擎自己的实现细节，没有人读 model YAML 或论文时会看到这些文件名，全部跟着改属于大幅提高改动风险和工作量、却没有外部可见收益的深度内部重构，不在本次范围内。

具体到代码层面，落地方式是：读取/写入 YAML 顶层键的那一处字符串字面量（`data.get('optimizer')` 等）改成 `data.get('optimization')`，但持有解析结果的 Python 属性名 `self.optimizer` 不改；面向用户的报错信息、CLI 输出、GUI 提示文案、i18n 字符串里但凡引用的是字段路径（如"no `optimizer:` block"“`optimizer.startpoint.regimens` 缺失”）一律改成对应的 `optimization` 路径，但描述算法/工具本身的通用英文表述（如"the optimizer searches..."“skip the optimizer”“an optimizer fitness”）保留不变，因为这类用法描述的是"优化这个动作/优化器这个工具"的概念，不是字段名本身。GUI 的 Tab/菜单标签"Optimizer"同理保留，视为工具面板名称，不是字段名引用。

### 3. 涉及的四个仓库

- **`life-matters-models`**：`LM_format_1.0.md`（正文、YAML 骨架代码块、6.2 节标题、术语表、Inclusion Test 四问、版本历史里描述历史事件的行）、`docs/authoring/` 六个文件（尤其 `regimens_and_optimizer.md` 改名为 `regimens_and_optimization.md`，含正文标题与表格）、`models/` 下全部相关模型 YAML 文件（含 `test_fixtures/`、`test_validation/`、`references/`、`papers/`、`plan/`，既包括顶层 `optimizer:` 键，也包括 `description`/`ratings` 等散文字段里提到 `optimizer` 的句子）、`test_fixtures/invalid/test_invalid_optimizer_missing_method.yaml` 改名为 `test_invalid_optimization_missing_method.yaml`、`skill_agent/` 下引用该字段的技能文档、`models/plan/lm_model_library_plan.md`、`docs/decisions/` 下描述当前状态的 ADR 正文（历史 ADR 标题/文件名本身除外，见下）、`docs/DECISIONS.md`/`docs/decisions/README.md` 索引描述文字。
- **`life-matters-reference-engine`**：`reference_engine/src/model_structure/loader.py`（YAML 顶层键读取）、`core.py`（`export_to_yaml` 序列化）、`validator.py`（三处面向用户的校验报错文案）、`optimizer_engine.py`/`validation.py`/`routes/optimizer.py`/`routes/models.py`/`routes/files.py`（docstring、报错信息、API JSON 字段名 `"optimization": model.optimizer`）、`csv_export.py`/`mc_utils.py` 注释、`cli/main.py`/`cli/batch.py`/`cli/runner.py`（CLI 输出文案与 `model_declares_step` 的实际字段判断逻辑）、`test_verification/` 下的回归测试（含一处内嵌 YAML fixture）、`docs/opt.md`/`cli.md`/`architecture.md`/`mc.md`/`design.md`/`data_flow.md`、`CHANGELOG.md`、`docs/DECISIONS.md`/`docs/decisions/README.md` 索引描述文字。
- **`life-matters-game`**：`models/scenarios/social/` 下 20 个场景 YAML、`game/src/components/StoryEditor.tsx`。
- **`life-matters-home`**：`paper/*.md` 全部论文草稿（S1-S4，含摘要、正文、贡献清单里的真实字段引用；已关闭的批注/排查记录 callout 不追溯改写）、`tasks/`（含 `archive/`）里描述真实字段路径的任务文字、`process/veso_debug_checklist.md`、`process/model_validation_workflow.md` 两份活的方法论文档；`agent_reports/`、`process/veso_case_archive.md`、`outreach/` slides、`personal/` 下的内容不做追溯改写，理由见下。

### 4. GUI 前端

`gui/src/types.ts` 的 `ModelFile.optimizer` 字段改名 `optimization`；所有读取该字段的组件/hook（`Loader.tsx`、`useBuilderState.ts`、`useFileTree.ts`、`useModelInit.ts`、`useSimulation.ts`、`usePlans.ts`、`useOptimizer.ts`、`Simulator/index.tsx`）同步改用 `.optimization`；四语言 i18n 文件（`en`/`zh-CN`/`zh-TW`/`fr`）里引用字面 YAML 语法的字符串（如"optimizer: block""optimizer.inputs"）同步改名，但"Optimizer"这个 Tab/菜单标签本身，以及描述算法行为的通用文案（如"time granularity the optimizer steps through"）保留不变，理由同上。

## 改写范围的边界

处理原则与 ADR 0144 一致：**历史记录不做追溯改写**，但用户明确要求把这条原则的适用范围收紧到"真正的历史快照"，不包括仍在被引用、会误导当前读者的活文档。

例外（保留原状，不属于遗漏）：

- **两仓库 `docs/decisions/` 下已接受的历史 ADR 的标题与文件名本身**（如 0098 `optimizer-schedule-sustained-mode.md`、0147 `optimizer-t1-value-step-grid-quantization.md`），以及本 ADR 自身描述"改名前叫什么"的历史陈述——ADR 文件名与"改名前的旧名字是什么"这两类内容改写了就会造成事实错误，保留原状；但 ADR 正文里凡是描述"当前规范/代码是什么样"的陈述，已按用户要求一并更新为 `optimization`。
- **`LM_format_1.0.md` §10.1 版本历史表**：描述历史事件本身的行，其措辞已同步更新为当前术语（如"optimizer schedule tiers"改成"optimization schedule tiers"），但 2026-08-08 那一行专门记录"`variables`/`formulas`/`simulation`/`optimizer` 这一旧简写被正式命名为 V.E.S.O."这件事本身，保留 `optimizer` 原词不变——这一行描述的是"当时那个简写长什么样"，改写会造成事实错误。
- **`life-matters-home/agent_reports/`、`process/veso_case_archive.md`、`outreach/` slides、`personal/`**——按既有惯例（`outreach/` 沿用 ADR 0144 已定的例外）不做追溯改写；`life-matters-home/tasks/`（含 `archive/`）本轮已按用户要求处理，不再属于例外。
- **`life-matters-reference-engine` 的 API 路由路径 `/api/optimizer/*`、Python 模块文件名 `optimizer_*.py`、内部属性名 `self.optimizer`**——按本次范围裁定保留不变，见"决策"第 2 条。

## 已知环境限制与未修复的既有问题（发现但不属于本次改动范围）

- `reference_engine/src/model_structure/core.py` 的 `export_to_yaml` 方法序列化时用的是 `'simulator'` 键而非当前实际生效的 `'simulation'` 键，是改名前就存在的既有不一致（与 `simulator`/`simulation` 历史遗留有关，不是本次 `optimizer`/`optimization` 改动引入的），本次只做了 `optimizer` → `optimization` 的同名改动，未修复该既有问题。
- GUI i18n 文案 `sim.setup.no_opt_vars`/`sim.opt.no_inputs` 提到的 `optimizer.inputs`/`optimization.inputs` 字段本身，早在 ADR 0088（2026-05-28）就已被废弃、统一改为 `startpoint.regimens`，这两处文案从那时起就已经是过期字段名，本次只做了 `optimizer` → `optimization` 的同名改动，未修复"inputs 这个字段名本身已经过期"这一独立、更早存在的问题。
- `life-matters-home/validation/validation_report.md` 疑似 2026-07-06 报告迁移后留下的旧副本，与 models 仓库正式公开的 `models/test_validation/validation_report.md`（已确认干净）不是同一份文件，本次未处理，是否删除留待用户判断。

## 结果

- 改名：`optimizer:` → `optimization:`（LM format 顶层字段），子路径全部随之改变；`test_invalid_optimizer_missing_method.yaml` → `test_invalid_optimization_missing_method.yaml`；`docs/authoring/regimens_and_optimizer.md` → `regimens_and_optimization.md`。
- 覆盖：`life-matters-models`、`life-matters-reference-engine`、`life-matters-game`、`life-matters-home` 四仓库的 YAML、后端、前端（含 i18n）、规范文档、建模指南、论文、ADR 正文（历史标题/文件名除外）、`home/tasks`（含 archive）；engine 内部模块文件名/类名/属性名/API 路由路径按范围裁定保留不变。
- `LM_format_1.0.md` 仍为 v1.0（Draft 状态未发布，不构成对已发布契约的破坏性变更），VESO 现在四个字母与四个顶层字段名的构词方式完全一致：Variables, Equations, Simulation, Optimization。
