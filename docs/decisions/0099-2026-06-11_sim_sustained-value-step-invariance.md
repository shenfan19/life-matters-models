# 0099 — sustained 模式 value 语义修正：窗口总量 / N_steps（step-size 不变性）

**日期**：2026-06-11
**状态**：⚠️ N_steps 公式已被 [0131](0131-2026-07-13_model_sustained-value-per-day-not-per-span.md)（2026-07-13）取代——
"整个生效窗口内的总量"改为"每个匹配日独立满额"，`date_range`/`days` 覆盖多少天不再参与
`N_steps` 计算，只是命中过滤器。本 ADR 的另一条不变量（`step_size` 只影响精度，不影响结果）
被 0131 保留。
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

## 实现记录

- `regimen_runner.py`：新增 `_time_range_day_seconds`、`_n_active_days`、
  `precompute_sustained_divisors`；`apply_regimens` 的 sustained 分支按
  `value / ev['_n_steps']` 写入（pulse 事件 `_n_steps` 缺省为 1，行为不变）。
- `optimizer_engine.py`（`_run_sim`）、`session_manager.py`（`start_session`）：
  在仿真循环开始前各调用一次 `precompute_sustained_divisors`。
- `docs/model.md`："mode: sustained" 小节新增"value 语义：窗口总量，按 N_steps 自适应分摊"说明。

## 影响与验证

- **内部一个场景文件**：4 个 sustained
  schedule 条目的 `optimize.value` 已按 `N_steps`（144/264/360/360）重新换算
  （`[0,3]→[0,432]`、`[0,2]→[0,528]`、`[0,1]→[0,360]`、`[0.1,0.8]→[36,288]`）。
  `--opt` smoke test（pop=8, gen=2）：`success=True`，5 个解，`x` 落在新边界内，
  `f`（patients_saved / health）数值合理，约束 `health>=10` 满足。
- **`models/test/test_sustained_mode.yaml`**：`work_rate`/`recovery_rate` 的
  `optimize.value`/`value` 按 `N_steps=60` 重新换算（`×60`），使每步的
  `work_rate`/`recovery_rate` 变量值（从而轨迹）与改动前完全一致。
  `--opt` smoke test（pop=10, gen=3）：`success=True`，10 个解，
  `cumulative_output`/`fatigue` 的 work/fatigue 权衡符合预期。
- 不影响任何现有 pulse-only 模型（`N_steps=1` 时 `value/1=value`，数值不变）。

## 已知范围外问题（未在本 ADR 处理）

- **`simulation.schedules`（`_apply_schedules`，forward `--sim`/`plans` 路径）不支持
  `mode: sustained`**——该路径是独立实现（`model_structure/simulation.py`），只认
  `pulse`/`step`/`linear` 插值，不读取 `mode`/`time_range`/`_n_steps`。
  `mode: sustained` 目前**仅在 `optimizer.schedules` 与 GUI regimen 路径
  （均经过 `apply_regimens`）生效**。如果某个 forward-sim-only 场景
  （不跑 `--opt`）需要 sustained 输入，需要单独的 ADR 把 `_apply_schedules`
  也接入 `precompute_sustained_divisors`/`apply_regimens` 的逻辑，或复用同一套
  N_steps 计算。
- 冒烟测试中发现一个与本 ADR 无关的预存在 bug：`run_optimizer(progress_callback=None)`
  时 NSGA-II 因 `callback=None` 被当作可调用对象触发 `TypeError`
  （`optimizer_engine.py` `_run_nsga2` 附近，`_cb = _ProgressCb() if progress_callback else None`
  后传给 `pymoo_minimize(..., callback=_cb)`）。GUI/CLI 路径目前总是传入回调，
  实际未受影响，仅在脚本直接调用 `run_optimizer()` 不传回调时触发。

## 与 0100 的关系

[0100](0100-2026-06-11_sim_unify-pulse-sustained-time-interval.md) 提出的"统一区间表示"
复用本 ADR 的 `N_steps` 除法逻辑（pulse 作为 `N_steps=1` 的特例），因此**本 ADR 应先于 0100 实施**。
