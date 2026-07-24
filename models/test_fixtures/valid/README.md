# models/test_fixtures/valid — 仿真器功能测试用例（正确性）

## 定位

`test_fixtures/valid/` 存放**结构合法、用于测试和解释仿真器各项功能的最小化示例**，每个文件聚焦 LM format 的一个特性（如 imports 组合、MC 分布、K×4 优化的各 Tier、公式条件、阶段性 schedule 等）。与故意写错、用于验证错误检测机制的 `test_fixtures/invalid/` 相对（见其 README）。

这些文件不代表真实临床或社会场景，主要用途是：
- 验证仿真引擎/优化器对应功能正确工作
- 作为该功能的最小可读示例，供开发者和模型作者参考

与 `papers/`（论文场景）、`scenarios/`（客观可用仿真）、`references/`（基础子模型构件）不同，`test_fixtures/valid/` 不需要文献参数校准。

每个模型文件都欢迎任何用户参与编辑、补充测试用例或修复问题。
