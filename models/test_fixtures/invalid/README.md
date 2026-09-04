# models/test_fixtures/invalid — 错误检测测试用例

## 定位

`test_fixtures/invalid/` 下每个文件都**故意写错**，专门用来验证引擎的错误检测机制：不只是能正确加载
结构合法的模型（见 `test_fixtures/valid/`），还要能在遇到结构错误、循环 import、evidence 配置错误、
非法日期等情况时**可靠地失败**，并把具体原因通过正常调用路径（`ReferenceEngine.load_models()` /
`run_simulation()` 等，而不是只写日志）暴露给调用方。

**每个文件只故意写错一处**，其余部分保持结构合法——这样一个文件对应一个明确的校验分支，
改动校验逻辑时，只需要看这一个文件的测试是否还按预期失败（与 `test_verification/models/README.md` 的
"每个变量单独一个文件夹"原则同源）。

对应的 pytest 断言在 `test_verification/errors/`，每个文件在其 `metadata.description.result` 里注明了
预期的错误信息片段和对应的测试文件。**这些模型永远不会、也不应该被修复成能跑通**——
`--sim`/`--opt` 失败或 `cli/batch.py` 报 FAIL 是设计如此，不是回归。

## 文件清单

| 文件 | 触发的校验 | 校验位置 |
|------|-----------|---------|
| `test_invalid_step_size.yaml` | `simulation.step_size` 必须为正数 | `validator.py` `Validator.validate_model` |
| `test_invalid_optimization_missing_method.yaml` | `optimization.method` 必填 | `validator.py` `Validator.validate_model` |
| `test_invalid_equation_undefined_var.yaml` | `dynamics` 引用未声明变量 | `validator.py` `validate_equations`（AST 提取变量） |
| `test_invalid_equation_deprecated_dt.yaml` | `dynamics` 使用废弃符号 `dt`，应改用 `step` | `validator.py` `validate_equations` |
| `test_invalid_import_circular_a.yaml` + `_b.yaml` | 循环 import 检测（a↔b 互相导入） | `loader.py` `Loader._load_model_data` |
| `test_invalid_import_escapes_root.yaml` | 相对 import 越出 `models/` 根目录 | `loader.py` `Loader._load_model_data` |
| `test_invalid_evidence_name_collision.yaml` | `evidence` 名称与 `variables` 重名 | `loader.py` `Loader._apply_model_data` |
| `test_invalid_evidence_missing_baseline_ref.yaml` | `rr`/`or` evidence 用 `applies_to` 时缺 `baseline_ref` | `loader.py` `Loader._apply_model_data` |
| `test_invalid_yaml_not_dict.yaml` | 顶层 YAML 必须是 mapping，不能是 list/标量 | `loader.py` `Loader._load_model_data` |
| `test_invalid_date_range.yaml` | `end_date` 不能早于 `start_date` | `validation.py` `validate_simulator_dates`（仅在 run 时校验，加载/`validate_model()` 不检查，见文件内 description） |

## 新增一个错误检测用例

1. 先确认目标校验分支在引擎代码里存在（`validator.py` / `loader.py` / `validation.py`），
   不要为不存在的校验写 fixture。
2. 复制 `test_invalid_*.yaml` 里结构最接近的一个作为模板，只改动触发目标校验所需的最小字段。
3. 在 `metadata.description` 里写清楚：故意写错的是什么、预期报错信息包含什么子串、对应哪个
   `test_verification/errors/*.py` 文件。
4. 在 `test_verification/errors/` 对应文件里加断言，并在上表补一行。
