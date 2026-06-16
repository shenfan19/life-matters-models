# ADR 0105 — step_unit 条件必填 + 废弃 dt/step_size 动力学符号

**日期**：2026-06-16  
**状态**：已采纳  
**修订**：ADR 0104（`formulas.step_unit` 必填规则过于严格）

---

## 背景

ADR 0104 规定每条公式必须声明 `step_unit`。但执行后发现该规则过于严格：

- **静态代数公式**（`formula:` 字典形式，如 `performance: p0 + fitness - fatigue`）根本不使用 `step`，强制声明 `step_unit` 没有意义。
- 已有模型（banister、bergman 等）大量静态公式因此触发 validator 报错，阻碍正常加载。

此外，原始公式符号表（ADR 0046）曾记录 `dt` 和 `step_size` 作为 `step` 的别名，部分旧模型（如 `digestive_system`）在动力学表达式中使用 `dt`。随着 `step` 成为唯一规范符号，这些别名应当废弃。

`formula:` 字典形式（即将 key-value 映射直接写在 `formula:` 块下，用于静态代数赋值）曾被允许，与 `dynamics:` 语义重叠，已由 ADR 0106 统一移除：所有变量更新一律使用 `dynamics:`，`formula:` 仅保留字符串形式（存入 formula_results）。

---

## 决策

### 1. `step_unit` 改为条件必填

**规则**：仅当 `dynamics` 表达式中出现 `step`（或废弃符号 `dt`/`step_size`）时，`step_unit` 才是必填字段。

| 公式类型 | `step_unit` | 说明 |
|---------|-------------|------|
| `dynamics:` 含 `step` | **必填** | validator 强制检查 |
| `dynamics:` 不含 `step`（纯代数赋值） | **不需要** | 无时间步进，无需声明 |
| `formula:` 字符串（存入 formula_results） | **不需要** | 不更新模型变量，无步进语义 |

### 2. 废弃 `dt` 和 `step_size` 作为动力学符号

- `dt` 和 `step_size` 禁止在 `dynamics` 表达式中使用。
- Validator 检测到即报错（`is_valid = False`），与缺少 `step_unit` 同级别。
- 所有存量模型中的 `dt` 均替换为 `step`。
- `model.md` 更新：将二者标记为废弃，`step` 是唯一规范符号。

> **引擎兼容性**：`simulation.py` 仍向符号表注入 `dt` 和 `step_size`（`_STEP_SYMS`），以支持极少数未迁移模型的过渡期运行。后续版本可逐步移除注入，但需先确认所有模型已迁移。

---

## 引擎适配

| 模块 | 变更 |
|------|------|
| `validator.py` | `validate_formulas()` 中：只在 dynamics 含 `step`/`dt`/`step_size` 时检查 `step_unit`；检测到 `dt`/`step_size` 时额外报错 |
| `model.md` | 更新"公式内符号"表；更新 `step_unit` 必填说明为条件必填 |
| 存量 YAML 模型 | 全量迁移：`dt` → `step`；静态公式移除多余 `step_unit`（可选，不强制） |

---

## 取舍

**放弃**：全局强制所有公式声明 `step_unit`（语义完整但过于严格）。

**获得**：
- 静态公式不再误报 validator 错误
- `step` 成为唯一步长符号，消除 dt/step_size 带来的混淆
- 动力学公式仍强制声明 `step_unit`，跨模块 import 语义不变
