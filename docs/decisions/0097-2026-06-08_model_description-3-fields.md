# ADR 0097 — papers/ 模型 description 简化为三字段

## 状态

✅ 已实施

## 日期

2026-06-08

## 背景

ADR 0065 定义了结构化 description 规范，推荐字段为 `brief`、`need`、`problem`、`method`、`simulation`、`optimization`、`result`、`conclusion`、`limitations`（共 9 个）。

随着论文模型数量增长，实践中暴露了两个问题：

1. **字段过多**：9 个字段即使每个只写 2–3 句，累积仍有 500–800 字，GUI 中难以快速浏览。
2. **语气过于学术**：`method` 和 `optimization` 大量使用框架内部符号（T1/T2/T3/T4、K×4、NSGA-II、Pareto），外行读者（临床医生、感兴趣的非专业人士）难以理解，而研究者可以直接读 YAML 变量和公式。

`description` 的主要受众是 GUI 的非专业读者，不是替代论文正文。

## 决策

`papers/` 目录下的模型 `description` 统一采用三字段结构：

```yaml
description:
  problem: >
    是什么 + 为什么存在 + 核心科学张力，3–5 句，非专业读者可理解。
    去掉框架内部符号（T1/T2/T3/T4、K×4、NSGA-II 等）。
  result: >
    仿真结论或 Pareto 前沿摘要，2–4 句；
    未运行时写理论预期并注明"注：仿真尚未运行"。
  limitations: >
    已知建模边界与待精化参数，1–3 句。
```

**字段合并规则**：
- `problem` = 原 `brief` + `need`（背景动机融入问题陈述）
- `result` = 原 `result` + `conclusion`（结论作为结果的解读，不独立成字段）
- `limitations`：保留不变

**删除字段**：`brief`、`need`、`method`、`simulation`、`optimization`、`conclusion`。

**适用范围**：`papers/`（s1–s5 及顶层）。`references/` 和其他模型不受约束，可继续使用任意字段。

**顶层 `clinical_brief`**：已并入 `description.problem`，不再作为 metadata 顶层字段使用。

## 语气要求

- 去掉框架内部符号，用普通医学/科学语言描述
- 不用"T1 强度 × T3 频率 × T4 时机"，改用"每阶段负荷强度、每周训练次数、升级时机"
- 保留领域术语（ALT、GFR、皮质醇、Pareto 前沿等），这些对目标读者已足够直观
- `result` 允许随 Pareto 复杂度自然增长（2 句到 5 句均可）

## 影响

- ADR 0065 仍有效；本 ADR 在 `papers/` 范围内收窄了推荐字段集。
- 已于 2026-06-08 对 papers/ 下全部 15 个 YAML 完成迁移。
- `model.md` 已同步更新 `metadata.description` 规范段落。
- GUI 无需改动：字段顺序展示逻辑不变，字段数减少会自动变得更紧凑。

## 非目标

- 不强制 `references/` 模型采用三字段。
- 不把三字段变成 schema 硬约束（validator 保持宽松）。
- 不在 description 中恢复 method/optimization 的技术细节；这些内容属于变量/公式的 `description` 和 `reference` 字段。
