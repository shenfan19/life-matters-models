# 0100 — 统一 pulse/sustained 为时间区间 [start,end)；GUI 取消 full day / time / sustained 三态

**日期**：2026-06-11
**状态**：🟡 部分实施（schema/引擎统一已完成；GUI 与 T2 x 向量重设计未实施）
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

### 向后兼容（旧 YAML 数值不变）

- 旧 `time: "HH:MM"` → 等价于 `time_start = time_end = "HH:MM"`（pulse，`N_steps=1`，数值与现状完全一致）。
- 旧 `mode: sustained` + `time_range: [a,b]` → 等价于 `time_start=a, time_end=b`。
- **不需要重新仿真任何现有模型**——schema 解析层做字段映射即可，输出数值不变。

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

## 未实施部分

- **GUI**：`sim_gui` 的 `types.ts`（`InputEvent` 字段）、`Simulator.tsx`
  （YAML↔state 映射）、`SimSetupTab.tsx`/`OptSetupTab.tsx`（控件改造为
  "起始时间 + 可选结束时间"）、`optUtils.ts`、各语言 locale 仍使用
  `time`/`mode: sustained`+`time_range`。
- **T2 x 向量重设计**：`optimize.time_start`/`time_end` 的 1/2/4 维编码
  （区间宽度固定 only-start / 双端独立 / 双端各自带上下界）、
  `_build_regimen_events` 对应解码逻辑、`docs/model.md` x 向量编码表的
  对应分支。
- **papers/s5**：K×4 → K×(可变维度) 的术语调整。

由于向后兼容，旧模型无需因本 ADR 重新仿真；上述未实施部分可作为独立任务排期。
