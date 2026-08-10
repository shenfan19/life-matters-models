# models/test_fixtures — 引擎 verification 用的 YAML fixture 库

## 本目录测的是 verify，不是 validate

这个仓库里有几个名字容易混淆的目录，先说清楚各自测什么：

- **`models/test_fixtures/`（本目录）**：一批 YAML fixture（`valid/` 结构合法、`invalid/` 故意写错），本身不是测试代码，是给别处的测试提供输入数据——引擎能否正确加载/跑通合法模型、能否在写错的模型上可靠报错。这测的是**引擎代码写得对不对**（verify），不涉及任何模型的科学可信度，`valid/README.md`、`invalid/README.md` 已说明"两者都不代表真实临床或社会场景，不需要文献参数校准"。
- **`test_verification/`（life-matters-reference-engine 仓库根目录）**：真正执行断言的 pytest 代码，读取本目录的 fixture 作为输入，断言加载/运行结果符合预期。方法论和结果见 `test_verification/verification_report.md`——**全项目只有这一份 verification 结果报告**，本目录不重复维护。
- **`models/test_validation/`**：完全不同的另一件事——文献对标（这个模型的仿真结果是否落在已发表文献报告的范围内）、优化合理性等，测的是**模型代表不代表真实世界**（validate），跟本目录的 fixture 内容无关。注意本目录曾经也叫这个名字：2026-07-21 上午 ADR 0135 先把当时名实不符的 `models/test_validation/`（装的是本目录这批 fixture）拆分为本目录 `models/test_fixtures/` + 新建的纯报告目录 `models/validation/`；当天下午 ADR 0136 又把 `models/validation/` 改回 `models/test_validation/`（此时内容已经是纯 validation_report.md，不再有名实不符的问题），与本目录不是同一个目录、只是先后用过同一个名字。见 `models/test_validation/validation_report.md`。

一句话区分：**本目录（+ `test_verification/`）问"这行引擎代码写对了吗"，`models/test_validation/` 问"这个模型/这次仿真结果可信吗"。**

## 目录内容

[`fixture_catalog.md`](fixture_catalog.md)：`valid`、`invalid` 每个 YAML fixture 按功能领域分组的逐项说明——测什么、为什么单独测。

- **`valid/`** — 结构合法、能正常加载和跑通的最小化功能示例，每个文件聚焦 LM format 的一个特性（imports 组合、MC 分布、K×4 优化各 Tier、方程条件、evidence 子类型等），文件名统一以 `test_valid_` 开头。用途见 `valid/README.md`。
- **`invalid/`** — 故意写错的 fixture，用于验证引擎的**错误检测机制**：加载/校验时必须失败，且失败原因必须通过 `ReferenceEngine.load_models()` 之类的正常调用路径可见（而不是只进日志），文件名统一以 `test_invalid_` 开头。用途见 `invalid/README.md`；对应的 pytest 断言在 `test_verification/errors/`。
