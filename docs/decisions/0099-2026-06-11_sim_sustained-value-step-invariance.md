# 0099 — sustained 模式 value 语义修正：窗口总量 / N_steps（step-size 不变性）

**日期**：2026-06-11
**状态**：🟡 提议（修订 0098，尚未实施）
**类别**：仿真引擎 / 优化器 schema

---

## 背景

[0098](0098-2026-06-11_sim_optimizer-schedule-sustained-mode.md) 实现的 `mode: sustained`，
当前在每个命中 step 都原样写入 `value`（`current + ev['value']`，未做任何缩放）。

这带来一个与框架核心原则冲突的问题：**`step_size` 本应只是精度参数，不应改变模型的物理结果**
（Euler 离散积分章节、ADR 0092 的"裸单位/事件量"规则均隐含此原则）。但当前实现下：

```
总贡献量 = value × N_steps，其中 N_steps = 生效窗口时长 / step_size
```

即同一个 YAML，把 `step_size` 从 `hour` 改成 `30min`，sustained 变量对 `state` 的总贡献会翻倍——
这违反了"step 只管精度"的设计目标，也和 pulse 的语义不一致：pulse 的 `value` 是"一次性总量"，
与 `step_size` 无关（只要事件仍落在某一个 step 内）。

## 决策

**`mode: sustained` 的 `value` 字段语义改为"整个生效窗口内的总量"**（与 pulse 的"一次性总量"同一量纲），
引擎按 `value / N_steps` 写入每个命中 step：

```
N_steps = 生效窗口总时长 / step_size
per_step_value = value / N_steps
```

`state` 公式侧**不需要任何改动**——继续遵循"`type: input` 变量贡献给 `state` 公式时不乘 `step`"
的既有规则（model.md 388-405 行）。验证：

```
总贡献 = per_step_value × N_steps = (value / N_steps) × N_steps = value
```

与 `step_size` 无关，和 pulse（`N_steps=1` 时 `value/1=value`）在同一套公式下自洽——
**pulse 是 sustained 在 `N_steps=1` 时的特例**，不是另一套逻辑。

### N_steps 的计算规则

`window_duration` 由 `date_range` × `time_range` 决定：

| `date_range` | `time_range` | `window_duration` |
|---|---|---|
| 给定 `[d0,d1]` | 给定 `[t0,t1]` | `(d1-d0+1天) × (t1-t0)` |
| 给定 `[d0,d1]` | 缺省 | `(d1-d0+1天) × 24h` |
| 缺省（全程） | 给定 `[t0,t1]` | `仿真总天数 × (t1-t0)` |
| 缺省（全程） | 缺省 | `仿真总时长` |

`date_range` 缺省时需要仿真总时长——这在 `apply_regimens` 单次调用里拿不到，因此
**`N_steps` 必须在循环开始前预计算一次**（每个 sustained 事件算一次，缓存为 `ev['_n_steps']`，
不修改原始 YAML dict），而不是每个 step 都重算。

## 实现要点

1. `regimen_runner.py` 新增 `precompute_sustained_divisors(regimens, step_size_sec, total_steps, sim_start_date)`：
   遍历所有 `mode == 'sustained'` 的事件，按上表算出 `n_steps`（四舍五入取整，最小为 1），
   写入事件副本的 `_n_steps` 字段。
2. `apply_regimens` 中 sustained 分支：`current + float(ev.get('value', 0)) / ev.get('_n_steps', 1)`。
3. 调用方改动（在主循环外调用一次预计算）：
   - `session_manager.py`：两处仿真循环（实时/批量）入口处。
   - `optimizer_engine.py`：`_run_sim` 调用前（`total_steps` 已在该函数可用）。
4. `docs/model.md` "mode: sustained" 小节补充 value 语义说明 + 换算公式 + 示例。

## 影响与后续验证

- **`models/scenarios/social/ad1945_jp_hiroshima_nurse_nosim_noopt.yaml`**：`care_intensity`、
  `self_protection`、`rest_hours` 的 `optimize.value` 边界数值含义从"每小时值"变为"窗口总量"，
  需要按对应窗口的 `N_steps` 重新换算边界（例如原边界 `[0.0, 3.0]` 若窗口为 24 步，
  新边界约为 `[0.0, 72.0]`，具体取决于物理含义），并重跑一次 `--opt` 确认 `feasible: 100%` 仍成立。
- **诸葛亮疲劳 scenario**：用新语义（"这段时间的工作总量"）重新填写 `value`，与本 ADR 一起验证。
- 不影响任何现有 pulse-only 模型（`N_steps=1` 时数值不变）。

## 与 0100 的关系

[0100](0100-2026-06-11_sim_unify-pulse-sustained-time-interval.md) 提出的"统一区间表示"
复用本 ADR 的 `N_steps` 除法逻辑（pulse 作为 `N_steps=1` 的特例），因此**本 ADR 应先于 0100 实施**。
