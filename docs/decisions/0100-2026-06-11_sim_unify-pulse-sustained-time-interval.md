# 0100 — 统一 pulse/sustained 为时间区间 [start,end)；GUI 取消 full day / time / sustained 三态

**日期**：2026-06-11（补充：2026-06-16）
**状态**：✅ 已完成（schema/引擎/GUI/T2/CLI 路径全部实施；旧字段向下兼容已移除）
**类别**：仿真引擎 / 优化器 schema / GUI

---

## 背景

[0098](0098-2026-06-11_sim_optimizer-schedule-sustained-mode.md) 引入 `mode: sustained` +
`time_range` 作为 pulse（单点 `time`）之外的第二套机制。这带来三重冗余：

1. **schema 层**：一个 schedule 条目要在 `time` 单点 vs `mode:sustained`+`time_range` 区间
   之间二选一，两套字段、两套校验逻辑。
2. **GUI 层**：`SimSetupTab` 已有 `time` 开关（pulse 单点）、新增的 `sustained` 开关，
   以及"不写 `time_range` = 全天生效"的隐式第三态——三者语义有重叠，用户需要先理解
   "我要的是单点/区间/全天"这个分类，才能知道该开哪个开关。
3. **K×4 理论**：iCal 式日历事件本质是 `[start, end)` 区间（两个时间点），pulse 把它
   压缩成单点是一种特例化，但现状把"特例"和"通用形式"做成了两套并行字段。

## 决策

### Schema：单一字段对 `time_start` / `time_end`

子日时间统一用一对 `"HH:MM"` 字段表示区间 `[time_start, time_end)`，取代 `time` + `mode` + `time_range`：

| 取值 | 含义 |
|---|---|
| `time_end == time_start` | **pulse**：零宽区间，`N_steps=1`（按 0099 公式，`value` 原样写入该 step） |
| `time_end != time_start`（不跨越 00:00–24:00 全程） | **sustained**：区间内每个 step 按 `value/N_steps`（0099） |
| `time_start="00:00", time_end="24:00"` | **全天**：`[0,24)` 全覆盖，是 sustained 的一个特定取值，不是单独状态 |

三者是同一对字段在数轴上的不同位置，**不再是三个独立的开关/分支**——pulse、sustained、全天
统一走 0099 的 `value/N_steps` 路径（pulse 是 `N_steps=1` 的特例）。

`days`（星期几过滤）、`date_range`（日历区间）字段不变，与 `time_start/time_end` 正交。

### 向下兼容（已废弃，2026-06-16 移除）

旧字段（`time`、`mode: sustained`、`time_range`）**不再受支持**，引擎不再做字段映射。
旧 YAML 文件须手动更新，对应关系如下：

| 旧写法（不再支持） | 等价的新写法 |
|---|---|
| `time: "HH:MM"` | `time_start: "HH:MM"`（`time_end` 省略默认同值 = pulse） |
| `mode: sustained` + `time_range: [a, b]` | `time_start: a, time_end: b` |
| `mode: sustained`（无 `time_range`） | `time_start: "00:00", time_end: "24:00"` |

### GUI：单一"起始时间"控件 + 可选"结束时间"

- 每个 input event 始终显示一个"起始时间"输入框（取代现有 `time` 开关）。
- "结束时间"默认等于起始时间，视觉上折叠/灰显（呈现为单点，即 pulse）；
  用户拖动/填写使其不同 → 自动展开为区间（即 sustained），无需单独勾选 "sustained" 开关。
- 将结束时间设为跨越 `00:00–24:00`（即 `time_start="00:00", time_end="24:00"`）即表示"全天"，
  不需要单独的"全天"勾选框——全天只是区间宽度的一种取值。
- **取消**：`sustained` toggle、`time` 开关的"仅单点"含义、"不填 time_range = 全天"的隐式状态。
  GUI 状态数从"3 种独立开关的组合"降为"1 对时间字段的相对关系"，理解负担降低一个维度。

### K×4 / x 向量编码的影响

- T1（value）：不变，语义按 0099 改为"区间内总量"。
- T2（时间）：从"单点 1 维 / sustained 的 `time_range` 2 维（不可优化）"统一为：
  - 仅优化 `time_start`，`time_end - time_start`（区间宽度）固定不变 → **1 维**（最常见情况，
    例如"诸葛亮今天几点开始工作"，工作时长固定）。
  - `time_start`、`time_end` 独立优化 → **2 维**（例如"护理强度的起止时刻都待搜索"）。
  - 两端都优化 + 各自上下界 → opt 输入侧为 4 个数（`start_lo,start_hi,end_lo,end_hi`），
    对应用户描述的"4 个值"。
- `docs/model.md` 的 x 向量编码表（927-938 行）需要新增"区间宽度是否参与优化"这一分支；
  多数场景宽度固定，维度不增加，只有显式声明"优化时长"的条目才进入 2/4 维分支。

## 实施前提与范围

**依赖 0099 先实施**：`N_steps` 除法逻辑是本 ADR 的基础（pulse 复用 `N_steps=1` 路径），
且 0099 已经要求"预计算 N_steps"的改动点（regimen_runner + 调用方），本 ADR 在此基础上
只需把"区间从哪些字段读取"换成 `time_start/time_end`，不需要重新设计计算路径。

涉及范围（实施时逐项处理）：

- `sim_engine`：`regimen_runner.py`（区间解析）、`routes/simulation.py`（schema 字段）、
  `optimizer_engine.py`（x 向量编码、`_build_regimen_events`）。
- `sim_gui`：`types.ts`（`InputEvent` 字段）、`Simulator.tsx`（YAML↔state 映射）、
  `SimSetupTab.tsx` / `OptSetupTab.tsx`（GUI 控件）、`optUtils.ts`（编码）、各语言 locale。
- `docs/model.md`：K×4 章节、x 向量编码表、"mode: sustained" 小节（删除/合并为区间小节）。
- `papers/s5`（K×4 控制理论）：术语从"K×4"调整为"K×(可变维度)"或保留 K×4 作为
  "T1+T2(start only)"的常见情形说明。

## 不做的事

- 不引入固定的"最小区间宽度"（如 30 分钟）。区间最小宽度按 `step_size` 自然定义
  （`time_end == time_start` 即 `N_steps=1`），与"step 只管精度"的原则一致，
  避免引入与 `step_size` 无关的第二套时间粒度。
- 不引入 "intensity/rate" 概念——`value` 永远是"区间内总量"（同 0099），
  与 ADR 0092 的裸单位规则一致。

## 实施记录（schema/引擎部分）

- `regimen_runner.py`：新增 `_normalize_time_interval(ev) -> (time_start, time_end)`，
  按本 ADR 的等价表把 `time` / `mode: sustained`+`time_range` / 显式 `time_start`+`time_end`
  统一映射为一对 `"HH:MM"` 字符串；`_time_range_day_seconds` 改为接收
  `(time_start, time_end)`；`precompute_sustained_divisors` 与 `apply_regimens`
  均按 `time_start == time_end`（pulse）/ `!=`（sustained，含全天）分支，
  不再依赖 `mode` 字段判断。
- `routes/simulation.py`：`RegimenEventData` 新增 `time_start`/`time_end`
  （`Optional[str]`），与 `time`/`mode`/`time_range` 并存，按解析优先级生效。
- `optimizer_engine.py`：`fixed_events_map` 与 `_build_regimen_events` 的
  `d0`/`ev2` 透传 `time_start`/`time_end`（若 YAML 条目提供）。
- `docs/model.md`：新增"统一区间表示：time_start / time_end"小节（含等价表、
  向后兼容映射），`mode: sustained` 小节标注为旧格式但仍受支持，x 向量编码
  小节注明 T2 多维重设计未实施。
- 验证：`models/test/test_sustained_mode.yaml`（`--sim`/`--opt`）、
  `models/scenarios/social/ad1945_jp_hiroshima_nurse_nosim_noopt.yaml`（`--opt`）
  数值结果与改动前一致；旧 YAML 无需修改。

## 实施记录（GUI 部分）

- `types.ts`：`InputEvent` 删除 `time`/`timeEnabled`/`sustained`/`timeRangeStart`/
  `timeRangeEnd`，新增 `timeStart`/`timeEnd: string`（始终有值；相等=pulse，
  不等=sustained，含 `"00:00"~"24:00"` 全天）。
- `simUtils.ts`：新增 `normalizeTimeInterval(raw)`（镜像后端
  `_normalize_time_interval` 的等价表，用于 YAML/会话 → `{timeStart, timeEnd}`）
  与 `migrateInputEvent`/`migrateInputEvents`（旧版 localStorage 会话的
  `time`/`timeEnabled`/`sustained`/`timeRangeStart/End` → `timeStart`/`timeEnd`
  迁移，已迁移过的事件原样返回）；`xToInputEvents` 的事件匹配、新建、T2 slot
  写回均改用 `timeStart`/`timeEnd`。
- `Simulator.tsx`：YAML↔state 各映射点（`schedList`/`schedDict`/`plan.schedules`/
  `optimizer.schedules` 决策项匹配/会话恢复/新建事件默认值/Pareto 标签）统一改用
  `normalizeTimeInterval`/`migrateInputEvents`/`timeStart`/`timeEnd`。
- `optUtils.ts`：`buildOptSchedules` 用 `isPulse = ev.timeStart === ev.timeEnd`
  统一三路 `mode='sustained'`/`time_range`/`time` 分支为 `entry.time_start`/
  `entry.time_end`；T2（`isPulse && ev.optimizeTime`）分支保持 `optBlock.time`/
  `time_step`，不发送 `time_start`/`time_end`（避免与后端区间解析优先级冲突）。
- `useSimulation.ts`：regimen payload 三处统一为
  `{ id, time: ev.timeStart, value, time_start: ev.timeStart, time_end: ev.timeEnd }`。
- `SimSetupTab.tsx`/`OptSetupTab.tsx`：删除"时"/"续"开关与"每天"提示，新增
  始终显示的"起始时间 → 结束时间"控件对；`timeStart===timeEnd` 时结束时间
  灰显/虚线（pulse），编辑结束时间使其不同即变为 sustained，并提供折叠按钮
  （×）重置回 pulse。`OptSetupTab.tsx` 中 T2（`opt` toggle + 时间窗 + step
  选择器）仅在 pulse 态显示，逻辑与字段名不变。
- 4 个 locale（en/zh-CN/zh-TW/fr）：删除 `tog.time`/`tog.time_tip`/
  `tog.sustained`/`tog.sustained_tip`/`setup.daily`，新增
  `time_start_tip`/`time_end_tip`/`time_collapse_tip`。

## 实施记录（T2 x 向量重设计）

- **schema**：`optimize.time` 改名为 `optimize.time_start`（旧名仍受支持，作为别名）。
  仅写 `time_start` → 1 维（区间宽度固定，`time_end` = 搜索后 `time_start` + 原宽度）；
  额外写 `optimize.time_end` → 2 维（起止独立搜索）。未实现"4 维（各自带独立上下界）"——
  ADR 草案中的"4 个数"对应的就是 2 维场景下两个窗口各自的 `[lo,hi]`，并非额外维度。
- `optimizer_engine.py`：新增 `_hhmm_to_min`/`_shift_time` 辅助函数；`var_specs`
  的 `kind='time'` 拆分为 `'time_start'`/`'time_end'`；解码时 `d0` 预计算
  `_width_min`（= 条目自身 `time_end - time_start`）与 `_time2dim`
  （是否声明了 `optimize.time_end`）；`time_start` 解码后若非 2 维，
  按 `_shift_time` 推算 `time_end`；输出 `ev2` 始终带 `time_start`/`time_end`
  （及兼容字段 `time = time_start`）。
- `optUtils.ts`：`buildOptSchedules` 始终透传 `entry.time_start`/`entry.time_end`
  作为宽度模板；`ev.optimizeTime` → `optBlock.time_start`；sustained 事件下
  新增 `ev.optimizeTimeEnd` → `optBlock.time_end`（2 维）。
- `types.ts`：`InputEvent` 新增 `optimizeTimeEnd`/`timeEndWindowStart`/`timeEndWindowEnd`。
- `OptSetupTab.tsx`：T2 `opt` toggle 对 pulse/sustained 均显示（不再仅限 pulse）；
  sustained 且 `optimizeTime` 时新增"终"（`time_end`）toggle 行，控制是否独立搜索区间终点。
- `simUtils.ts`：新增 `hhmmToMin`/`shiftTime`（镜像后端），`xToInputEvents` 的 T2
  分支按 1/2 维分别消费 1/2 个 x 分量。
- `Simulator.tsx`：YAML→state 的 `optimize.time_start`/`time_end` 解析（含
  `optimize.time` 旧别名兼容）。
- `docs/model.md`：T2 小节改写为 1/2 维 schema 说明，x 向量编码表新增对应分支。
- 4 个 locale：新增 `sim.opt.tog.time_end`/`time_end_on_tip`/`time_end_off_tip`/
  `time_end_fixed_hint`。

## 实施记录（papers 部分）

- 经核查，S1 §3.2 的形式化定义 $R_j = \{(t_k, d_k, p_k, v_k)\}$、$4KM$ 维搜索空间表述
  本身与本 ADR 兼容：$t_k$（开始时刻）即 `time_start`，子日生效宽度
  $w_k$（`time_end - time_start`）多数场景固定不变，不增加决策变量数。
  因此**未采用"K×4 → K×(可变维度)"的整体改名**，而是在 S1 §3.2 定义 1 后新增
  "补充说明（子日生效区间）"段落：明确 $w_k$ 的语义，并说明仅当区间起止均独立
  搜索时该分段贡献第 5 个决策变量（K×5 扩展，罕见情形）。S3 定义 2 同步加注指向
  该补充说明。S2/S4 仅非形式化引用 K×4，无需改动。S5（尚未起草）写作时需纳入该扩展。

旧字段移除后，旧 YAML 文件需手动更新（参见"向下兼容"映射表）。

## 实施记录（CLI 路径，2026-06-16 补充）

原实施仅覆盖 API/GUI 路径（`apply_regimens`）；CLI 路径（YAML 文件直接跑仿真）的
`_apply_schedules` + `InputSchedule`/`SchedulePoint` 机制不支持 `time_end` 和 sustained。
本次补充将 CLI 路径对齐 GUI，两条路径均走 `apply_regimens`：

- `model_structure/loader.py`：`_parse_schedule_entries` 不再展开绝对时间点
  （`InputSchedule`/`SchedulePoint`），改为输出 regimen 兼容格式的 list，每条 entry
  变成 `{variable, events: [{time_start, time_end, value, days?, valid_start?, valid_end?}]}`。
  结果存入 `model.plans[plan_id]`（`List[dict]`）和第一个 plan 存入 `model.schedule_entries`。
  旧字段 `time:` 的读取一并删除（不再向下兼容）。
- `model_structure/core.py`：新增 `self.schedule_entries: list = []` 属性。
- `simulator_engine.py`：`run_simulation` 在步进循环前调用
  `precompute_sustained_divisors`，每步先调 `apply_regimens` 再调 `model.step()`，
  与 `session_manager.py` 的 GUI 循环完全对称；`run_simulation_all_plans` 改为
  设置 `model.schedule_entries` 而非 `model.schedules`。
- `_apply_schedules` 保留，但仅处理 `daily_inputs`（绝对时间 `InputSchedule` 对象）；
  plan-based schedule entries 已移出 `self.schedules`，不会双重计算。
- `docs/model.md`：`mode: sustained` 小节标注为"已废弃（不再支持）"；
  旧字段映射表措辞从"仍受支持"改为"旧写法（不再支持）"。
- 验证：`test_plans`（三 plan、pulse）、`test_sustained_mode`（pulse + sustained，
  含跨午夜窗口 `20:00~08:00`）两个测试模型通过 `run_simulation_all_plans`，
  结果与 GUI 路径预期一致（pulse 5 天 cumulative_output=300；sustained 5 天=60）。
