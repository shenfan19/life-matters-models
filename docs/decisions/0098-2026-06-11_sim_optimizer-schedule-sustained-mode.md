# 0098 — optimizer.schedules 新增 mode: sustained（子日步长持续输入）

**日期**：2026-06-11
**状态**：✅ 已实施
**类别**：仿真引擎 / 优化器 schema

---

## 背景

`optimizer.schedules` 的默认调度是 **pulse 模式**（`regimen_runner.apply_regimens`）：
每个 step 开始时所有受控变量先清零，仅 `time: "HH:MM"` 命中的那一个 step 写入 `value`。

对于 `step_size: day`（或更长，恰为 86400 秒整数倍）的模型，配合 `days: [Mon..Sun]` 全勾选，
该机制等价于"每天都生效"，可以表示日级持续输入。

但对于 `step_size: hour`/`minute` 的模型，单条 `time: "HH:MM"` 调度只在每天某一个小时/分钟生效，
其余 23+ 小时该变量被清零为 0。这使得"持续强度"、"持续防护水平"、"持续休息比例"等
在多个连续 step 上应保持恒定的决策变量无法表达——在迁移一个 `step_size: hour`（360 步）
场景文件的废弃 `variables_to_optimize`/`maps_to` schema 时发现此限制。

## 决策

新增两个可选字段，向后兼容（不写则保持原 pulse 语义）：

- `mode: sustained`：调度事件不依赖 `time` 单点触发；只要满足 `days`/`date_range` 过滤，
  该 step 即生效（值不被清零）。顶层 `time:` 字段在此模式下被忽略。
- `time_range: ["HH:MM", "HH:MM"]`（可选，需配合 `mode: sustained`）：进一步限定每天内的
  生效时间窗，用于子日内的非整日区间（例如 `[0,8)` 小时）。

修改文件：`sim_engine/src/regimen_runner.py`（`apply_regimens` 新增 sustained 分支）、
`sim_engine/src/optimizer_engine.py`（`_build_regimen_events`/`fixed_events_map` 透传 `mode`/`time_range`）。

文档：`docs/model.md` 新增 "mode: sustained" 小节。

## 验证

内部一个场景文件：将三个决策变量（按 `date_range` 拆两段的一个 + 另两个）改为
`mode: sustained` 后，`--opt` 运行 `feasible: 100%`（gen3 起），帕累托前沿呈现真实的
双目标权衡。

## 适用范围 / 后续

- 适用于其余子日步长模型（`ad079_it_pompeii`、`ad1666_uk_issac_newton`）的 schema 迁移：
  - ad079 的 `[0,8)`/`[8,19)` 小时窗口不与日边界对齐，需配合 `time_range` 使用。
  - ad1666 中 `@ bedtime` 等依赖运行时状态的符号引用仍超出本扩展范围，需另行设计。
- `mode: sustained` 仅新增分支，不改变现有模型的 pulse 行为，无需迁移已有 YAML。
