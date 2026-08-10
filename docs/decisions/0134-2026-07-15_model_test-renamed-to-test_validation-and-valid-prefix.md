# 0134 — `models/test/` 改名为 `models/test_validation/`，`valid/` 下文件加 `test_valid_` 前缀

**日期**：2026-07-15
**状态**：✅ 已接受

---

## 背景

仓库里有两套名字很像但职责不同的测试目录：`models/test/`（YAML fixture + `cli/batch.py`
批量跑法，验证"模型/仿真结果对不对"）和 life-matters-reference-engine 仓库里刚从 `tests/` 改名为
`test_verify/` 的 pytest 套件（验证"引擎代码写得对不对"，见 †life-matters-reference-engine 仓库同日
提交）。两者都叫"test"，用户在实际使用中（`python cli/batch.py --input-dir test` 跑全
模型 vs 阅读 pytest 代码）分不清哪个是哪个，要求把二者的名字都改得能自解释：pytest 套件
按"verify"改名，本目录按"validate"改名——即本次决策。

## 决策

### 1. 顶层目录 `models/test/` → `models/test_validation/`

与 life-matters-reference-engine 的 `tests/` → `test_verify/` 对称：`test_validation` 对应"验证模型/
仿真结果的可信度"，`test_verify` 对应"验证引擎代码的正确性"。见新版
`models/test_validation/README.md` 开篇专门解释这组术语区别（用户要求"首先说明
validation 和 verify 的区别"）。

### 2. `valid/` 下 35 个文件加 `test_valid_` 前缀，`invalid/` 不变

`valid/test_X.yaml` → `valid/test_valid_X.yaml`（如 `test_banister_v1_analytical.yaml` →
`test_valid_banister_v1_analytical.yaml`）。`invalid/` 下 11 个文件本来就是
`test_invalid_*.yaml` 格式（ADR 0125 时就已如此命名），不需要改。子目录本身仍然叫
`valid/`、`invalid/`，不受本次改名影响——只有目录**里面的文件名**统一加前缀，配合
`metadata.name` 同步更新为新文件名（沿用改名前的约定：文件名与 `metadata.name` 一致）。

**为什么两边都要改，不是只改目录名**：`invalid/` 下文件名早已经是
`test_invalid_*.yaml`（ADR 0125 的产物），如果 `valid/` 不跟进改成 `test_valid_*.yaml`，
两个子目录里的文件命名规则会不一致，走读代码时看到裸 `test_opt_t2.yaml` 分不清它属于
哪个子目录、是否合法。

### 3. 影响范围：目录改名 + 文件改名叠加，需要在同一批替换里处理路径 token

`imports:`、`metadata.name`、`description` 里互相提及其他 fixture 名字的地方（如
`test_import_top.yaml` 的 problem 描述里写"imports test_import_base AND
test_import_layer"）都需要同步替换成新文件名，而不能只改目录前缀——否则
`imports: [test/valid/test_import_base]` 这类三层耦合（目录名+子目录+文件名）会因为
目录改名和文件改名分两步做而在中间状态产生死链接。本次用脚本一次性替换"路径形式"
（`test/valid/<旧名>` → `test_validation/valid/<新名>`）和"裸词形式"（`<旧名>` →
`<新名>`，用于 `metadata.name` 和 prose 提及），按长度降序应用避免前缀冲突
（如 `test_opt_t1_pareto` 和 `test_opt_t1_single` 共享前缀 `test_opt_t1`）。

life-matters-reference-engine 仓库同步更新的文件：`cli/batch.py`（docstring + argparse help）、
`docs/reference_engine/cli.md`、`docs/reference_engine/DECISIONS.md`、`README.md`（文档
索引表）、`gui/src/components/sim_tab/optUtils.test.ts` 的 `FIXTURES_DIR` 和 4 个 T1-T4
fixture 文件名、`gui/e2e/specs/run-simulation.spec.ts` 的 3 个 GUI 文件树 testid 字符串、
`reference_engine/scripts/validate_banister.py` 的 `MODEL_PATH`，以及 `test_verify/`
下全部引用具体 model_key 的 pytest 用例（`test_capacity_limits.py`、
`test_same_day_duration.py`、`test_schedule_runner.py`、`test_session_cleanup.py`、
`test_sim_cli_consistency.py`、`errors/*.py`）。`test_verify/models/test_mc_distributions/`、
`test_verify/models/test_plans/` 两个子目录本身也重命名为
`test_verify/models/test_valid_mc_distributions/`、`test_verify/models/test_valid_plans/`，
与 `tests/models/README.md`（现 `test_verify/models/README.md`）"目录名=模型
`metadata.name`"的既有约定保持一致。

life-matters-home 仓库的 `paper/c_paper_s1_cn.md` 里 3 处引用具体路径/文件名的审阅批注同步更新
（`models/test/test_report.md`、`test_banister_v1_analytical.yaml` 相关提及），批注内容
本身（数值结论）不受影响，只是路径更新。

### 4. 历史 ADR 不做追溯性改写

`docs/decisions/`（本目录）和 life-matters-reference-engine `docs/reference_engine/decisions/` 下已有的
带日期 ADR 文件、以及两边 `decisions/README.md` 索引表里描述"某次改动做了什么"的历史行，
一律不因为本次改名回去改写旧路径——它们记录的是当时为真的状态，事后编辑等于篡改历史
记录。只有明确代表"当前状态"的活文档（如 `DECISIONS.md` 顶层导览、`README.md` 文档索引）
才同步更新为新路径。

## 结果

- 改名：`models/test/` → `models/test_validation/`
- 改名：`models/test_validation/valid/` 下 35 个 `.yaml` 文件加 `test_valid_` 前缀，
  `metadata.name` 同步更新，`imports:`/描述文字里的互相提及同步更新
- 新增/改写：`models/test_validation/README.md`（顶层导览，开篇解释 validation vs verify）
- 同步：life-matters-reference-engine 仓库约 15 处文件的路径引用（见上）、`test_verify/models/` 下 2 个
  子目录改名
- 同步：life-matters-home `paper/c_paper_s1_cn.md` 3 处审阅批注里的路径引用
- 验证：改名后 `pytest`（`test_verify/`，38 个用例）全部通过

## 未决

- `models/test_validation/test_plan.md`、`test_report.md` 内部除路径 token 外的实质内容
  本次未重新审阅，仅做路径替换。
