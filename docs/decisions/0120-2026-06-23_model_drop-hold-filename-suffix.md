# 0120 — 废除 `_HOLD` 文件名后缀，状态判定仅看 `metadata.todo`

**日期**：2026-06-23
**状态**：🟢 已实施（部分取代 [0101](0101-2026-06-14_model_hold-suffix-todo-field.md)：文件名后缀部分废除，`metadata.todo` 字段及其语义不变）
**类别**：模型库管理 / 工程约定

---

## 背景

ADR 0101 用文件名 `_HOLD` 后缀 + `metadata.todo` 双重信号表示"该文件存在待处理事项"：`_HOLD` 后缀本身不携带信息，只是 `metadata.todo` 非空这一事实在文件名上的镶嵌。

实践中暴露的问题：todo 项处理完毕、去掉 `_HOLD` 后缀时，文件路径发生变化，**所有引用该文件的 `imports:` 路径需要同步更新**，否则 `--sim`/`--opt` 报 `Cannot load model`。这不是假设风险——`papers/s4/ckd_protein_pareto.yaml` 和 `papers/s4/hypertension_gout_3obj.yaml` 均发生过这类故障（imports 路径未跟着上游文件改名同步更新）。

## 决策

### 1. 状态判定唯一信号：`metadata.todo`

- `metadata.todo` 存在且非空 = 该文件有待处理事项（草稿/未确认状态）。
- `metadata.todo` 不存在或为空 = 已确认通过、可发布。
- 文件名与发布状态**完全脱钩**：草稿阶段不再要求任何文件名后缀，todo 清空后**不需要重命名文件**。

### 2. 废除内容

- `_HOLD` 文件名后缀约定（ADR 0101 的文件名部分）废除。
- `.gitignore` 中 `**/*_HOLD.yaml`、`**/*_HOLD/` 规则移除——发布门控改为外部脚本读取 `metadata.todo` 字段判定，不再依赖文件名 pattern。
- `metadata.todo` 字段结构、`type`/`issue`/`evidence`/`next` 子字段语义不变，沿用 ADR 0101 原定义。

### 3. 与目录级 `_HOLD`（论文结构维度）的关系

ADR 0101 第 5 节提到的目录级 `_HOLD`（如 `papers/s3_HOLD/`，表示论文结构"暂缓"）当前仓库中没有实例，本次一并移除该约定的文档描述；如未来需要"暂缓发布某个目录"的信号，应另起一个不依赖文件名/目录名的机制（如目录下放置 `.draft` 标记文件，或在该目录的索引文件中显式声明），避免重复 ADR 0101 暴露的同一类问题。

## 迁移（2026-06-23）

内部开发工作副本下 104 个 `*_HOLD.yaml` 文件去除后缀，改名后确认零处 `imports:` 引用、零处跨文件 prose 提及指向这些文件的旧名——纯文件名变更，无需同步修改其他文件。

## 关联

- ADR 0101 — 原约定（文件名部分被本 ADR 取代，`metadata.todo` 字段定义保留有效）
- `docs/model.md` — 文件名质量标记节
