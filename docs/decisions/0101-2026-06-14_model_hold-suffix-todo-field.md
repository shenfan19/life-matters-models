# 0101 — 文件名质量标记统一为 `_HOLD` + `metadata.todo` 任务列表

**日期**：2026-06-14
**状态**：⚪ 文件名部分已被 [0120](0120-2026-06-23_model_drop-hold-filename-suffix.md) 取代（`_HOLD` 后缀废除；`metadata.todo` 字段定义保留有效，仍是当前约定）
**类别**：模型库管理 / 工程约定

---

## 背景

ADR 0096 用 `_nosim` / `_noopt` / `_noref` 三种文件名后缀分别标记 sim、optimization、文献来源三个维度的"未完成"状态，组合写法（如 `_nosim_noopt_noref`）在实践中暴露两个问题：

1. **后缀不携带诊断信息**：一个 `_noopt` 文件只说明"optimization 失败"，不说明*为什么*。每次复查（无论 AI 还是人工）都要重新跑一遍 `--opt`、重新分析日志和 Pareto 前沿，诊断过程不可复用。
2. **二元状态表达不了"能跑但有疑点"**：用新版 CLI 跑 `models/papers/` 的修复队列时发现，部分模型 `--sim`/`--opt` 均返回成功（技术意义上的 PASS），但结果本身有问题——例如 Pareto 前沿坍缩为单点、或硬约束在模型自带的"最优方案"示例下都无法满足（联合可行域为空集）。这类"运行成功但结果不可用"的情况，在三后缀体系里无法标记，只能靠人工记忆或额外文档追踪。

---

## 决策

### 1. 三种状态后缀统一为单一后缀 `_HOLD`

- `_HOLD` = 该文件存在一项或多项待处理事项，详情记录在 `metadata.todo`（见下）。
- gitignore 复用既有规则 `**/*_HOLD.yaml`（该规则此前已存在于 `.gitignore`，但未在 `docs/model.md` / ADR 中文档化）。
- **无 `_HOLD` 后缀 且 无 `metadata.todo` = 已确认通过、可发布**（与 ADR 0096 的"无后缀=通过"语义一致，只是判定信号从"三个后缀都不在"变为"`_HOLD` 不在 + todo 为空"）。

### 2. 新增 `metadata.todo`：结构化任务列表

```yaml
metadata:
  todo:
    - type: nosim | noopt | noref | quality | other
      issue: "一句话描述问题"
      evidence: "诊断依据：具体数值/现象/复现方式，使下次处理不需要重新诊断"
      next: "建议的下一步，或留给人工判断的选项；不替人工下结论"
```

- `type` 延续 ADR 0096 三种分类的语义（`nosim`/`noopt`/`noref`），新增：
  - `quality`：sim/opt 均成功，但结果存在疑点（前沿退化、可行域为空集等）
  - `other`：不属于以上分类的待办
- `evidence` 是本次改动的核心价值：把诊断过程的结论（具体数值、运行条件、复现路径）写下来，AI 或人工下次直接读取即可继续，不必重新运行/重新推导。
- `next` 给出选项而非直接执行的修复方案——涉及参数量级、约束阈值等建模判断的内容，留给人工确认。

### 3. 状态转换

- 所有 `metadata.todo` 项处理完毕并清空（删除该字段）后，去掉 `_HOLD` 后缀，文件回到"干净"状态。
- 处理过程中可以保留部分 `todo` 项、去掉已解决的项；只要 `todo` 非空，文件名保留 `_HOLD`。

### 4. 与旧约定（ADR 0096）的关系

- `_nosim` / `_noopt` / `_noref` 三后缀及其 gitignore 规则已废弃，全部迁移为 `_HOLD` + `metadata.todo`（2026-06-14 完成全量迁移）。
- 新发现的问题统一用 `_HOLD` + `metadata.todo` 记录。

### 5. 与目录级 `_HOLD` 的关系

`models/papers/` 下已存在 `s3_HOLD/`、`s4_HOLD/` 等目录级 `_HOLD` 标记（论文结构"暂缓"信号，`.gitignore` 中 `**/*_HOLD/` 使整个目录不发布）。这是**论文结构维度**的标记，与本 ADR 的**技术/质量维度**标记（文件名 `_HOLD` + `metadata.todo`）是两个独立维度，可以共存（例如 `s3_HOLD/foo_HOLD.yaml`），互不影响。

---

## 试点（2026-06-14）

对一个 `models/papers/` 下的 `_noopt` 文件应用新约定：重命名为 `_HOLD.yaml`，添加 `metadata.todo`，记录本次用新版 `sim_cli` 跑 `--opt` 时发现的"联合可行域为空集"诊断结论（一处硬约束与当前动力学不匹配，模型自带的示例方案在仿真窗口内使受约束状态量偏离约束范围）。

## 全量迁移（2026-06-14）

`models/` 下其余约 100 个 `_nosim`/`_noopt`/`_noref`（含组合）文件已批量迁移为 `_HOLD` + `metadata.todo`。迁移仅做后缀→`todo` 的结构转换，`evidence` 字段标注为"迁移自旧后缀标记，尚未重新运行 --sim/--opt 诊断"；后续通过 `sim_cli/batch.py` 修复队列逐个跑 `--sim`/`--opt` 补充具体诊断证据并清空 `todo`。

---

## 关联

- `docs/model.md` — 文件名质量标记节
- ADR 0096 — 三后缀约定（已被本 ADR 取代）
- 内部任务记录 `2026-06-14_task_hold-todo-migration.md` — 全量迁移记录
