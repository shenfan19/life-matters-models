# 贡献指南

欢迎向 Life Matters 模型库贡献 YAML 模型！

## 贡献方式

- **新增模型**：基于公开文献建立新的生理、疾病或社会动力学模型
- **改进现有模型**：补充文献来源、修正参数、添加优化器配置
- **修复问题**：修复带 `metadata.todo` 待办标记的问题模型
- **反馈问题**：在 [Issue Tracker](https://github.com/shenfan19/life-matters/issues) 提交问题或建议

## 模型格式

所有模型须符合 LM format 格式规范：

- 格式入门：[docs/quickstart.md](docs/quickstart.md)
- 格式规范全文：[docs/LM_format_1.0.md](docs/LM_format_1.0.md)
- 建模实践指南：[docs/authoring/README.md](docs/authoring/README.md)

## 质量要求

提交前须满足：

1. **`description` 完整**：每个变量和方程都有 `description` 字段
2. **`reference` 完整**：所有数值、范围和方程标注文献来源；暂无来源填 `TODO:SOURCE`
3. **sim 可运行**：无 `metadata.todo` 中 `type: nosim` 的待办（本地用 `python cli/batch.py --input-dir <目录>` 验证，需配套仿真引擎）
4. **文件名规范**：`{topic}_{year}_{author}.yaml`，使用 snake_case
5. **原创转写**：`description` 等文字字段须用贡献者自己的话转写文献中的机制与结论，不逐句翻译/抄录原文，不粘贴论文图表截图或完整表格——只提取建模所需的数值，并通过 `reference`/`locator` 标注出处

## 贡献者协议（CLA）

提交 Pull Request 即表示你同意 [CLA.md](CLA.md) 中的条款，主要内容：你有权提交该内容、同意以 CC BY 4.0 发布、并对参数准确性和文献引用负责。无需签名。

## 提交流程

1. Fork 本仓库
2. 将模型放入对应子目录（`models/references/medical/` 或 `models/references/social/` 等）
3. 本地运行仿真引擎验证可通过（无 `metadata.todo` 待办）
4. 提交 Pull Request，描述模型的文献来源和建模场景

## 状态标记

模型是否"可发布"只看 `metadata.todo` 字段，与文件名无关（ADR 0120）：

```yaml
metadata:
  todo:
    - type: nosim | noopt | noref | quality | other
      issue: "一句话描述问题"
      evidence: "诊断依据，使下次处理不需要重新诊断"
      next: "建议的下一步"
```

新建或未验证的模型先加上对应的 `todo` 项；问题处理完毕后删除该项，`todo` 清空即视为通过，**不需要改文件名**。完整字段说明见 [docs/authoring/bookkeeping.md](docs/authoring/bookkeeping.md)「状态标记」一节。
