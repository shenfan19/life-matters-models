# ADR 0141 — description 恢复 method 字段 + 调试历史移入 history/ 目录

## 状态

✅ 已实施

## 日期

2026-08-05

## 背景

ADR 0097 把 `papers/` 模型的 `description` 收窄为 `problem/result/limitations` 三字段，`method` 被列入删除字段，理由是当时的 `method` 内容大量使用框架内部符号（T1/T2/T3/T4、NSGA-II 等），外行读者难以理解。

随着模型数量增长，暴露出两个新问题：

1. **`method` 被删除后，"这个模型融合了哪些机制"这一信息在 `description` 里没有位置**：LM 的核心方法论是耦合多个独立机制而非堆叠，这恰恰是每个模型最值得在 description 里讲清楚的内容，但当前三字段里没有专门位置承载它，导致这条信息要么散落在 `problem` 里挤占篇幅，要么完全缺失。
2. **反复调试同一个模型时，`problem`/`result` 和 `optimization.results` 容易积累过程痕迹**：实际案例见 `s1/ckd_protein/ckd_protein_opt_joint_largepop.yaml`——`problem` 字段写的是"当前搜索规模可能遗漏前沿尾端，需要扩大规模验证"，这是调试过程的自我说明，不是模型要回答的科学问题；`optimization.results` 下堆积了 `rerun_2026-07-10`、`rerun_2026-07-11_mc_fixed`、`rerun_2026-07-14`、`historical_stale_from_2026-06-15` 四个历史版本的完整运行记录，只有最后一次是当前有效结果，其余三个除了追溯调试脉络外没有对外价值，却让文件持续膨胀。这类内容违反全局 `~/.claude/CLAUDE.md`《面向最终读者的交付物：不留过程痕迹》一节，但此前该规则未明确覆盖 model YAML 的 description 和 optimization 结果块。

## 决策

### 1. `papers/` 模型 description 恢复为四字段：`problem / method / result / limitations`

- `problem`：不变，合并原 `brief` + `need`，但额外明确排除调试/优化过程本身（搜索规模是否足够、此前表述经核查是巧合之类）——这类内容属于 `metadata.todo`/`metadata.log`/`history/`，不进 `description`。
- **`method`（恢复）**：说明该模型融合了哪些机制/动力学模型，以及它们之间共享哪个决策变量或资源竞争通路，体现耦合而非堆叠。字段不定长，机制数量多时用列表逐条列举。
- `result`：不变，合并原 `result` + `conclusion`，同样排除调试过程叙述，只写最终仿真/优化结果。
- `limitations`：不变。

`method` 同时恢复为通用 Schema（ADR 0065 九字段集合）里的正式字段定义，措辞从"模型结构、时间步长、核心状态和输入"改为"该模型融合了哪些机制/动力学模型，以及它们之间共享哪个决策变量或资源竞争通路"，与 `papers/` 的用法保持一致——`references/` 等非 `papers/` 模型不强制要求这个字段，但新建模型时鼓励一并写上。

### 2. 调试历史版本移入同目录 `history/` 子文件夹

反复调试模型时，被取代的完整版本（旧 `optimization.results`、被推翻的旧 `description` 表述）不再累积保留在主 YAML 文件内。做法：改动前把当前文件原样复制到 `history/YYYY-MM-DD_原文件名.yaml`，再在主文件中只保留反映当前状态的干净版本。`history/` 内容不受"不留过程痕迹"规则约束，可以如实保留调试细节，仅供作者本人日后复查；整个目录通过 `.gitignore` 的 `**/history/` 规则排除，不随仓库发布。

`metadata.log` 不受影响，继续保留在主文件——它只是一行"改了什么/为什么"的索引，体量小、对外部读者有用（追溯设计动机），不属于需要搬去 `history/` 的过程痕迹。

## 影响

- `model.md` 已同步更新：`metadata.description` 章节恢复 `method` 定义、补充 `problem` 的过程语言排除说明；新增「调试历史版本管理：`history/` 目录」一节。
- `models/.gitignore` 已新增 `**/history/`。
- 全局 `~/.claude/CLAUDE.md`《面向最终读者的交付物：不留过程痕迹》一节已补充说明，明确该规则覆盖 model YAML 的 description 字段。
- `models/papers/` 下全部模型文件按新规则批量更新：清理 `problem`/`result` 中的过程语言，新增 `method` 字段，历史调试记录迁移至各自目录的 `history/`。

## 非目标

- 不强制 `references/` 模型采用四字段或立即回填 `method`。
- 不把四字段变成 schema 硬约束（validator 保持宽松）。
- `metadata.todo`/`metadata.log` 的既有规则不变，本 ADR 只新增 `history/` 这一个存放位置，不改变 `todo`/`log` 本身的用途和格式。
