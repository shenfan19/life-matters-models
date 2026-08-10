# 0132 — sustained regimen 新增 `delivery: total | level`，区分"总量摊分"与"恒定水平"

**日期**：2026-07-14
**状态**：✅ 已实施
**类别**：仿真引擎 / regimen schema

---

## 背景

ADR 0098（2026-06-11，sustained 模式最初提出）的原始动机是表达"持续救治强度"、"持续防护
水平"、"持续休息比例"这类**应在多个连续 step 上保持恒定**的决策变量——字面就是"水平"语义。

但同一天稍后的 ADR 0099 把 sustained 的 `value` 重新定义为"整个生效窗口内的总量，除以
`N_steps` 摊到每个命中 step"——这是为了解决另一个真实但不同的问题（`step_size` 变化不应
改变总贡献量）。这次重新定义**从未回头检查是否还满足 ADR 0098 自己举的例子**："应保持恒定"
的水平，在除以 `N_steps` 之后，读数本身会随 `step_size`（进而 ADR 0131 起还包括匹配天数）
反比例变化——在模型只曾在一种固定 `step_size` 下跑过时，这个矛盾从数值上完全看不出来。

2026-07-14 验证 ADR 0131 时，用变步长网格测试更多模型，发现 `burnout_allostatic_sim.yaml`/
`sleep_schedule_sim.yaml` 在非原生 `step_size` 下发散（[[project_sustained_level_vs_dose_gap]]，
已归档内部任务 `2026-07-14_issue_sustained-input-level-vs-dose-semantics-gap.md`）。
根因排查确认：`sleep_hours`/`bedtime_hour`/`nutrition_score`（以及更早的
ADR 0098 自己验证案例里的三个决策变量）在各自公式里都是被当"瞬时水平"直接读
（乘法因子或偏离基线计算），不是当"累加的总量"用——这正是 ADR 0098 最初想表达的语义，
被 ADR 0099 的"总量÷N_steps"规则悄悄覆盖，两条决议自 2026-06-11 起就互相矛盾，只是没被
发现。

Banister 的 `training_load`（真正的日速率，"50 AU/day，总量随天数正比变化"）则确实需要
ADR 0099/0131 的"总量摊分"语义——不是所有 sustained input 都该是水平，两种语义都有真实
存在的必要，问题是过去只有一种可选。

## 决策

regimen 条目新增可选字段 `delivery: total | level`，**默认 `total`**（=现状，完全向后兼容，
不需要迁移任何未受本问题影响的现有模型）：

```yaml
regimens:
  - variable: sleep_hours
    time_start: "00:00"
    time_end: "24:00"
    value: 6.0              # 直接就是目标水平，不需要手算乘 N_steps
    delivery: level          # 每个命中 step 直接交付 value 本身，不除以 N_steps
    days: [Mon, Tue, Wed, Thu, Fri]
```

| `delivery` | 语义 | 每个命中 step 的交付量 | 适用场景 |
|---|---|---|---|
| `total`（默认） | 窗口/匹配日总量，按 ADR 0131 摊分 | `value / N_steps` | 训练负荷、进食总量、给药总量——量本身是"这段时间投入了多少"，随时间累积 |
| `level` | 恒定水平，不摊分 | `value` 本身 | 睡眠时长、就寝时机、饮食质量得分、救治强度、防护水平——量本身是"当前处于什么状态/设定"，不随时间累积 |

`delivery: level` 对单 step 窗口（pulse，`time_start==time_end`）是 no-op——pulse 本来就是
`N_steps=1` 的特例，除或不除结果相同，不产生冲突，两种 `delivery` 在 pulse 上退化为同一件事。

## 与既有原则的关系

- **不是重新引入 ADR 0100 去掉的 pulse/sustained 开关**：ADR 0100 去掉的是一个**冗余**区分
  （"点 vs 窗"完全由窗宽一个数决定，去掉不丢信息）。`total` vs `level` 是一个**新的、独立的**
  维度——Banister 的 `training_load` 和本 ADR 修复的 `sleep_hours`/`care_intensity` 用的是
  完全相同的窗宽（全天），仅靠窗宽无法区分二者，必须显式声明。
- **不是靠变量名约定**（如内部 `_step` 后缀）：2026-07-09 已明确否决"靠变量名让引擎做特殊
  处理"这条路（"变量名自由、引擎不认保留名"的既有原则），本字段是显式写在 regimen 条目上
  的声明，跟 `time_start`/`time_end`/`days`/`date_range` 是同一层级的属性，不是命名约定。

## 实现

- `reference_engine/src/schedule_runner.py`：`apply_schedules` 内 `delta = value/n_steps`
  改为按 `ev.get('delivery', 'total')` 分支，`level` 时 `delta = value` 直接交付。
  `precompute_sustained_divisors` 不变（`_n_steps` 仍然预计算，只是 `level` 分支不使用它）。
- `model_structure/loader.py`：`_parse_schedule_entries` 透传 `delivery` 字段（缺省不写）。
- `optimizer_engine.py`：`_build_regimen_events` 透传 `delivery`（T1 `optimize.value` 的
  搜索边界在 `level` 下直接就是目标水平的上下界，不需要再乘 `N_steps`）。

## 迁移

以下模型的 `value`/`optimize.value` 从"手算乘 N_steps 的总量"还原为直接的水平数值，并加上
`delivery: level`：

- `models/papers/s3/burnout_allostatic/burnout_allostatic_sim.yaml`
  （`sleep_hours`/`bedtime_hour`/`nutrition_score`，21 处）
- `models/papers/s3/sleep_schedule/sleep_schedule_sim.yaml`（同上三个变量，22 处）
- 内部另一个场景文件（同上三个变量，4 处 `optimize.value`）
- `burnout_allostatic_opt_{workoutput,cvdrisk,joint}.yaml`、
  `sleep_schedule_opt_{cognitive,healthrisk,joint}.yaml`（各自独立的 `optimizer.startpoint.regimens`
  边界，同一批变量）

`exercise_min`/`break_min`/`nap_minutes`（脉冲）、`training_load`（Banister，真正的日速率）
不受影响，维持 `delivery: total`（默认，无需声明）。

验证：变步长网格（1h/30min/15min）下 `sleep_hours` 读数保持恒定不再随步长漂移，
`burnout_allostatic_sim.yaml`/`sleep_schedule_sim.yaml` 的 `--sim` 结果与迁移前
（原生 `step_size=1h`）数值完全一致，且在非原生步长下不再发散。完整验证记录见
已归档内部任务 `2026-07-14_issue_sustained-input-level-vs-dose-semantics-gap.md`。

## 已知局限（不在本 ADR 处理）

- `models/` 里是否还有其他模型存在同一种"sustained input 被当水平读"的模式，本次只排查了
  已知触发案例（burnout_allostatic/sleep_schedule/内部另一个场景文件），未对全部 `models/` 做系统性
  AST 扫描——留作后续 task。
