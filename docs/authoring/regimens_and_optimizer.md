# 干预方案（regimens）与优化器

## simulation.plans[*].regimens — 时间驱动的 input 序列

仿真输入方案的唯一合法位置是 `simulation.plans[*].regimens`（ADR 0109，字段名见 ADR 0117）。每个 plan 包含一组 regimens 条目，描述该方案中各 `type: input` 变量的时间驱动输入。**所有 input 都是 sustained（ADR 0127）**：每个条目对应一个 `[time_start, time_end)` 生效窗口，命中窗口的每个 step 按 `value / N_steps` 写入，窗口外自动为 0，**每次命中（每个匹配日）独立累计贡献恒等于 `value`**，与 step_size 无关，也与 `days`/`date_range` 让这个条目匹配了多少天无关（ADR 0131，见后文"value 语义"）。不存在一个独立于 sustained 之外、字面意义的"pulse 模式"——窗宽窄到 1 个 step，数值上就是过去说的"pulse"。

**不再支持 `simulation.schedules` 顶层字段**（旧格式，已于 ADR 0109 废弃）。

### 标准格式（单方案模型）

```yaml
simulation:
  start_date: "2026-01-01"
  end_date:   "2026-01-04"
  plans:
    - id: default
      label: "Baseline plan"
      regimens:
        - variable: carb_intake
          time_start: "07:00"
          value: 50.0
          days: [Mon, Tue, Wed, Thu, Fri, Sat, Sun]   # 可省略，缺席 = 每天
          date_range: ["2026-01-01", "2026-01-04"]        # 可省略，缺席 = 全程
          label: "早餐碳水"
        - variable: carb_intake
          time_start: "12:00"
          value: 80.0
          label: "午餐碳水"
        - variable: carb_intake
          time_start: "18:30"
          value: 60.0
          label: "晚餐碳水"
```

### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `variable` | string | ✅ | 必须是 `variables` 中 `type: input` 的变量名 |
| `time_start` | `"HH:MM"` | — | 生效窗口起点（24 小时制）；缺省规则见下方"窗宽默认规则" |
| `time_end` | `"HH:MM"` | — | 生效窗口终点；缺省规则见下方"窗宽默认规则" |
| `value` | number | ✅ | 窗口内的**总量**（不是每步的量），累计贡献恒等于这个数，与 step_size/窗宽无关 |
| `days` | `[Mon…Sun]` | — | 三字母缩写列表；缺席 = 每天都触发 |
| `date_range` | `["YYYY-MM-DD", "YYYY-MM-DD"]` | — | 条目仅在此日历区间内生效；缺席 = 从 `start_date` 到 `end_date` 全程 |
| `label` | string | — | GUI 展示用说明文字 |

#### 窗宽默认规则（ADR 0127）

`time_start`/`time_end` 都不是必填——**都不写，不代表"缺配置"，代表建模者选择了一种明确的
窗宽**，引擎按下面规则解析（不需要建模者自己拼 `time_start`/`time_end`，写清楚意图就够了）：

| 写法 | 生效窗口 | 适用场景 |
|---|---|---|
| 两个都不写 | 全天 `["00:00", "24:00"]` | day-rate 输入（如日总热量缺口、日均摄入量）——这类量本来就没有"发生在哪一刻"这个概念，不该被迫编一个不存在的触发时刻 |
| 只写 `time_start` | `time_end` = `time_start`（单 step 窗口） | 离散事件（进食、给药）——数值上与旧称呼的"pulse"完全相同 |
| 两个都写 | 显式区间 | 子日步长模型里"持续强度/持续防护"这类跨多个 step 的输入 |

不要为了凑出"pulse 效果"而手写 `time_start == time_end`——只写 `time_start` 就是这个效果，
显式写两个相同值反而容易让读者误以为二者独立可调。

### 多条目 vs 多周期

同一变量**可以有多个条目**（如三餐），引擎在同一步内累加所有命中事件：

```yaml
# 三餐：每步最多命中一个，累加结果 = 单餐值（不同时段错开）
# 步长 1 天时：三个条目在同一步内全部命中 → dietary_protein = 0.27+0.27+0.26 = 0.80
```

`date_range` 用于表达**分阶段方案**（如训练周期渐进），不要用"每周重复列条目"替代：

```yaml
# ✅ 正确：用 date_range 区分阶段
regimens:
  - variable: training_load
    time_start: "09:00"
    value: 50.0
    days: [Mon, Tue, Wed, Thu, Fri]
    date_range: ["2026-01-01", "2026-01-28"]   # 基础期 4 周
  - variable: training_load
    time_start: "09:00"
    value: 100.0
    days: [Mon, Tue, Wed, Thu, Fri]
    date_range: ["2026-01-29", "2026-02-25"]   # 强化期 4 周

# ❌ 错误：逐周罗列（冗余，条目数 = 周数 × 2）
regimens:
  - variable: training_load
    value: 50.0
    date_range: ["2026-01-01", "2026-01-07"]   # 第1周
  - variable: training_load
    value: 50.0
    date_range: ["2026-01-08", "2026-01-14"]   # 第2周（与第1周相同，无意义）
```

### 多阶段方案的推荐写法：baseline + 增量（ADR 0126）

引擎对同一变量的多条 regimen 是"每步清零后逐条累加"（不是覆盖）——这个行为本身是对的
（上面三餐累加就是这么设计的），但如果用"每个阶段各写一条完整目标值，靠 `date_range`
首尾相接实现互斥切换"这种写法，**要求每一条都精确写对 date_range，漏写其中一条就等于让
它全程生效，与其余阶段叠加**（真实案例：某三阶段饮食方案的限制期条目漏写 `date_range`，
导致进入后续阶段后限制期目标量仍在叠加，症状分数被系统性拉高）。

推荐换一种写法：一条**不写 `date_range` 的 baseline 条目**（本来就该全程生效，不存在
"忘记设终止日期"的陷阱）+ 若干条**限定 `date_range` 的增量条目**（值是"相对 baseline 的
差量"，不是目标绝对值）：

```yaml
# ✅ baseline + 增量：唯一不写 date_range 的条目本来就该全程生效
regimens:
  - variable: fodmap_intake
    time_start: "00:00"
    value: 22.0                              # baseline：维持期目标量，全程生效
  - variable: fodmap_intake
    time_start: "00:00"
    value: -15.0                             # 限制期相对 baseline 的减量
    date_range: ["2026-01-01", "2026-01-28"]
  - variable: fodmap_intake
    time_start: "00:00"
    value: -2.0                              # 重引入期相对 baseline 的减量
    date_range: ["2026-01-29", "2026-04-01"]

# ❌ 每阶段各写完整目标值：漏写/写错任意一段 date_range 都会导致累加而非替代
regimens:
  - variable: fodmap_intake
    value: 7.0
    date_range: ["2026-01-01", "2026-01-28"]   # 限制期目标量
  - variable: fodmap_intake
    value: 20.0
    date_range: ["2026-01-29", "2026-04-01"]   # 重引入期目标量
  - variable: fodmap_intake
    value: 22.0                                 # 维持期目标量——如果忘记写这一条的 date_range，
                                                 # 会在限制期/重引入期也生效，与其余条目叠加
```

baseline+增量写法里，即使某条增量条目漏写 `date_range`，也只是让增量多算了几天（同量级的
小偏差），不会出现"整个目标值"量级的叠加错误——这是选它而不是"每阶段完整目标值"的原因。

#### opt 端已知局限：阶段切换时机是搜索变量时，增量条目的 `date_range` 无法跟着联动

T4（干预日期范围优化，见后文）可以让"阶段切换的时间点"本身成为搜索变量。但如果同时想用
baseline+增量写法表达多阶段，且某个增量条目的 `date_range` 端点应该"跟着 T4 搜索到的切换
时机走"，**目前引擎不支持这种跨条目的日期联动**——`date_range` 只能是写死的日期，或者
本条目自己的 `optimize.date_range` 搜索窗，不能引用"另一个 regimen 条目搜索到的值"：

```yaml
# T4 搜索"限制期结束在哪天"，但重引入期增量条目的 date_range 起点无法自动跟随这个搜索结果
regimens:
  - variable: fodmap_intake
    value: 22.0                       # baseline
  - variable: fodmap_intake
    optimize:
      value: [-18.0, -10.0]
      date_range:                      # T4：限制期本身的持续时长是搜索变量
        - ["2026-01-01", "2026-01-01"]
        - ["2026-01-22", "2026-03-05"]
  - variable: fodmap_intake
    value: -2.0
    date_range: ["???", "2026-04-01"]  # ❌ 起点无法写成"上面 T4 搜索到的结束日"
```

**当前的应对方式**（接受为受控成本，非引擎缺陷）：opt 端涉及多阶段日期联动的需求，退回
手动限定固定的 `date_range` 搜索窗，由建模者在 Sim 侧确认结果、必要时手动调整 opt 输入的
日期范围后重新搜索，不追求"一次优化自动联动所有阶段边界"。这类 opt 端输入设计的便利性，
留给未来的功能改进或 plugin，不在当前投入范围内（ADR 0126）。

### 与 GUI inputEvents 的关系

`simulation.plans[*].regimens` 是模型加载时 GUI `inputEvents` 的来源：GUI 按 plan 解析 YAML 后填充
`inputEvents`，此后一次仿真 session 实际使用的就是 `inputEvents`（用户可编辑、可被优化结果覆盖）——
不存在"YAML 在每步覆盖 GUI 编辑"的运行时冲突（ADR 0074/0115）。

**建模者须知**：GUI 上对某变量的手动调整一般直接生效；若该变量同时被 `optimizer.startpoint.regimens`
标记为决策变量（`optimize:` 块），优化运行时由优化器接管该变量的取值，与 Sim 面板的手动值是两套独立的
搜索/预览状态。

### 离散输入不写零值点

> `type: input` 的离散量（进食、给药等）无需插入 `value: 0` 的关闭点——窗口外自动为 0。

```yaml
# ✅ 只写非零时刻
- variable: carb_intake
  time_start: "07:00"
  value: 50.0

# ❌ 冗余的 0 值点
- variable: carb_intake
  time_start: "07:30"
  value: 0.0    # 不需要，窗口外自动补零
```

例外：连续速率类变量（如持续泵药 `infusion_rate`）需要保留明确的关闭点。

### 向后兼容：旧字典格式

旧版 dict 格式（`{varName: {interpolation, points: [{time: 秒数, value}]}}`）在引擎中仍可解析，但不再推荐，新模型应使用扁平列表格式。

---

## simulation.plans — 预定义多方案比较

`simulation.plans` 允许建模者在 YAML 中预置多个命名方案，GUI 加载模型时直接呈现为 Plan 列表供多方案并行仿真（F-MPLAN）。

**使用场景**：
- 论文模型（papers/）：将 Pareto 前沿的代表点写成具名方案，读者打开即可比较"肾保护优先"vs"肌肉保留优先"
- 临床对照：预置"指南标准剂量"与"优化剂量"方案，直接展示论文图表对应的输入

**格式**：

```yaml
simulation:
  start_date: "2026-01-01"
  end_date:   "2026-12-31"
  plans:
    - id: "kidney_protect"
      label: "肾保护优先（Pareto 端点）"
      regimens:
        - variable: dietary_protein
          time_start: "08:00"
          value: 0.22
          days: [Mon, Tue, Wed, Thu, Fri, Sat, Sun]
          label: "早餐蛋白质"
        - variable: dietary_protein
          time_start: "12:00"
          value: 0.21
          label: "午餐蛋白质"
        - variable: dietary_protein
          time_start: "18:00"
          value: 0.22
          label: "晚餐蛋白质"
    - id: "balanced"
      label: "临床平衡方案"
      regimens:
        - variable: dietary_protein
          time_start: "08:00"
          value: 0.29
          label: "早餐蛋白质"
        - variable: dietary_protein
          time_start: "12:00"
          value: 0.27
          label: "午餐蛋白质"
        - variable: dietary_protein
          time_start: "18:00"
          value: 0.28
          label: "晚餐蛋白质"
    - id: "muscle_preserve"
      label: "肌肉保留优先（Pareto 端点）"
      regimens:
        - variable: dietary_protein
          time_start: "08:00"
          value: 0.38
          label: "早餐蛋白质"
        - variable: dietary_protein
          time_start: "12:00"
          value: 0.36
          label: "午餐蛋白质"
        - variable: dietary_protein
          time_start: "18:00"
          value: 0.37
          label: "晚餐蛋白质"
```

**字段说明**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | ✅ | 方案唯一标识（小写加下划线） |
| `label` | string | ✅ | GUI 显示名称 |
| `regimens` | list | ✅ | 条目格式同上，每个条目为一个时间驱动输入事件 |

**唯一合法位置（ADR 0109）**：仿真输入方案只允许存在于 `simulation.plans[*].regimens`，不允许顶层 `simulation.schedules`。单方案模型使用 `id: default` 的单个 plan。

**Plan 的 session 语义**：Plan 是 GUI 运行时对象，建模者在 YAML 中预置的是初始状态；用户在 GUI 中可继续添加、修改、删除方案，不会回写到 YAML 文件。

---

## 分层约束

1. **Model**：只能 `import` 其他 Model，严禁引用 Story。
2. **Story**：组合 Model 并配置场景，允许 `optimizer` 配置和 `patches`。
3. **循环检测**：`LoaderEngine` 自动阻止循环导入。

---

## lm_score — Life Matters 健康时长核心指标

`lm_score` 是 Life Matters 框架的约定核心变量，表示**关键指标同时满足健康条件的累计时长**。它是普通的 `state` 变量 + 标准方程，建模者在 YAML 中完整写出，无任何引擎特殊处理。变量名 `lm_score` 是约定俗成，可自由覆盖或重命名。

### 两种积累语义

| 语义 | 描述 | 适用场景 |
|------|------|---------|
| **可恢复**（cumulative） | 条件满足期间累加，不满足期间暂停；恢复后继续累计 | 慢性病管理、低血糖可扛过、轻度症状 |
| **不可逆**（latch） | 条件一旦不满足，`lm_alive` 标志永久归零，之后即使恢复也不再累计 | 器官衰竭、不可逆死亡事件 |

### YAML 写法

**可恢复模式**（推荐默认）：

```yaml
variables:
  lm_score:
    type: state
    value: 0.0
    unit: day
    description: "健康时长：GFR 与血压同时在安全范围内的累计仿真天数"
    reference: "Life Matters Framework core metric"

equations:
  lm_score_update:
    step_unit: day    # lm_score 单位是 day，step_unit 必须声明为 day——
                      # 若 simulation.step_size 是 hour 而这里误写成 hour，
                      # step 会按小时累加，把 lm_score 放大 24 倍（实测过的真实事故）。
    dynamics:
      lm_score: "lm_score + step if (GFR >= 15 and SBP <= 160) else lm_score"
    description: "累加健康时长（可恢复）"
```

**不可逆模式**（latch，适合死亡/器官衰竭）：

```yaml
variables:
  lm_score:
    type: state
    value: 0.0
    unit: day
    description: "健康时长：首次崩溃前的累计天数（不可逆）"
    reference: "Life Matters Framework core metric"
  lm_alive:
    type: state
    value: 1.0
    description: "存活标志：0 = 不可逆崩溃，1 = 存活"

equations:
  lm_alive_check:
    condition: "not (GFR >= 15 and SBP <= 160)"
    dynamics:
      lm_alive: "0.0"                    # 一旦触发，永久为 0
    description: "检测崩溃并锁定存活标志"
  lm_score_update:
    step_unit: day    # 同上：必须与 lm_score 的 day 语义一致，不能照抄其他方程的 hour
    dynamics:
      lm_score: "lm_score + lm_alive * step"
    description: "累加健康时长（不可逆）"
```

### 作为优化目标

```yaml
optimizer:
  objectives:
    - variable: lm_score
      metric: final          # 仿真结束时的累计健康天数
      direction: maximize    # 最大化健康时长
```

### 多模型 Import 的合并

当多个子模型各自定义了 `lm_score`（条件不同），import 时后者会覆盖前者（遵循标准 import 覆盖规则）。若需 AND 合并多个子模型的条件，建模者在顶层模型中显式重写 `lm_score_update` 方程：

```yaml
# 顶层模型：显式合并 Model A（GFR 条件）和 Model B（SBP 条件）
equations:
  lm_score_update:
    step_unit: day
    dynamics:
      lm_score: "lm_score + step if (GFR >= 15 and SBP <= 160) else lm_score"
```

### 设计原则

- `lm_score` 是普通变量，完全透明，所有仿真步的值均可输出和查看
- 条件表达式使用与方程相同的 asteval 沙箱，可引用模型中任意变量
- 多个健康条件用 `and`/`or` 自由组合
- GUI 目标变量选择器中，`lm_score` 显示 ⭐ 标记以便识别，无其他特殊行为

---

## optimizer — 决策变量与调度优化

### Sim 与 Opt 的分离原则

`simulation:` 和 `optimizer:` 是相互独立的场景描述，但可以通过 GUI 相互转化：

| 字段/概念 | simulation | optimizer |
|----------|-----------|-----------|
| 时间范围 | `simulation.start_date`/`end_date` | `optimizer.start_date`/`end_date`（可选） |
| 步长 | `simulation.step_size`（必填） | `optimizer.step_size`（可选，缺省沿用 sim） |
| Monte Carlo | — | `optimizer.mc` |
| 固定输入 + 决策变量 | `simulation.plans[*].regimens`（可视化用） | `optimizer.startpoint.regimens`（统一列表） |

**Fallback**：`optimizer.*` 字段缺省时，引擎从对应 `simulation.*` 继承；GUI 明确标注来源（"来自 sim" vs "已覆盖"）。

**GUI 转化**：
- "← 从 Sim 导入"：将 Sim tab 当前 inputEvents 复制为 `optimizer.startpoint.regimens` 决策变量，自动推算 bounds
- "发送到 Sim"：将 Pareto 推荐解的 regimen 预填为 Sim inputEvents

### optimizer.startpoint.regimens — 决策变量与固定背景量统一列表

`optimizer.startpoint.regimens` 是决策变量和固定背景量的统一列表（ADR 0109）。有 `optimize:` 块的条目是决策变量；无 `optimize:` 块的是固定背景量。`startpoint` 块描述优化器从哪个初始协议出发搜索。

```yaml
optimizer:
  startpoint:
    regimens:
      - variable: metformin_dose      # 固定背景量（无 optimize 块）
        time_start: "08:00"
        value: 500
        days: [Mon, Tue, Wed, Thu, Fri, Sat, Sun]
        label: "二甲双胍基础用药（背景）"
```

**独立性**：优化器每次评估在自己的内部仿真里运行，只使用 `optimizer.startpoint.regimens` 解码出的事件，不读取
`simulation.plans[*].regimens`，两条路径互不影响（见 `life-matters-reference-engine` 仓库 `optimizer_eval.py`）。`optimizer.startpoint.regimens`
必须显式定义；无隐式 fallback（ADR 0109 移除 fallback 链）。

### 评估时间窗（start_date / end_date / step_size）

优化器在每次评估时内部运行一次仿真，其时间范围和步长可独立于 GUI 的可视化设置：

| 字段 | 类型 | 说明 |
|------|------|------|
| `start_date` | `"YYYY-MM-DD"` | 优化评估起始日；缺省沿用 `simulation.start_date` |
| `end_date` | `"YYYY-MM-DD"` | 优化评估结束日；缺省沿用 `simulation.end_date` |
| `step_size` | `{value, unit}` | 评估步长；缺省沿用 `simulation.step_size` |

**设计原则：**
- 三者均为可选；不声明则从 simulation 继承。
- 显式声明可保证结果可复现：发布带 `optimizer.results` 的 YAML 时，读者可用相同时间窗重跑优化。
- 评估步长建议与 `simulation.step_size` 一致；若模型动力学时间尺度允许，可适当粗化以加速搜索。
- GUI 的时间控件值（工具栏上的日期和步长）在运行优化时作为 `optimizer_override` 传入引擎，优先级高于 YAML 静态值。

**典型用法（缩短评估窗以加速搜索）：**

```yaml
simulation:
  start_date: "2026-01-01"
  end_date:   "2030-12-31"   # 5 年可视化

optimizer:
  start_date: "2026-01-01"
  end_date:   "2027-12-31"   # 仅用 2 年评估，加速搜索
  step_size:
    value: 1
    unit: day
```

---

优化器将干预方案的参数化搜索分为四个粒度层（Tier），按科学价值与计算复杂度排序：

| Tier | 优化对象 | 变量类型 | 典型场景 |
|------|---------|---------|---------|
| T1 | 事件值（剂量/强度） | 连续实数，可选 `value_step` 离散化 | 药物剂量、营养摄入量 |
| T2 | 事件时刻（在时间窗内） | 离散整数（时间槽索引） | 进食窗口、给药时机、昼夜节律 |
| T3 | 星期组合（从候选日自由组合） | 离散整数（组合索引） | 运动频率、断食日安排 |
| T4 | 干预起始日（在日期窗内） | 整数（天偏移） | 治疗时机、季节性干预 |

每个 `inputs` 条目可独立启用任意 Tier 组合；x 向量是所有已启用维度按顺序拼接的结果。

T1 的 `optimize.value` 默认在 `[lo, hi]` 连续区间内搜索，解会带任意小数精度；声明可选的 `optimize.value_step` 后，引擎在解码阶段把连续解 snap 到以 `lo` 为起点、以 `value_step` 为间隔的网格点上，适合按临床或工程可读精度取值的场景，例如喂养量按 5 mL 一档、代谢当量按 0.1 MET-h 一档；不声明时行为不变，仍是连续搜索。

### T2：时间窗优化

T2 基于 `time_start`/`time_end` 统一区间字段（上一节）。`optimize.time_start`
搜索区间起点；区间宽度（`time_end - time_start`）默认固定不变（**1 维**，最常见情形）。
若额外声明 `optimize.time_end`，区间终点也独立搜索（**2 维**）。

**1 维：起点搜索，宽度固定**（"几点触发"这种宽度=0 的窄窗、"几点开始，持续时长不变"这种
宽窗，均属此类）：

```yaml
regimens:
  - variable: meal_carbs
    time_start: "08:00"
    time_end: "08:00"            # 宽度 = 0（单 step 窗口），搜索后宽度仍为 0
    days: [Mon, Tue, Wed, Thu, Fri, Sat, Sun]   # 固定星期（T3 未激活）
    label: "早餐碳水"
    optimize:
      value: [30, 80]
      time_start: ["07:00", "09:00"]   # 起点搜索窗 [lo, hi]
      time_step: "1h"                  # 可选；缺省 1h；精细场景可设 15min
```

```yaml
regimens:
  - variable: care_intensity
    time_start: "08:00"
    time_end: "20:00"             # sustained，宽度 = 12h
    label: "白天救治强度"
    optimize:
      value: [0.0, 288.0]
      time_start: ["06:00", "10:00"]   # 起点在 [06:00,10:00] 内搜索，宽度仍为 12h
```

**2 维：起点、终点独立搜索**（区间宽度本身也是决策变量）：

```yaml
regimens:
  - variable: care_intensity
    time_start: "08:00"
    time_end: "20:00"
    label: "白天救治强度（起止均搜索）"
    optimize:
      value: [0.0, 288.0]
      time_start: ["06:00", "10:00"]   # 起点搜索窗
      time_end: ["18:00", "22:00"]     # 终点搜索窗（独立于起点）
```

- `optimize.time_start` / `optimize.time_end` 格式均为 `["HH:MM", "HH:MM"]`（24 小时制，起止含边界）。
- `time_step` 合法值：`"1h"`（缺省）、`"15min"`，对两个窗口同时生效。引擎展开为离散时间槽，
  例如 `["07:00","09:00"]` + `1h` → `["07:00","08:00","09:00"]`（3 个槽）。
- 仅写 `optimize.time_start`（不写 `optimize.time_end`）时为 1 维：搜索后的 `time_end` =
  搜索后的 `time_start` + 固定宽度（= 该条目自身 `time_end - time_start`，单 step 窗口时宽度为 0）。
- 同时写 `optimize.time_start` 和 `optimize.time_end` 时为 2 维：两端独立搜索，互不联动。
- 旧字段 `optimize.time` 已废弃，不再受支持。请使用 `optimize.time_start`。
- 科学意义：时间生物学（Chrono-nutrition / Chronopharmacology）中，干预时机本身是关键决策变量，本框架将其显式纳入优化搜索空间。

### T3：星期组合搜索

```yaml
regimens:
  - variable: exercise_load
    time_start: "17:00"
    label: "运动"
    optimize:
      value: [30, 90]
      days_pool: [Mon, Tue, Wed, Thu, Fri, Sat]  # 候选日集合
      days_n: [3, 5]                             # 从 pool 中选 3~5 天
```

- `days_pool`：候选日集合（Mon–Sun 三字母缩写）。
- `days_n: [min, max]`：后台从 pool 中枚举所有满足 min ≤ n ≤ max 的合法组合，编码为整数决策变量。
- T3 激活时，顶层 `days:` 字段不写（无固定星期）。

### T4：干预日期范围优化

```yaml
regimens:
  - variable: caloric_restriction
    time_start: "08:00"
    days: [Mon, Tue, Wed, Thu, Fri, Sat, Sun]
    label: "热量限制"
    optimize:
      value: [400, 800]
      date_range:                                # 强制两组，均必填
        - ["2026-05-01", "2026-05-30"]           # 起始日搜索窗 [lo, hi]
        - ["2026-12-31", "2026-12-31"]           # 结束日搜索窗（固定时写同一日期两次）
```

- `optimize.date_range` 必须恰好两组：第一组为起始日搜索窗，第二组为结束日搜索窗。
- 若结束日固定，写 `["YYYY-MM-DD", "YYYY-MM-DD"]`（两值相同）。
- T4 激活时，顶层 `date_range:` 字段不写（固定日期范围）。
- 典型场景：治疗介入时机、季节性干预窗口、灾后救援资源投放时机。

> `mode: sustained` + `time_range` 这一写法（ADR 0098）已被 ADR 0100 取代，引擎不再支持，旧 YAML 文件需改用下方 `time_start`/`time_end` 统一区间字段。

#### value 语义：每个匹配日独立满额，按 N_steps 自适应分摊（ADR 0131，取代 0099）

`value`（以及 `optimize.value` 的上下界）表示**单次命中窗口（一个匹配日）内的总量**——
与 pulse 的"一次性总量"是同一量纲，也是**每个匹配日都独立、完整地交付一次**，不因为
`date_range`/`days` 让这个条目多匹配了几天而被稀释，也不因为少匹配几天而被加浓。这是为了
同时满足两条不变量：`step_size` 只影响精度（Euler 离散积分章节的原则），以及"匹配了多少天"
只影响交付的**总次数**，不影响**每次交付多少**——后者正是"日速率"这个建模意图本身的定义。

引擎在装载阶段为每个 sustained 条目预计算：

```
N_steps = 单次命中窗口的时长 / step_size
```

运行时每个命中 step 写入 `value / N_steps`（仍遵循"`type: input` 不乘 `step`"规则，dynamics 方程不需要改动）。
每个匹配日的累计贡献 = `(value / N_steps) × N_steps = value`，与 `step_size` 无关；仿真总时长、
`date_range` 覆盖天数、`days` 星期过滤只决定**这个条目在多少天里各自独立交付一次 `value`**，
不改变 `value` 本身的含义。窗宽=0（单 step 窗口，过去称"pulse"）是 `N_steps=1` 的特例，跟任何
其他窗宽共用同一条规则——不是两种不同的机制。

`单次命中窗口的时长` 只看这个条目自己的 `time_start`/`time_end`（缺省=全天 24h），**不看**
`date_range` 覆盖了多少天、`days` 过滤剩多少个星期几——这两者只是"要不要在这一天触发"的
过滤器，正如它们对 pulse 事件从来只是过滤器、从不参与 pulse 的取值一样。

> 建模时按"这个条目每次命中（每天）投入多少"来填 `value`（例如"训练负荷 50/天"就写
> `value: 50`），不需要手算"这个方案总共跑多少天"再乘进去；只有当这个 `value` 真正表达的是
> "不管实际匹配几天、总量都锁定为这个数"这种预算类语义时，才需要在 YAML 之外自己控制
> `date_range`/仿真总时长不再变化（此时 `value` 起到的是"人工设定总预算并平均摊薄"的作用，
> 引擎不做区分，是否需要这种效果、以及匹配天数一旦变化总预算要不要连带调整，都由建模者
> 自行判断——ADR 0099/0126 里"总量恒定"的旧语义已被本 ADR 取代，不再是引擎默认行为，
> 需要该效果只能靠建模者自己不去改变匹配天数）。

**示例：同一个建模意图，"日速率固定"（现在的默认、唯一行为）**

```yaml
regimens:
  - variable: training_load
    time_start: "00:00"
    time_end: "24:00"
    value: 50.0            # 每天 50 单位，不管这个 plan 实际跑几天
    date_range: ["2026-01-01", "2026-02-01"]  # 32 天：总交付 = 50 × 32 = 1600
    label: "constant daily training load"
```

把 `date_range` 缩短成 16 天（其余不变），总交付变成 `50 × 16 = 800`——**每天仍是 50**，
只是天数变了、总量跟着按比例变化，这正是"日速率"应有的行为。（对照：ADR 0099 时代的旧
行为会反过来锁死总量、让每天的有效速率随天数缩短而翻倍到 100——已被本 ADR 取代。）

#### `delivery: total | level`——总量摊分 vs 恒定水平（ADR 0132）

上一节的"每日独立满额"解决的是"总量应该随匹配天数正比变化"这一件事，但没解决另一件
不同的事：**有些 `type: input` 变量的物理意义根本不是"这段时间投入了多少"，而是"当前
处于什么状态/设定"**——如"当晚睡眠时长"、"就寝时机"、"救治强度"。这类变量在方程里从不
被累加，而是被当瞬时读数直接用（跟基线比较、当乘法因子），本该在整个生效窗口内**保持
不变**。ADR 0099/0131 的"总量÷N_steps"摊分规则对这类变量是错的：读数会随 `step_size`
（乃至匹配天数）反比例漂移，模型只要换一个跑法（变步长做收敛性检验、或被 import 到
`step_size` 不同的另一个模型里）就会悄悄失真——这个问题从 ADR 0098（sustained 最初提出，
动机正是"持续救治强度"这类应保持恒定的变量）到 ADR 0099 重新定义 `value` 语义那天起就
存在，只是从未在变步长场景下暴露过。

**判断标准**：这个变量在下游方程里是被"累加"（贡献总量随时间推移增长）还是被"直接读取"
（跟某个基线比较、当系数使用，同一时刻的读数不该因为换了 step_size 就不一样）？

| `delivery` | 语义 | 每个命中 step 的交付量 | 判断依据 |
|---|---|---|---|
| `total`（默认，缺省不写） | 每个匹配日的窗口总量，按上一节规则摊分 | `value / N_steps` | 训练负荷、进食总量、给药总量——量本身是"这段时间投入/摄入了多少"，会被下游累加 |
| `level` | 恒定水平，不摊分 | `value` 本身 | 睡眠时长、就寝时机、饮食质量得分、救治强度、防护水平——量本身是"当前的设定/状态"，被下游当瞬时读数比较或相乘 |

```yaml
regimens:
  - variable: sleep_hours
    time_start: "00:00"
    time_end: "24:00"
    value: 6.0          # 直接就是目标水平，不需要手算乘 N_steps
    delivery: level       # 每个命中 step 直接交付 6.0，不随 step_size/匹配天数变化
    days: [Mon, Tue, Wed, Thu, Fri]
    label: "工作日睡眠时长"
```

`delivery: level` 对单 step 窗口（pulse）是 no-op——pulse 本来就是 `N_steps=1` 的特例，
除或不除结果相同。`days`/`date_range`/`time_start`/`time_end` 在两种 `delivery` 下语义
完全一致，只决定"这一天要不要触发"，不影响交付量。

**不是靠变量名约定实现**（如给变量名加内部后缀让引擎特殊处理）——那条路已经在讨论
`consumed_by` 白名单方案时明确否决过（"变量名自由、引擎不认保留名"的既有原则）。
`delivery` 是显式写在 regimen 条目上的声明，跟 `time_start`/`time_end`/`days` 是同一
层级的属性，不是命名约定，也不是重新引入 ADR 0100 去掉的 pulse/sustained 开关——那个
开关是真冗余（窗宽一个数就决定"点 vs 窗"，去掉不丢信息），`total`/`level` 是一个新的、
独立的维度：Banister 的 `training_load` 和这里的 `sleep_hours` 用的是完全相同的窗宽
（全天），仅靠窗宽无法区分二者，必须显式声明。

#### 何时用固定窗宽的 sustained，何时用脉冲触发+衰减态（ADR 0126/0127/0131）

`N_steps` 是引擎在**装载阶段**（仿真开始前）预计算的，不是运行时动态确定的——这意味着
条目自己的 `[time_start, time_end)` 窗宽必须在写 YAML 时就是一个确定数字（ADR 0131 起，
`date_range`/`days` 覆盖多少天不再参与这个计算，只是命中过滤器，因此不再需要"总时长"是
确定数字，只需要"这一次窗口多宽"是确定数字）。据此判断：

- **这个输入自己的 `[time_start, time_end)` 窗宽，在装载阶段是否已经确定？**
  - 是（不依赖优化器搜索结果就能算出准确窗宽，例如固定写死的 `"08:00"~"20:00"`，或
    `days`/`date_range` 这类只影响"哪几天触发"、不影响窗宽本身的过滤条件）→ 正常按
    ADR 0127 的默认规则或显式区间写，不需要额外机制。
  - 否（窗宽本身是 T2/T3/T4 搜索变量，比如"每天工作到几点"起止都待搜索，或者依赖运行时才能
    确定的状态）→ 固定窗宽的写法无法工作（`N_steps` 在装载时算不出来），改用**脉冲触发
    （单 step 窗口）→ 写入 `type: state` 衰减态变量 → 下游方程读衰减态**。这是纯局部的
    逐步递推机制（每步只需要"当前状态 + 当前输入"），不需要预先知道未来会跑多少步，因此
    不受"窗宽未知"的限制。

### 统一区间表示：time_start / time_end（ADR 0100/0127）

任何输入的生效窗口都是同一对 `[time_start, time_end)` 字段在数轴上的取值，不是几种互斥
的"模式"选一个，区别只在窗宽：

| `time_start` / `time_end` 关系 | 含义 |
|---|---|
| 两个都不写 | 全天 `["00:00","24:00")`（ADR 0127 默认规则，day-rate 输入） |
| `time_end == time_start` | 单 step 窗口，`N_steps=1`，`value` 原样写入该 step（过去称"pulse"） |
| `time_end != time_start`（不跨越全天） | 区间内每个 step 按 `value/N_steps`（上节方程） |
| `time_start="00:00"`, `time_end="24:00"` | `[0,24)` 全覆盖，是窗宽=全天时的取值，跟"两个都不写"的默认结果相同，不是单独状态 |

```yaml
regimens:
  - variable: care_intensity
    time_start: "08:00"
    time_end: "20:00"           # [08:00, 20:00) 区间，sustained
    date_range: ["1945-08-06", "1945-08-11"]   # 只影响哪几天触发，不影响下面的 value
    label: "白天救治强度"
    optimize:
      value: [0.0, 36.0]        # 每个匹配日的窗口总量；12h / step=1h → N_steps=12
```

#### 窗宽是同一范畴的量，衰减是独立于窗宽之外的下游关注点

不同窗宽的 input（单 step、全天、显式区间）是同一个东西（`value` = 生效窗口内的总量，按
`N_steps` 均分到每个命中 step）在窗口宽度上的不同取值，不是几种互斥的"input 类型"（ADR
0127）。所有窗宽共享同一套单位约定（裸单位、不含时间分母，见前面"input 变量的单位规范"），
`value` 的量纲不随宽度变化。

**衰减不属于"窗宽"这条轴的讨论范围**——衰减是下游 `type: state` 变量自己的 `dynamics`
方程要不要写"随时间自然回落"这一项，是一个独立的方程维度，跟驱动它的 input 窗宽多宽无关：
宽窗输入驱动的状态同样可能需要衰减（比如"持续输液速率"停止后，血药浓度仍按半衰期继续
回落）。

**窄窗（单 step 窗口）有一个特有的陷阱**（宽窗没有）：窄窗只在命中的那一个 step 写入非零值，
其余 step 该变量读到的是 0；如果某个下游方程**直接读这个变量**、而这个方程的评估频率比
窗口更密（例如 `step_size: hour` 但窗口每天只命中一次），这个方程在 23/24 的评估里都会读到
0——如果方程想表达的是"当前是否仍处于某种持续状态"，而不是"当天有没有发生这个一次性事件"，
就会被这个"其余时刻是 0"的假象带偏（2026-07-09 在 `hypertension_gout_sim.yaml`/
`smoking_stress_sim.yaml` 各发现一例，见 ADR 0126）。宽窗没有这个陷阱，因为它在整个生效
窗口内的每个 step 都是非零的——**这也是 ADR 0127 把"完全不写时间 = 全天"定为默认值的原因
之一**：day-rate 输入默认给宽窗，从源头上避免这个陷阱，只有真正的离散事件（进食、给药）
才需要窄窗，需要窄窗时建模者是明确写了 `time_start` 的，不是引擎替他选的。

**窄窗情形下的判断标准（看这个变量出现在方程的哪个结构位置，不是看有没有 `*step`）**：

容易走进的误区：以为"这一项有没有乘 `step`"是判据。不是——`step` 在"该方程 `step_unit` 与
`simulation.step_size` 相同"时恒等于 1（`step = step_size_sec / equation_step_sec`），乘不乘
`*step` 在这种情况下数值上完全等价，不产生任何摊薄或增强，`*step` 只是"这个量纲是否需要跨
step_unit 换算"的标记，跟"能不能读这个 input"无关。真正的判据是**这个 input 变量出现在方程的
哪个结构位置**：

- **允许**：出现在某个 `state` 自己 `dynamics` 里、**顶层加/减项**（不管这一项有没有再乘
  `*step`），即形如 `state: state + ... ± f(input变量) ...` 的自引用累加式：
  `body_weight: body_weight - caloric_deficit/7700`、
  `nicotine_plasma: nicotine_plasma + eta_abs*cigarettes_per_day - k_nic*nicotine_plasma*step`
  里的 `eta_abs*cigarettes_per_day` 项、`uric_acid: uric_acid + ... + max(0, psi_ketone
  *caloric_deficit/700 - 15.0)*step` 里的酮体尖峰项（这里虽然乘了 `*step`，但
  `step_unit`与`simulation.step_size`相同、`step`恒为1，跟不乘完全等价，仍然是合法的顶层
  加项）——这是"一次性事件触发时，把它的贡献累加进自己的持久总账"，state 自己记得累计结果，
  不需要每步重新读原始 input。
- **不允许**：出现在**"目标值/弛豫目标"子表达式内部**（`rate*(state - 目标值)*step` 里目标值
  那一坨），或出现在**没有 `+ 自身` 的纯代数快照方程**里——这两种结构表达的都是"当前是什么
  状态/系统正在收敛到哪里"，只能由 `state`/`parameter` 拼出来，因为这类计算隐含假设"这个量
  在相邻几步之间大致稳定"，而窄窗输入在 23/24 步是 0、其余步骤突然非零，会让目标值在"完全
  无效"和"满额生效"之间剧烈闪烁，破坏弛豫动力学的前提——不是"读到 0 不对"（读到 0 本身没
  问题，0 就是没触发时该有的值），而是"用一个会剧烈闪烁的量去扮演本该稳定的目标值"这个结构性
  错配。两个已知 bug（`bp_dynamics` 的弛豫目标、`health_economic_index` 的代数快照）都精确落在
  这一类；所有已知正确先例（`nicotine_plasma`/`thiazide_level`/`uric_acid` 的酮体项等）都是
  顶层自引用加项。**给宽窗输入(如 `health_economic_index_update` 需要的"今天抽了多少")，
  比给窄窗输入更不容易踩这个坑——这正是 ADR 0127 的"全天默认"和这里的结构判据互补的地方。**

> 上面这条"结构位置"判据目前只能靠建模者自己核对（尚未有引擎校验），已知会漏——已规划一个
> 替代方向（尚未实现）：给每个 `type: input` 变量声明
> `consumed_by: [方程名, ...]` 白名单，引擎校验该
> input 是否只被白名单内的方程引用——解决的是"读错物理量"这一类（如 `bp_dynamics` 该读
> `body_weight` 却读了 `caloric_deficit`，不管窗宽怎么调都修不好，只能靠白名单拦），跟
> ADR 0127（窗宽默认规则，解决"窗太窄、别的方程读到假 0"那一类）是互补的两个机制，不是
> 同一个方案的两个版本。

**旧字段已废弃**（旧 YAML 文件需手动更新，旧字段不再被引擎读取）：

| 旧字段（不再支持） | 等价的新写法 |
|---|---|
| `time: "HH:MM"`（pulse） | `time_start: "HH:MM"`（`time_end` 省略 = 默认同值 = pulse） |
| `mode: sustained` + `time_range: [a, b]` | `time_start: a, time_end: b` |
| `mode: sustained`（无 `time_range`） | `time_start: "00:00", time_end: "24:00"` |

直接写 `time_start`/`time_end`，不需要先判断"我要的是单点/区间/全天"——
三者是同一对字段在数轴上的位置关系，不是三个独立的开关/分支。`days`（星期几过滤）、
`date_range`（日历区间）字段不变，与 `time_start`/`time_end` 正交。

### x 向量编码规则

x 向量按 `optimizer.startpoint.regimens` 列表顺序展开，每个条目按 `[value?, time_start?, time_end?, days?, date_start?, date_end?]` 顺序贡献维度：

| 条目启用的 Tier | x 贡献维度 | 变量类型 |
|--------------|-----------|---------|
| T1 only | 1（value） | 连续实数 |
| T2 only，1 维（仅 `time_start`） | 1（time_start_idx） | 整数 |
| T2 only，2 维（`time_start`+`time_end`） | 2（time_start_idx, time_end_idx） | 整数×2 |
| T1 + T2（1 维） | 2（value, time_start_idx） | 实数 + 整数 |
| T1 + T2（2 维） | 3（value, time_start_idx, time_end_idx） | 实数 + 整数×2 |
| T1 + T3 | 2（value, combo_idx） | 实数 + 整数 |
| T1 + T4 | 2~3（value, date_start_offset[, date_end_offset]） | 实数 + 整数×1~2 |
| 固定背景量（无 optimize） | 0 | — |

混合整数向量由 NSGA-II 连续松弛处理；单目标算法（L-BFGS-B / Nelder-Mead）不支持整数变量，启用 T2/T3/T4 时自动切换为 NSGA-II 并给出警告。

**示例**：`meal_carbs`（T1 + T2 1维）和 `exercise_load`（T1 + T3）各贡献 2 维，x 长度为 4：

```
x = [carbs_value, time_start_idx, exercise_value, combo_idx]
    [   55.3,           1,            62.0,            2   ]
# time_start_idx=1 → slots[1] = "08:00"；time_end = "08:00" + 固定宽度
# combo_idx=2      → combinations(pool, n)[2] = [Mon, Wed, Fri]
```

`optimizer.results.recommended` 只存 `x`/`f` 原始向量，不存解码后的人类可读结果——解码是从 `x` + `optimizer.startpoint.regimens` 的结构纯算法推导，不需要额外持久化（见 §`optimizer.results` 一节）。

---

## optimizer.results — 优化结果内嵌格式

优化完成后，结果写回 `optimizer.results` 块，与配置并列存于同一 YAML 文件。
这意味着**发布模型即发布结果**；有结果的模型加载时，Opt 面板的"继续计算"复选框默认开启，用户可选择热启动（warm-start）或冷启动。

### 完整结构

```yaml
optimizer:
  method: nsga2
  objectives: [...]
  startpoint: {...}
  algorithm: {...}

  results:                          # ← 优化完成后由 GUI 写入，无需手动填写
    generated_at: "YYYY-MM-DD"     # 生成日期（ISO 8601 日期部分）
    method: nsga2                  # 使用的算法
    n_solutions: 8                 # Pareto 前沿解的数量
    elapsed_seconds: 87.3          # 本次运行耗时（秒）
    pareto_front:                  # 所有非支配解（flow-style，每行一个解）
      - {x: [0.30, 0.29, 0.30], f: [65.8, 47.1]}
      - {x: [0.35, 0.33, 0.34], f: [66.9, 44.8]}
    recommended:                    # 建模者从 Pareto 前沿中标注的推荐点（非唯一最优）
      x: [0.30, 0.29, 0.30]       # 决策变量值（与 optimizer.startpoint.regimens 决策条目顺序对应）
      f: [65.8, 47.1]             # 目标函数值（与 objectives 顺序对应）
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `generated_at` | 日期字符串 | 写入日期，用于判断结果是否过期 |
| `method` | string | 算法名（nsga2 / l-bfgs-b / nelder-mead） |
| `n_solutions` | int | Pareto 前沿解的数量 |
| `elapsed_seconds` | float | 本次运行耗时 |
| `pareto_front` | list | 所有非支配解，每个元素为 `{x: [...], f: [...]}` |
| `recommended.x` | list | 推荐点的决策变量值（建模者从 Pareto 前沿中选定，非唯一最优） |
| `recommended.f` | list | 推荐点的目标值 |

不再存储解码后的人类可读方案/目标字典——人类可读的展示和"发送到 Sim"功能都从 `x`/`f` 现场解码（前端 `xToInputEvents`），避免维护两份格式（曾有 `recommended.regimen`/`recommended.objectives` 字典，因从未被任何代码路径读取、保存逻辑也早已不再生成，于 2026-06-21 移除）。

**`x` 向量与 inputEvents 的映射关系**：x 向量按 `optimizer.startpoint.regimens` 中决策条目（有 `optimize:` 块）的顺序展开，每个条目按启用的 Tier 贡献维度：T1 贡献 1 维连续实数（value），T2/T3/T4 各贡献 1 维整数（时间槽索引 / 组合索引 / 天偏移）。此映射关系由 `optimizer.startpoint.regimens` 的结构隐含，不需要额外存储；前端 `xToInputEvents` 函数按相同顺序解析（见 `sim_design.md`）。

### 设计原则

- **`results` 整体覆写**：每次保存时用新前沿完整替换旧 `results`，不保留历史；Pareto 前沿只会随搜索改善或持平，不会退化。
- **格式统一**：`pareto_front` 使用 YAML flow-style（`{x: [...], f: [...]}` 单行），50 个解 = 50 行，不破坏模型可读性。
- **热/冷启动（用户选择）**：Opt 控制栏的"继续计算"复选框始终可见；有已有结果时可勾选（热启动），无结果时 disabled（冷启动）。勾选热启动后若修改了目标函数、约束或决策变量搜索范围，复选框变为橙色"⚠ 继续计算"提示匹配度可能下降，但不强制切换为冷启动。
- **Sim 读取 opt 结果**：加载含 `recommended.x` 的模型时，Sim 面板询问是否将推荐点预填为当前 inputEvents；用户可选择加载或忽略。
- **Opt→Sim 多输出（N-N）**：Pareto 前沿是 N 组输入组合；软件将 N 个 Pareto 解各自重组为合规的 Sim inputEvents（Plan），供 F-MPLAN 并行仿真和比较；opt.results 仅保留原始 x/f 向量。
- **`recommended` 不代表唯一最优**：多目标优化没有单一"最优解"，`recommended` 是建模者标注的平衡点，用户应结合 `pareto_front` 自行权衡选择。命名避开 `reference`，是为了不与 `variables.<name>.reference`/`equations.<name>.reference`（文献引用字段）混淆。
- **发布即结果**：建模者运行优化、保存模型、上传 YAML，接收者打开即看到 Pareto 前沿和推荐点；`results` 可独立阅读。
- **无结果也合法**：`optimizer.results` 是可选块；没有该字段的模型正常运行，从随机初始种群开始搜索。

### 工作流

```
建模者                          GUI                          模型文件
  │                              │                              │
  │── 打开含 results 的模型 ──>  │ 显示历史 Pareto 前沿          │
  │                              │ 工具栏：● 模型含有历史结果     │
  │── 点击"运行优化" ──────────> │ warm-start（历史解为初始种群）  │
  │                              │ 继续进化 n 代                │
  │── 点击"保存结果到模型" ────> │ POST /api/optimizer/write-results
  │                              │──────────────────────────>  │ optimizer.results 覆写
  │── 点击"下载模型" ──────────> │ GET /api/file-raw/{path}     │
  │   接收 .yaml 文件             │                              │
```
