# 0131 — sustained value 语义修正：每个匹配日独立满额，取代 0099 的"整跨度总量"

**日期**：2026-07-13
**状态**：✅ 已实施（取代 0099 的 N_steps 公式，取代 0126 第3条）
**类别**：仿真引擎 / regimen 语义

---

## 背景

ADR 0099 把 sustained `value` 定义为"整个生效窗口内的总量"：`N_steps = 生效窗口总时长 /
step_size`，其中"生效窗口总时长" = `date_range` 覆盖的全部匹配日天数 × 每日窗宽
（`_n_active_days` × `_time_range_day_seconds`）。ADR 0126（2026-07-09）第3条重新确认了
这条规则。这套设计的目标是"`step_size` 只管精度，不改变总贡献量"——在**固定的活跃天数**下，
这个目标确实达到了。

2026-07-13 排查论文 S1 Banister 数值精度验证时，用户指出这套定义有反直觉的副作用：**`value`
的累计总量与"实际匹配了多少天"成反比**——同一个 `value`，`date_range` 覆盖的天数越多，
每天摊到的实际数值反而越小；改变仿真总时长、`date_range`、或 `days` 星期过滤，都会在不改
`value` 的前提下悄悄改变"每天实际交付多少"。这与"日速率"这一常见建模意图（如"每天恒定
50 单位训练负荷，不管这个方案跑多少天"）正好相反：真正的日速率应该是"每天交付量不变，
总量随天数正比例变化"，而不是"总量不变，每天交付量随天数反比变化"。

**关键证据**：`models/test/valid/test_sustained_mode.yaml` 的 `sustained` plan 一直以来的
`label` 字段就写着"same daily totals (work +60/day...) final cumulative_output/fatigue
should match the pulse plan"——但该 plan 从未设 `date_range`（默认覆盖整个 5 天仿真），
按 ADR 0099 的公式，`value: 60.0` 会被除以"整个 5 天窗口的总步数"，导致实际每小时只交付
`60/(5×12)=1/hr`，5 天累计仅 60，而不是 fixture 自己期望且 ADR 0100"实施记录"当年就已经
测出来并如实记录的"sustained 5天=60"——这行记录本身就是"这个 bug 从 ADR 0099/0100 落地
第一天就存在，只是从未有人拿它跟 fixture 自己的文字期望做交叉核对"的直接证据。

## 决策

**sustained `value` 改为"每个匹配日独立交付的窗口总量"，与该条目在整个 `date_range`/
仿真跨度内实际匹配了多少天无关**：

```
N_steps = 单次命中窗口时长 / step_size
        = _time_range_day_seconds(time_start, time_end) / step_size
per_step_value = value / N_steps
```

不再计算"活跃天数"（`_n_active_days` 整个函数删除）。`days`（星期过滤）、`date_range`/
`valid_start`/`valid_end`（日历区间）**降级为纯粹的"是否命中"过滤器**，只决定"这一天要不要
触发"，不再参与 `N_steps` 的计算——语义上与 pulse 事件的 `days`/`date_range` 完全对称
（pulse 从来就是"命中就交付满额 `value`，不命中就是 0"，从未按天数摊分过）。

`state` 公式侧不需要任何改动，继续遵循既有的"`type: input` 变量不乘 `step`"规则。验证：

```
每个匹配日的累计贡献 = per_step_value × N_steps = value
```

与 `step_size` 无关（ADR 0099 的目标保留），也与匹配了多少天无关（新增的不变量）——
`test_sustained_mode.yaml` 的 `sustained` plan 用 `value: 60.0`（不做任何改动）在新规则下
直接给出 60/day、5 天 300 的正确结果，与 `pulse` plan 完全一致，实测验证见下方"影响与验证"。

## 与 0099/0126 的关系

- **取代 ADR 0099** 的 N_steps 公式（"整跨度总量"），保留其"step_size 只管精度"的不变量，
  新增"匹配天数不影响每日交付量"的不变量。
- **取代 ADR 0126 第3条**（"sustained 的 value 表示整个生效窗口内的总量……这要求生效窗口
  总时长在装载阶段就是确定数字"）——该条描述的正是本 ADR 现在改掉的旧规则；ADR 0126 其余
  三条（pulse-decay 定位、多条目覆盖/累加维持现状、`valid_range` 日期对齐独立处理）不受影响。
- 原计划中"给 regimen 补一个新的 `rate` 字段"的方向（见已归档内部任务
  `2026-07-13_task_step-size-adaptive-input-rate-design.md`）
  **不再需要**：本 ADR 修正后，`value` 本身天然就是"日速率"语义，不需要额外并行字段。

## 实现记录

- `reference_engine/src/schedule_runner.py`：删除 `_n_active_days`（连同 `math` import）；
  `precompute_sustained_divisors` 签名简化为 `(schedules, step_size_sec)`（去掉不再需要的
  `total_steps`/`sim_start_date`），`_n_steps` 直接等于 `day_sec / step_size_sec`；
  `apply_schedules` 的 docstring 同步改写"每个匹配日独立满额"。
- 4 处调用点（`session_manager.py`、`reference_engine.py` ×2、`optimizer_eval.py`）同步
  去掉调用时多传的 `total_steps`/`sim_start_date` 实参。
- `docs/model.md`：value 语义章节、N_steps 公式表、窗宽默认规则一节改写为新规则，新增
  多天重复触发的 regimen 示例（区分"总量固定"与"日速率固定"两种历史语义，明确后者是
  现在唯一支持的语义）。

## 迁移：5 个受影响文件的 value/optimize.value

审计范围：`simulation.plans[*].regimens` + `optimization.startpoint.regimens` 中，
`time_start != time_end`（sustained，非 pulse）且旧规则下"匹配天数 > 1"的条目。

**核心判断标准**（逐条目核实，不是无脑除以旧活跃天数）：条目的 `label`/注释是否有
"`= 日速率 × 旧N_steps`"这类痕迹，证明作者当年是按 ADR 0099 手动把日速率乘成了总量——
有则说明当前 `value` 是"被旧规则要求手算出来的总量"，需要除以旧活跃天数换回日速率；
没有（`value` 本身就是作者写下的日速率、只是被 ADR 0099 的 bug 悄悄稀释/加浓）则不touch，
新规则下这个数字自动就是对的。

| 文件 | 处理 | 备注 |
|---|---|---|
| `models/test/valid/test_sustained_mode.yaml` | plan 级 `sustained.work_rate/recovery_rate` **不变**（60.0/24.0）；`optimization.startpoint` 两条目除以旧活跃天数 5（`[60,600]→[12,120]`，`120→24`） | plan 级注释("5/hour"、"same daily totals")证明 60/24 本来就是日速率，是 ADR 0099 的 bug 一直在稀释它，不是作者预乘过；optimization 部分注释明确写着"old per-hour bounds x 60"，证明是预乘过的，要除回来 |
| `models/papers/s3/burnout_allostatic/burnout_allostatic_sim.yaml` | 6 个 plan、21 处 `value` 全部除以各自旧活跃天数（86/62/24） | 每处 label 都带"×2064"/"×1488"/"×576"字样，直接证明是"日速率×旧N_steps"手算出来的总量；除以旧活跃天数后数值验证（见下）与旧引擎+旧值完全一致 |
| `models/papers/s3/sleep_schedule/sleep_schedule_sim.yaml` | 7 个 plan、22 处 `value` 除以各自旧活跃天数（28/20/8） | 同上，label 带"×672"/"×480"/"×192" |
| 内部另一个场景文件 | `optimization.startpoint.regimens` 4 处 `optimize.value` 除以各自旧活跃天数（6/11/15） | 注释"= [0,3] × 144"等直接证明预乘过 |
| `models/test/valid/test_opt_t2.yaml` | **不改**，从"受影响文件"名单中移除 | 初次审计脚本对这两个 T2 条目误判：YAML 里没写 `time_start`/`time_end`，静态看是"全天默认"，但这两条目都配了 `optimize.time_start` 区间搜索；`optimizer_engine._build_regimen_events` 解码时按条目自身 `_width_min`（用同一个默认全天窗口算出的 1440 分钟）重新推算 `time_end`，正好整圈绕回 `time_start`，运行时实际总是退化成 pulse（`time_start==time_end`），从未真正走过 sustained 的 `N_steps` 除法，从来不受 ADR 0099 影响 |

**双重验证方法**：对 `burnout_allostatic_sim.yaml`/`sleep_schedule_sim.yaml` 的迁移，用
`git stash` 切换回"旧引擎代码 + 旧 value"跑一遍 baseline plan（`--sim`），再切回"新引擎 +
新 value"跑同一个 plan，逐字段比对最终状态——`cortisol_chronic`/`cvd_risk`/
`health_risk_index`/`cognitive_performance` 等全部数值精确一致，确认迁移未改变任何已发布
仿真结果的数值行为。`test_sustained_mode.yaml` 用新引擎跑 `sustained` plan，5 天后
`cumulative_output=300.0`，与 `pulse` plan 一致（此前是 60.0，ADR 0100 当年的"实施记录"
记的正是这个被本 ADR 判定为 bug 的数字）。

## 已知后续（不在本 ADR 处理，留给独立 task）

- `test_plan.md` 层1数值精度验证协议补一节"步长收敛性检验"（与解析解对比并列的独立步骤）。
- `burnout_allostatic_opt_*`/`sleep_schedule_opt_*` 三个优化场景需要用修正后的 startpoint
  边界重新跑 `--opt`，核对论文 S3 稿（`c_paper_s3_cn.md`）里已写的具体数字（workoutput/
  cvdrisk/joint 前沿数值、cognitive_performance/health_risk_index 等）是否随之变化——
  这两个场景的 `optimize.value` 语义本来就依赖"每日搜索一个值，工作日/周末各自独立"，
  本 ADR 修正的是"多日期跨度会怎样摊分"这一层，会直接影响这两个场景的 T1 搜索边界数值。
- 详见内部任务 `2026-07-13_task_s3-opt-rerun-after-adr0131.md`（已从已归档内部任务
  `2026-07-13_task_step-size-adaptive-input-rate-design.md`
  的"第二批遗留"拆分为独立仍活跃的 task）。
