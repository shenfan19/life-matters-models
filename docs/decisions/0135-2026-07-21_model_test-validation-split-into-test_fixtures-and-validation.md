# 0135 — `models/test_validation/` 拆分为 `models/test_fixtures/` + `models/validation/`

**日期**：2026-07-21
**状态**：✅ 已接受

---

## 背景

ADR 0134（2026-07-15）把 `models/test/` 改名为 `models/test_validation/`，动机是与 life-matters-reference-engine
的 `tests/`→`test_verify/` 对称命名（"validate 对应模型可信度，verify 对应引擎代码正确性"）。
但该目录的**实际内容**——`valid/`（结构合法的引擎功能示例）+ `invalid/`（故意写错的错误检测
fixture）——从 ADR 0125 起就是"引擎能不能正确加载/跑通/拒绝一个 YAML"，属于 V&V 框架里的
**verification**（代码写得对不对），不是 **validation**（模型代表不代表真实世界）。`valid/README.md`
自己也早已写明"两者都不代表真实临床或社会场景，不需要文献参数校准"。

真正的 validation 内容——`models/test_validation/validation_report.md` 第1节"文献对标"——
从来不依赖 `valid/`/`invalid/` 的任何 fixture，而是直接指向 `models/papers/s1/banister/`、
`models/papers/s1/ckd_protein/`、`models/papers/s4/hypertension_gout/` 这些真实论文模型。
两者只是历史上恰好放在同一个目录下，内容上无关。这个名实不符在 2026-07-21 一次围绕
`verification_report.md`/`validation_report.md`/`test_verify/` 三者关系的讨论中被发现。

## 决策

### 1. 顶层目录一分为二

- `models/test_validation/` → `models/test_fixtures/`：只装引擎 verification 用的 YAML
  fixture（`valid/`、`invalid/` 子目录不变，`test_valid_`/`test_invalid_` 文件前缀不变——
  这两个"valid/invalid"是通用工程语汇（YAML 结构合法与否），跟 V&V 的"validation"专有
  术语不是一回事，本身没有歧义，不需要跟着改）。
- 新建 `models/validation/`：只装 `validation_report.md`（文献对标/优化合理性/API-IO边界/
  逐模型科学内容核对结果），以及后续的 `reports/` CSV 归档目录。与 `papers/`、`references/`、
  `scenarios/`、`test_fixtures/` 同级，不再从属于 fixture 目录。

**为什么不直接叫 `test_verification`**：考虑过让 `test_fixtures` 改叫 `test_verification`
以呼应 verify 语汇，但这会跟 life-matters-reference-engine 的 `test_verify/` 撞得更严重——两个目录名几乎
一样，反而比现在的 `test_validation` vs `test_verify` 更难分清"哪个放数据、哪个放断言"。
`test_fixtures` 准确描述内容（可复用的测试用 YAML 数据），且与 `test_verify` 无字面重叠。

### 2. `fixture_catalog.md` 改名（原 `validation_catalog.md`）+ 不新建报告文件

`validation_catalog.md`（`valid`/`invalid` 逐 fixture 导览）随目录迁移并改名为
`models/test_fixtures/fixture_catalog.md`，标题/自引用同步更新。**不给 `test_fixtures/`
新建"verification_report.md"**——`test_verify/verification_report.md`（life-matters-reference-engine
仓库）§1.1 已经把消费这批 fixture 的 pytest（`test_verify/errors/`）纳入自己的报告范围，
全项目只应该有这一份 verification 结果报告；`test_fixtures/` 只需要 README + catalog 做
导览，不需要单独的结果报告。

### 3. 影响范围：两仓库 + home 共约 30 处路径引用同步更新

**life-matters-models 仓库**：`models/test_fixtures/{README.md,fixture_catalog.md,valid/README.md,
invalid/README.md}` 自引用；3 个 import 路径（`test_valid_import_layer/top.yaml`、
`test_valid_step_cross_import_top.yaml`）；3 个 invalid import fixture 的 import 路径
（`test_invalid_import_circular_a/b.yaml`、`test_invalid_import_escapes_root.yaml`）；
`models/validation/validation_report.md` 自身的交叉引用；
`models/papers/s1/banister/banister_step_convergence_grid_POINTER.yaml`（顺带修正了一处
更早就存在的错误引用——该文件把步长收敛协议标成"V4"，但 `verification_report.md` 里这个
协议编号是 V2，V4 是另一件事，见下条）。

**life-matters-reference-engine 仓库**：`test_verify/{README.md,verification_report.md,models/README.md,
errors/README.md}`；`test_verify/` 下 9 个 pytest 文件里加载 fixture 的路径字符串；
`reference_engine/scripts/validate_banister{,_step_grid}.py` 的 `MODEL_PATH`/`MODEL_DIR`；
`cli/batch.py`、`docs/reference_engine/{cli.md,DECISIONS.md,evidence/conversion.md}`、
根 `README.md`；`gui/src/components/sim_tab/optUtils.test.ts` 的 `FIXTURES_DIR`；
`gui/e2e/specs/run-simulation.spec.ts` 的 3 处 GUI 文件树 testid 字符串（含一处此前遗漏、
本次一并发现的裸目录名 testid）。`test_verify/README.md`"和 `models/test_validation/` 的
关系"一节按新划分重写——原文把 `test_fixtures`（改名前）描述成"validate 侧"，现已改为
准确描述其 verify 属性，并新增对 `models/validation/` 的独立说明。

**life-matters-home 仓库**：`process/model_validation_workflow.md`（validation_report.md 路径、
CSV 归档路径）；`tasks/task_index.md`、`tasks/2026-07-21_task_gui-invalid-model-error-display.md`
里当天新写的活跃引用。历史 ADR 正文（0134 及其在 `decisions/README.md` 的索引行）、
`tasks/archive/` 下已归档文件、`paper/c_paper_s1_cn.md` 里独立于本次改动、更早就已过期的
`test_plan.md`/`test_report.md` 审阅批注引用，均按"历史记录不做追溯性改写"惯例不动。

### 4. 顺带修正：banister 步长收敛协议编号 V4 → V2

`verification_report.md` 的协议编号是 V1（解析解逐日对比）/V2（步长收敛性检验），V3/V4 是
后来分配给姊妹文件 `validation_report.md`"训练适应-疲劳"小节的文献场景复现检验点
（减量后表现峰值/超量恢复出现时间），跟步长收敛完全是两回事。6 个
`test_valid_banister_v1_step_*.yaml` fixture 和 `banister_step_convergence_grid_POINTER.yaml`
之前引用的是更早、现已不存在的 `test_plan.md` 里的"协议V4"编号，本次一并同步为当前
`verification_report.md` 的 V2。

## 结果

- 改名：`models/test_validation/` → `models/test_fixtures/`（`valid/`、`invalid/` 子目录
  与文件名不变）
- 改名：`models/test_validation/validation_catalog.md` → `models/test_fixtures/fixture_catalog.md`
- 新增：`models/validation/`，迁入 `validation_report.md`
- 同步：两仓库 + home 约 30 处路径引用（见上）
- 验证：`pytest test_verify/errors/ test_verify/models/ test_verify/test_sim_cli_consistency.py
  test_verify/test_same_day_duration.py test_verify/test_schedule_runner.py` 全部通过；
  `reference_engine/scripts/validate_banister.py`（新路径）实跑确认能正确找到并加载模型

## 未决

- `cli/batch.py --input-dir test_fixtures`（不带 `/valid`）扫描整个 `test_fixtures/` 的
  batch 报告，本次改名后未重新实跑核对格式；预期行为与改名前一致（`invalid/` 下 FAIL 是
  设计如此），未发现需要单独处理的理由，留作后续常规巡检覆盖，不阻塞本次改名。
