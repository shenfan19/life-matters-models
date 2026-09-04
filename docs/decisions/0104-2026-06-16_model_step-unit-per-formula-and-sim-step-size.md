# ADR 0104 — 步长设计重构：per-formula step_unit + simulation.step_size

**日期**：2026-06-16  
**状态**：已采纳  
**替代**：ADR 0046（metadata.step_size 设计）

---

## 背景

ADR 0046 将 `step_size` 放在 `metadata` 下，作为整个模型的全局步长，同时赋予它两个角色：

1. **公式语义单位**：公式中 `step` 符号所代表的时间长度
2. **仿真执行步长**：引擎每步实际推进的时间

此外，`optimization.step_size` 可选地覆盖仿真执行步长用于优化。

**问题**：

- `metadata` 的定位是描述性字段（name、tags、description），包含计算语义的步长在语义上不属于此处。
- `optimization.step_size` 的存在使得这两个角色已经分离，但仿真侧没有对称字段，导致不对称。
- 公式的步长单位是*公式本身的属性*（系数按什么时间尺度标定），而不是模型级别的全局属性——跨模块 import 时，同一运行中的不同公式可能来自不同步长的源模型，全局 metadata 字段无法准确表达这一差异。

---

## 决策

将两个角色彻底分离，使用两个独立字段：

### 1. `formulas.<name>.step_unit`（必填，字符串）

```yaml
formulas:
  bp_dynamics:
    description: "..."
    step_unit: day          # minute | hour | day
    dynamics:
      systolic_bp: "systolic_bp + (...) * step"
    priority: 7
```

- 声明该条公式中 `step` 符号所代表的时间单位。
- **必填**：validator 强制检查，缺失即报错。
- 无 `value` 字段——公式校准单位的 value 永远是 1，不需要声明。
- 跨模块 import 时，每条公式携带自己的 `step_unit`，loader 直接读取，无需查询来源模块的 metadata。

### 2. `simulation.step_size`（必填，`{value, unit}`）

```yaml
simulation:
  step_size:
    value: 1
    unit: day               # minute | hour | day
  start_date: "YYYY-MM-DD"
  end_date:   "YYYY-MM-DD"
```

- 声明仿真执行的步长，与 `optimization.step_size` 完全对称。
- **必填**：validator 强制检查。
- `optimization.step_size` 保持可选（缺省沿用 API 传入的覆盖值）。

### 3. `step` 数值的计算方式

引擎每步将 `simulation.step_size` 换算为秒（`step_size_sec`），将 `formula.step_unit` 也换算为秒（`step_unit_sec`），两者相除得到注入公式的 `step` 数值：

```
step = step_size_sec / step_unit_sec
```

示例：`simulation.step_size = 1 day`，`formula.step_unit = hour` → `step = 86400 / 3600 = 24`。

### 4. `metadata.step_size` 移除

- 从 metadata 中删除 `step_size` 字段。
- 现有 YAML 文件全部迁移：`metadata.step_size` 拆分为 `simulation.step_size` + 每条 `formula.step_unit`。

---

## 引擎适配

| 模块 | 变更 |
|------|------|
| `loader.py` | 读取 `simulation.step_size` 而非 `metadata.step_size`；为每条公式读取 `form_data['step_unit']` 并设 `formula.step_unit` 和 `formula.step_size_sec` |
| `validator.py` | 新增：`simulation.step_size` 必填检查；每条 formula 的 `step_unit` 必填且值域检查 |
| `base.py` | `Formula` dataclass 新增 `step_unit: Optional[str]` 字段 |
| `optimizer_engine.py` | 不变（已独立读取 `optimization.step_size`） |
| `simulator_engine.py` | 不变（读取 loader 注入的 `simulator['step_size']`） |

---

## 跨模块 import 行为

跨步长 import 之前依赖 `merged_sources['step_sizes']` 字典（记录每个来源模块的 metadata.step_size）。

新方案：每条公式的 `step_unit` 字段在 import 合并后仍保留在 formula 数据中，loader 直接读取——不再需要来源模块的 step_sizes 索引。

示例：top 模型（step=1 hour）import base 模型（step=1 day）：
- `mass_decay`（来自 base）：`step_unit: day` → `step_size_sec = 86400`
- `drug_decay`（top 本身）：`step_unit: hour` → `step_size_sec = 3600`

引擎在每步对每条公式分别注入正确的 `step` 值。

---

## 取舍

**放弃**：`metadata.step_size` 作为全局默认，可以让用户在单模块模型中少写一个字段。

**获得**：
- 语义显式：公式步长是公式的属性，不是模型的属性
- 对称性：sim 和 opt 各自独立声明步长，地位平等
- 无隐式继承：validator 强制全显式，符合 LM format 的平直表达原则
- 跨模块 import 自包含：每条公式携带自己的单位，无需全局索引
