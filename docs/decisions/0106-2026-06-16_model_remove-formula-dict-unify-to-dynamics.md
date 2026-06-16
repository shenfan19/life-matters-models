# ADR 0106 — 移除 formula: 字典形式，统一变量更新为 dynamics:

**日期**：2026-06-16  
**状态**：已采纳  
**补充**：ADR 0105（formula: 字典条件必填的前提已消除）

---

## 背景

LM format 历史上允许 `formulas:` 条目使用两种变量更新写法：

```yaml
# 写法 A：dynamics: 字典（含或不含 step）
dynamics:
  fitness: fitness + (g * training_load - k1 * fitness) * step

# 写法 B：formula: 字典（静态代数赋值，与 dynamics 语义重叠）
formula:
  performance: p0 + fitness - fatigue
```

两者均直接更新模型状态变量，语义等价，区别仅在于是否使用 `step`。这造成：

- 建模者需要在两种写法之间做选择，但选择标准（"是否含 step"）是隐式的
- `formula:` 作为关键字承载了两种完全不同的含义：字典形式（更新变量）和字符串形式（存入 formula_results）
- ADR 0105 需要专门为 `formula: dict` 豁免 `step_unit` 必填规则，增加规则复杂度

---

## 决策

**移除 `formula:` 字典形式**。所有直接更新状态变量的操作统一使用 `dynamics:`，无论是否使用 `step`：

```yaml
# ✅ 统一写法：dynamics: 用于所有变量更新
dynamics:
  fitness:     "fitness + (g * training_load - k1 * fitness) * step"   # 含 step
  performance: "p0 + fitness - fatigue"                                  # 不含 step（静态代数）
```

`formula:` 保留字符串形式（语义：计算中间值并存入 `formula_results`，不直接写回模型变量）：

```yaml
formula: "p0 + fitness - fatigue"   # 存入 formula_results['formula_name']，不更新任何变量
```

---

## 消歧义后的字段语义

| 字段 | 类型 | 语义 |
|------|------|------|
| `dynamics: {var: expr}` | 字典 | 直接更新模型状态变量，支持含 step 和不含 step 的表达式 |
| `formula: "expression"` | 字符串 | 计算中间值，存入 `formula_results`（不更新变量） |

两者在同一条公式内**互斥**（只能选一）。

---

## 迁移

所有存量模型中的 `formula: {dict}` 条目已统一替换为 `dynamics:`。替换是纯机械的：字段名 `formula:` → `dynamics:`，子内容不变。被替换的条目均不含 `step`，所以 `step_unit` 不需要（ADR 0105 规则不触发）。

---

## 影响

| 文件 | 变更 |
|------|------|
| 存量 YAML 模型 | `formula: {dict}` → `dynamics:`（已批量完成） |
| `model.md` | Schema 示例更新；`step_unit` 说明去除对 `formula: dict` 的豁免引用；`lm_score` 不可逆模式示例更新 |
| ADR 0105 | 表格中 `formula: 字典` 行改为 `dynamics: 不含 step` 行；背景节补充说明 |
| `validator.py` | 可选：移除 `formula:` 值类型的 dict 分支检查（向后兼容期可保留但不推荐新用） |
