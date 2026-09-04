# 0125 — `models/test/` 拆分为 `valid/` + `invalid/`：新增错误检测 fixture

**日期**：2026-07-05
**状态**：✅ 已接受

---

## 背景

`models/test/` 原本只放结构合法的引擎功能示例（imports 组合、MC 分布、K×4 优化各 Tier 等
31 个文件），验证的是"引擎能正确加载/跑通合法模型"。但错误检测机制审查发现，引擎对
**结构错误的模型**——循环 import、evidence 名称冲突、公式引用未声明变量等——完全没有
测试覆盖：既没有故意写错的 fixture，也没有断言"加载这类模型必须失败、且报出具体原因"
的测试（见 †0124，life-matters-reference-engine 仓库，那次审查同时发现 `LoaderEngine.fetch()` 会把具体
错误信息吞掉，只记日志）。

## 决策

### 1. 按"结构合法 / 故意写错"拆成两个子目录，而不是混放或建新顶层目录

- `models/test/valid/` — 原有 31 个文件原样迁移，用途不变（功能示例，非真实场景）。
- `models/test/invalid/` — 新增 11 个 fixture，每个只故意写错一处（simulation.step_size、
  optimization.method、公式引用未声明变量、废弃符号 `dt`、循环 import、import 越出 models
  根目录、evidence 名称冲突、evidence 缺 baseline_ref、顶层 YAML 非 mapping、
  `end_date` 早于 `start_date`），覆盖 `validator.py`/`loader.py`/`validation.py` 四类
  校验分支。

**放在 `test/` 下而不是新建顶层目录**：两者都不是真实场景，都是"引擎测试基础设施"，与
既有 `test/` 的定位一致，只是补上"负面"一半；新建顶层目录会制造第二套顶层分类
（`papers/`/`scenarios`/`references/`/`test/`），增加认知负担。

**每个 invalid fixture 只错一处**：与 `tests/models/README.md`"每个变量单独一个文件夹"
的隔离原则同源——改动某个校验分支的逻辑时，只需要看对应的一个文件是否还按预期失败，
不用担心一个 fixture 里多个错误互相干扰、掩盖回归。

### 2. 影响范围：3 处内部 `imports:` 路径 + 5 处外部引用需要同步改

`test_import_layer.yaml`/`test_import_top.yaml`/`test_step_cross_import_top.yaml` 内部的
`imports: [test/test_import_base]` 改为 `test/valid/test_import_base`（沿用 ADR 0120 的
教训：改名会破坏其他文件的 `imports:` 路径，必须逐一核对，不能只做文件系统移动）。

life-matters-reference-engine 仓库 5 处引用旧路径的代码同步更新：`cli/batch.py` 文档字符串、
`docs/reference_engine/cli.md`、`gui/src/components/sim_tab/optUtils.test.ts` 的
`FIXTURES_DIR`、`gui/e2e/specs/run-simulation.spec.ts`、`tests/test_sim_cli_consistency.py`
里 `test/test_opt_t1_single` → `test/valid/test_opt_t1_single`。

### 3. 已知副作用：整库/`--input-dir test` 批量测试会把 invalid fixture 报成 FAIL

`cli/batch.py` 目前没有排除目录的机制（`rglob('*.yaml')` 无过滤），扫描整个模型库或
`--input-dir test`（不指定到 `valid` 子目录）时，`invalid/` 下的模型会被跑一遍
`--sim`/`--opt` 并且预期 FAIL——这是设计如此（fixture 本来就该失败），已在
`cli/batch.py`、`docs/reference_engine/cli.md` 里加注释说明，但**没有**给 `batch.py`
加排除机制，也没有实跑一次整库 batch 确认这些 FAIL 行渲染正常、不会因为
`test_invalid_yaml_not_dict.yaml`（顶层是 list）之类的极端结构导致 batch.py 自身崩溃
而不是走正常的 FAIL 记录路径——这条留作未决事项，不阻塞本次决策。

## 结果

- 新增：`models/test/invalid/`（11 个 yaml + README.md）
- 迁移：`models/test/*.yaml` → `models/test/valid/`（31 个文件 + README.md，3 个内部
  import 路径同步更新）
- 新增：`models/test/README.md`（顶层导览，说明两个子目录的定位）
- 同步：life-matters-reference-engine 仓库 5 处外部路径引用（见上）
- 回归锁定：`tests/errors/`（life-matters-reference-engine 仓库，11 个 pytest，见 †0124）

## 未决

- `cli/batch.py` 是否需要排除目录机制，避免整库/宽泛 `--input-dir` 批量测试的报告里
  混入预期之内的 FAIL：未决定，需要单独评估。
- `gui/e2e/specs/run-simulation.spec.ts` 里"点击 `test` → `test/valid` 两级文件夹"的改动
  只依据 `SimModelTree.tsx` 代码走读推断，未启动实际 GUI 验证。
