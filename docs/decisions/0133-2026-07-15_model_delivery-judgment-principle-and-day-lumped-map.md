# 0133 — `delivery: total/level` 判断原则 + "day-lumped map" 反模式识别

**日期**：2026-07-15
**状态**：✅ 判断原则已决定；反模式修复（sleep_hours/bedtime_hour 重构）未实施，见下方 task 链接
**类别**：建模方法论 / regimen schema（延伸 [0132](0132-2026-07-14_model_sustained-delivery-total-vs-level.md)）

---

## 背景

ADR 0132 引入 `delivery: total | level`，解决了一个数值症状——sustained 窗口内的读数不该
被 `N_steps` 稀释。但没有回答"建模者该怎么判断一个 `type: input` 变量该用哪个 `delivery`"，
现有表述（"下游被累加 vs 被当系数直接读"）不够精确，容易掩盖一类更深层的问题：某些变量
表面上"需要 level"，实际原因不是这个变量本身的物理性质，而是下游公式本身写成了粗粒度
（`step_unit: day`）的一次性地图，而不是逐步可积的动力学方程——`delivery: level` 在这种
情况下只是把"一天算一次的答案"重复分发给每个更细的 step，本身不是错误的读数，但也不是
真正的逐步积分。

2026-07-15 `life-matters-reference-engine` 会话逐一核对了全仓库全部 `delivery: level` 用例（`running_2026.yaml`
的 `training_intensity`/`pace`；内部一个更早场景文件的三个决策变量；
`sleep_schedule_sim.yaml`/`burnout_allostatic_sim.yaml` 的
`sleep_hours`/`bedtime_hour`/`nutrition_score`），发现
这些用例能清楚分成两类，只有其中一类是"这个变量物理上只有 level 这一种合理解读"，
另一类是"公式写法选错了粒度，靠 level 打了补丁"。

## 决策

### 1. 术语澄清（避免后续讨论混淆三个独立的轴）

| 概念 | 所属层 | 管什么 |
|---|---|---|
| `sustained` | regimen 输入机制（ADR 0127） | `type: input` 唯一的交付机制；pulse（`time_start==time_end`，`N_steps=1`）是它的特例，不是与之并列的第二种模式 |
| `delivery: total \| level` | regimen 条目字段（ADR 0132） | sustained 窗口命中时，每格该交付 `value/N_steps`（total）还是 `value` 本身（level） |
| `step_unit` | **formula 的 `dynamics` 块字段** | 这条公式自己的 `step` 变量对应多长时间，与 regimen/delivery 完全无关，是公式层、不是输入层的概念 |

三者分属不同层，可能同时出现在同一个变量身上，但互不是对方的别名或子集。

### 2. `delivery` 判断规则（取代 ADR 0132 原有"累加 vs 直接读"表述）

**规则 A**：该变量在下游公式里是否作为"系数/瞬时状态"参与运算，且这条公式写在**原生 step
粒度**（不声明粗于 `simulation.step_size` 的 `step_unit`）上？
- 是 → `delivery: level`，且这是唯一正确、无需进一步处理的终态。
  例：`training_intensity`/`pace`（`heart_rate_response`/`fatigue_accumulate` 逐分钟原生读取）、
  `care_intensity`/`self_protection`/`rest_hours`（`radiation_accumulation`/`rest_slows_ars`
  逐小时原生读取）。这类变量物理上没有"总量"这个维度（问"training_intensity 的总量是
  多少"没有意义），不存在二义性。
- 该变量本身就是"这次投入了多少"，被下游 state 累加？→ `delivery: total`（默认）。

**规则 B（新增）——"day-lumped map" 反模式检验**：如果一个变量为了表现出"level"效果，其
下游公式必须用一个粗于 `simulation.step_size` 的 `step_unit`（如 `day`）把多个 simulation
step 的净变化一次性算出来，再靠 `delivery: level` 把这个"一次性答案"原样重复分发给每个
更细的 simulation step——这是设计异味信号，**不能靠调整 `delivery` 解决**，说明这个变量
选错了原语。正确方向：拆成"原生粒度可读的瞬时指示量（真 level，如 `is_asleep`）+ 由它
累积出的衍生 state（真 total，如 `sleep_hours_today`，与 `lm_score` 同一模式）"，公式相应
改写为原生 `step_unit` 的真正 ODE。

当前命中：`sleep_hours`/`bedtime_hour`（`sleep_schedule_sim.yaml`/`burnout_allostatic_sim.yaml`，
`sleep_pressure_dynamics` 声明 `step_unit: day` 却用全天 sustained + `delivery: level` 逐小时
重复读取同一个日常量）。`nutrition_score` 疑似同一模式，未逐条核实。

## 与已有原则的关系

- **延伸而非推翻 ADR 0132**——0132 的字段定义、引擎实现（`schedule_runner.py`）、既有迁移
  记录全部保持不变，本 ADR 只是补一层"该怎么判断用哪个、什么时候该怀疑 `delivery` 治标
  不治本"的方法论。
- **呼应 Banister `*step` 排查的教训**（ADR 0131 前身排查记录）："只在唯一一种粒度下测过，
  问题从未暴露"——规则 B 本质上是把这条教训沉淀成一条可执行的检验规则，防止同类反模式在
  其他模型里复现而不自知。
- **对 `draft_s1_numerical_consistency.md`（S1 论文候选小节）的影响**：该节论证的"数值
  一致性保证"只对规则 A 类模型（公式写在原生粒度）成立；规则 B 类模型（day-lumped map）
  的读数虽然不随 `step_size` 漂移（ADR 0132 已验证），但方程本身不会随步长细化而收敛到
  更精确解——这是两种不同强度的"一致性"，论文需要区分说明，已记入 task（见下）。

## 现状与代价（未实施部分）

- 规则 A/B 作为判断原则，本 ADR 已确定；写入 `docs/model.md`"delivery 判断规则"一节待执行。
- 规则 B 识别出的 `sleep_hours`/`bedtime_hour` 反模式**不在本 ADR 修复范围**——修复方案
  （`is_asleep` 状态量 + 脉冲对，取代 duration 型 input）已记录为独立 task，涉及重写公式、
  重新校准 S3 论文数字，是否/何时执行留给该 task 单独决定，详见内部任务
  `2026-07-15_task_sleep-model-native-step-reform.md`。
- 当前 `delivery: level` 对 `sleep_hours` 的数值修复（ADR 0132）不受影响、依然有效——已
  验证步长鲁棒（1h/30min/15min 回归不发散），只是不解决更细粒度的动力学表达问题。

## 实现

- `docs/authoring/regimens_and_optimization.md`（`docs/model.md` 的后继路径）：已补"delivery
  判断规则：结构位置检验，兼 day-lumped map 反模式识别（ADR 0133）"一节，规则 A + 规则 B 完整
  版本，替换掉此前偏模糊的"累加 vs 直接读取"表述；同一份文档另加了一节"概念基础"，把
  `value`/`delivery`/`days`/`date_range` 统一到广延量/强度量框架下（2026-08-25，随 S1 论文
  §4.4 同批改写一并落地，见该节的等价论证）。
- `draft_s1_numerical_consistency.md` / S1 论文 §4.4：论证只覆盖规则 A 类模型（公式写在原生
  粒度）；规则 B 类模型（day-lumped map）读数虽不随 `step_size` 漂移，但方程本身不收敛到更
  精确解，论文这次改写为只保留一句前提陈述，不展开反模式的完整技术描述（2026-08-25，用户
  审阅后判断论文正文应弱化举例，完整反模式描述保留在本 ADR 和上述 authoring 文档）。
  `sleep_schedule`/`burnout_allostatic` 论文 `limitations` 段落尚未补充更精确描述，留待接触
  这两篇论文时处理。

## 已知局限（不在本 ADR 处理）

- 全仓库 AST 扫描"还有没有其他模型命中 day-lumped map 反模式"未做，当前只人工核对了已知
  的 `delivery: level` 全部命中点（10 个文件）。
- `nutrition_score` 是否属于规则 B 命中（vs 规则 A）未逐条核实下游公式，留待下次触碰
  `burnout_allostatic_sim.yaml` 时确认。
