# 模型生命周期字段：todo / log / history / reviewed

## 状态标记

### `metadata.todo`（ADR 0120；文件名与发布状态无关）

模型文件是否"可发布"只看 `metadata.todo` 是否存在/非空，**与文件名无关**（旧约定用 `_HOLD` 文件名后缀镶嵌这一状态，已被 ADR 0120 废除——重命名会破坏其他文件 `imports:` 路径，曾导致 `Cannot load model` 故障）：

```yaml
metadata:
  todo:
    - type: nosim | noopt | noref | quality | other
      issue: "一句话描述问题"
      evidence: "诊断依据：具体数值/现象/复现方式，使下次处理不需要重新诊断"
      next: "建议的下一步，或留给人工判断的选项；不替人工下结论"
```

- `type` 取值含义：`nosim`=sim 无法运行；`noopt`=sim 通过但 optimization 失败；`noref`=缺文献来源（`TODO:SOURCE`）；`quality`=sim/opt 均成功但结果有疑点（如 Pareto 前沿退化、可行域为空集）；`other`=其他。
- `evidence` 是核心：把诊断过程中得到的具体数值/现象写下来，避免下次处理（无论 AI 或人工）重新运行诊断。
- **无 `metadata.todo`（或为空）= 已确认通过、可发布**：`--sim` ✓、`--opt` ✓（或无 `optimization:` 块时自动跳过）、所有参数有文献来源、结果无疑点。
- 所有 `todo` 项处理完毕后删除该字段，文件回到"干净"状态——**不需要重命名文件**。

### reviewed: true

可在 `metadata` 中加可选字段 `reviewed: true`，表示建模者已人工确认机制合理、参数量级正确。这不是发布门控。

---

## 改进历史：`metadata.log`（ADR 0103）

可选字段，记录模型的改动历史，弥补 git log 在跨文件批量 commit 下追溯单个模型修改脉络的不足：

```yaml
metadata:
  log:
    "2026-06-14_10-30-00":
      change: "一句话描述本次改了什么"
      reason: "为什么这样改"
```

- key 为 `YYYY-MM-DD_HH-mm-ss` 时间戳（本地时间）；字符串字典序即时间顺序，新条目追加在末尾，不修改/删除历史条目。
- 每条只有 `change` + `reason` 两个字段：`change` 是什么靠 git diff 可查证，核心价值在 `reason`——补全 git diff 给不出的修改动机/背景。关联的 ADR、`metadata.todo` 项编号等直接写进 `reason` 文本，不单独建字段。
- 与 `metadata.todo`（前瞻：待办）互补（回顾：已完成）；处理某个 `todo` 项后，可在 `log` 追加一条说明处理结果，再从 `todo` 中删除该项。
- 仅对有语义影响的改动（参数/方程/约束/结构调整、文献依据更新）记录；格式化、拼写修正不必记录。
- 可选字段，不是发布门控；现有模型不强制回填，从下次有意义的修改开始追加即可。

---

## 调试历史版本管理：`history/` 目录（ADR 0141）

反复调试同一个模型（多次调整 `optimization` 配置重跑、多次改写 `description` 以反映诊断结论）时，**旧版本的完整内容不进入主 YAML 文件**——不要把历次 rerun 的 `optimization.results`、被推翻的旧 `description` 表述、诊断过程本身累积保留在同一个文件里，这会让文件持续膨胀、新读者分不清哪部分是当前有效结论。这条与"改进历史：`metadata.log`"是两回事：`metadata.log` 只留一行"改了什么/为什么"的索引，本身很小，留在主文件里；`history/` 存的是被取代的完整文件内容，体量可能很大，不适合留在对外发布的文件里。

**做法**：调试出新版本前，先把当前文件原样复制一份到同目录下的 `history/` 子文件夹，文件名加日期戳前缀（`history/YYYY-MM-DD_原文件名.yaml`），再回到主文件里删除已被取代的内容，只保留反映当前状态的一份干净版本。`history/` 内容仅供作者本人日后复查调试脉络，不受《面向最终读者的交付物：不留过程痕迹》规则约束，可以如实保留调试细节、失败尝试、中间数值；整个 `history/` 目录通过仓库根 `.gitignore` 的 `**/history/` 规则排除，不随代码库发布，外部读者看到的永远只是主文件的最终版本。

同样的处理方式适用于 `metadata.todo` 里已经处理完但记录了大段诊断叙述的条目：处理完毕后按现有规则从 `todo` 中删除，叙述本身如果还有查证价值，搬进 `history/` 而不是继续留在主文件的 `todo`/`results` 里"暂时保留"。

---
