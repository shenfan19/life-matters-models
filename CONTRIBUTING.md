# 贡献指南

欢迎向 Life Matters 模型库贡献 YAML 模型！

## 贡献方式

- **新增模型**：基于公开文献建立新的生理、疾病或社会动力学模型
- **改进现有模型**：补充文献来源、修正参数、添加优化器配置
- **修复问题**：修复文件名带 `_nosim` / `_noopt` / `_noref` 的问题模型
- **反馈问题**：在 [Issue Tracker](https://github.com/shenfan19/life-matters/issues) 提交问题或建议

## 模型格式

所有模型须符合 LM format 格式规范：

- 格式入门：[docs/quickstart.md](docs/quickstart.md)
- 完整 Schema：[docs/model.md](docs/model.md)
- 格式规范全文：[docs/LM_format_1.0.md](docs/LM_format_1.0.md)

## 质量要求

提交前须满足：

1. **`description` 完整**：每个变量和公式都有 `description` 字段
2. **`reference` 完整**：所有数值、范围和公式标注文献来源；暂无来源填 `TODO:SOURCE`
3. **sim 可运行**：文件名无 `_nosim` 后缀（本地用 `bash script/test_batch.sh` 验证，需配套仿真引擎）
4. **文件名规范**：`{topic}_{year}_{author}.yaml`，使用 snake_case

## 贡献者协议（CLA）

提交 Pull Request 即表示你同意 [CLA.md](CLA.md) 中的条款，主要内容：你有权提交该内容、同意以 CC BY 4.0 发布、并对参数准确性和文献引用负责。无需签名。

## 提交流程

1. Fork 本仓库
2. 将模型放入对应子目录（`models/references/medical/` 或 `models/references/social/` 等）
3. 本地运行仿真引擎验证可通过（文件名无 `_nosim`）
4. 提交 Pull Request，描述模型的文献来源和建模场景

## 文件名质量标记

新建或未验证的模型，文件名加 `_nosim_noopt` 后缀作为初始状态：

```
glucose_regulation_2026_bergman_nosim_noopt.yaml   ← 新建草稿
glucose_regulation_2026_bergman_noopt.yaml         ← sim 通过，opt 待修
glucose_regulation_2026_bergman.yaml               ← 完整通过
```

通过测试后删除对应后缀。
