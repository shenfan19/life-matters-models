# 变量、方程与证据类型

## 变量类型（3 种）+ evidence_type 原地换算

`variables:` 块下的 `type` 只接受 3 个值：

| 类型 | 引擎读取 | 建模者填入 | 用途 | 优化归属 |
|------|---------|----------|------|---------|
| `state` | `value`（随时间更新） | 初始值 | 随时间演化的状态变量 | — |
| `input` | `value`（用户可调） | 控制量 | 用户干预量（行为、剂量） | **外环 opt（Simulator）** |
| `parameter` | `value`（不变；MC 模式每 run 采样一次） | 动力学系数或分布表达式 | 直接进方程的机制系数（PK速率、方程斜率、Bergman p1/p2/p3 等）；值由内环 opt 对文献数据拟合后确定。`value` 可写为 `normal(μ, σ)` 等分布形式，表示个体间差异；确定性模式取均值，MC 模式每 run 采样一次。 | **内环 opt（Modeller，待实现）** |

> **`probability_constant` 已退役**：发病率、病死率等概率值统一用 `evidence_type: ir` 表示。现有 YAML 中的 `probability_params:` 节仍可解析，Loader 会自动映射。

**`evidence_type` 不是第 4 种 `type`，而是 `variables:` 条目上的一个可选字段**（原为独立的顶层
`evidence:` 节，后并入 `variables:`——evidence 本质上仍是变量，独立成节反而制造了两套平行命名
空间，制造混乱，见 ADR 0137，取代 ADR 0040 的顶层节设计）：
文献直接给出的效应量（OR/HR/RR/Cohen's d 等）声明为某个 `variables:` 条目的 `evidence_type`
字段，`value` 填原始文献数值，`type` 必须是 `parameter`，Loader 在加载阶段原地把 `value` 换算
为可进方程的系数（同名，不加后缀），`equations`/`dynamics` 直接写该名字即可——不需要额外记一个
`_effective` 之类的衍生名字。
换算后的变量额外携带两个溯源字段（不在 YAML 里声明，由 Loader 自动填入，仅供查询/调试用）：

| 字段 | 含义 |
|------|------|
| `evidence_type` | 原始效应量的类型（如 `rr`/`or`/`hr`），非 evidence 来源的 parameter 为 `None` |
| `evidence_raw_value` | 换算前的原始文献数值（如 OR=1.65），与换算后的 `value` 分开保留，用于审查/溯源 |

**没有单独建 `VariableType.evidence`**：换算结果合并进 `parameter` 类型，靠上面两个字段做标记，
不引入第 4 种类型——原因是内环优化器（Modeller）尚未实现，目前没有任何代码路径会对 `parameter`
做自动调参，无需用类型隔离防止误优化；等 Modeller 实现时，只需让它跳过 `evidence_type is not None`
的 parameter 即可，不必现在为一个还不存在的优化器预留类型膨胀。这也是 evidence_type 没有和
`type`（state/input/parameter，变量在动力学里的角色）合并成一个复合字符串字段（如
`type: evidence-rr`）的原因：角色和"数据来源的统计效应量类型"是两个正交的维度，合并会让
`type` 枚举从 3 个膨胀到 11 个，且把两件事绑进同一个字段，后续按角色做判断（比如找所有
parameter）要从等值匹配退化成前缀匹配。声明了 `evidence_type` 的条目，`type` 必须是
`parameter`，否则 Loader 报错拒绝。

**`parameter` vs `evidence_type` 的判断准则：**
- 文献给你一个直接可进方程的数（但来自数学拟合而非直接测量，如 Bergman 模型系数）→ 普通 `parameter`（不声明 `evidence_type`，交由 Modeller 内环优化校准）
- 文献给你原始统计效应量（OR=1.65、HR=0.82、d=0.68、ke=0.198 h⁻¹）→ `parameter` + `evidence_type`（Loader 自动换算）

**当前能力边界**：gui 暂无 `evidence_type` 字段的专属可视化/编辑表单，建模者需直接编辑 YAML；
GUI 的 variables/equations 通用编辑器尚未实现，evidence 表单待该编辑器实现后一并补齐。

**自动接入 dynamics（`applies_to`，可选）**：换算出系数之后，把它接到某个状态变量的动力学
方程上，默认仍由建模者手写。8 种子类型里 `ir`/`ard`/`hr`/`rr`/`or` 这 5 种的"接入方式"只有
一种没有歧义的写法（都是"以换算后的系数为速率，累加进某个目标状态"），声明以下字段后
Loader 会自动生成对应 dynamics：

| 字段 | 适用子类型 | 含义 |
|------|----------|------|
| `applies_to` | ir/ard/hr/rr/or | 目标状态变量名（必须已在 `variables:` 声明），触发自动生成 |
| `step_unit` | ir/ard/hr/rr/or | 生成的 dynamics 所用的步长单位，必须是 `minute`/`hour`/`day`（与 `equations.step_unit` 同一约束） |
| `rate_unit` | ir/ard 自身声明；hr/rr/or 从 `baseline_ref` 指向的 ir/ard 条目读取 | 速率的自然时间单位（`minute`/`hour`/`day`/`week`/`month`/`year` 之一），与 `step_unit` 的比值算成系数写进生成的表达式，不依赖 `Equation.step_unit` 表达年/周/月 |
| `baseline_ref` | hr（已有）、rr/or（新增） | 必须指向同一文件内一个 `evidence_type: ir/ard` 的 `variables:` 条目；rr/or 的换算结果本身只是比例，需要这个基线才能生成"基线×比例"的速率 |

```yaml
variables:
  baseline_cvd_ir:
    type: parameter
    evidence_type: ir
    value: 0.012
    unit: prob/year
    rate_unit: year              # 速率的自然时间单位
    applies_to: cvd_risk_baseline # 自动生成：cvd_risk_baseline += baseline_cvd_ir * (day/year) * step
    step_unit: day

  smoking_cvd_rr:
    type: parameter
    evidence_type: rr
    value: 2.5
    baseline_ref: baseline_cvd_ir  # rr 本身只是比例，需要基线才能生成速率
    applies_to: cvd_risk_smoker
    step_unit: day
```

**约束**（确保不引入隐式科学假设，做不到就报错而非静默忽略）：
- `cohens_d`/`beta`/`pk` 声明 `applies_to`会直接报错——这 3 种的接入方式本身是建模判断
  （过渡形式、回归结构、PK 模型结构不唯一），永远不支持自动生成，必须手写 dynamics。
- 同一个 `applies_to` 目标被两条以上 evidence 同时声明会报错——多个风险因子怎么组合
  （相乘=比例风险假设，还是相加=竞争风险模型）本身是有争议的流行病学方法论问题，引擎不
  代为选择，请去掉 `applies_to` 手写 dynamics。
- `applies_to` 只在声明了 `evidence_type` 的条目上有意义；未声明 `evidence_type` 却写了
  `applies_to` 会报错，避免普通 parameter 误用这个字段。
- 不声明 `applies_to` 时行为完全不受影响，继续手写 dynamics——这是纯增量字段。

设计推导见 ADR 0040「实施记录」（顶层 `evidence:` 节的原始设计）与 ADR 0137（并入 `variables:` 的后续决策）。

**`input` 变量的单位规范（事件量，唯一规则）：**

LM 引擎以 sustained 模式执行 input 变量（ADR 0127）：命中生效窗口的 step 写入 `value/N_steps`，窗口外自动为 0，累计贡献恒等于 `value`。`unit` 字段描述**窗口内交付的物理总量**，始终使用裸单位，不含时间分母。`input` 方程直接加减，**不乘 `step`**：

| 正确写法 | 禁止写法 | 原因 |
|---------|---------|------|
| `mg`、`g`、`kcal`、`kg`、`MET-h`、`sessions` | ~~`mg/day`、`g/kg/day`、`kcal/day`、`kg/week`~~ | schedule 触发频率由 `days` 控制，`/day` 与 T3/T4 优化器不兼容 |

文献给出的速率参考值（如"500 mg/day"、"2 mg/kg/day"）记录在 `reference` 或 `description` 字段，不进入 `unit`。真正的持续速率过程（如静脉输注速率）建模为 `parameter`（带 `1/day`、`1/min` 单位），配合 `× step` 在方程中积分；`input` 变量的方程不使用 `× step`。

> 示例：`aspirin_dose = 50 mg`（晨服，文献"100 mg/day"记入 `reference`，`unit` 只写 `mg`）；
> `caloric_deficit = 500 kcal`（每日触发，不写 `kcal/day`）；
> `weight_loss_weekly = 0.5 kg`（每周一触发，不写 `kg/week`）。

**`evidence_type` 的 8 种取值**：`rr`（相对风险）、`or`（比值比，需 `baseline_prevalence`）、`hr`（风险比，需 `baseline_ref`）、`ard`（绝对风险差）、`cohens_d`（效应量，需 `population_sd`）、`ir`（发病率/死亡率）、`beta`（回归系数）、`pk`（PK/PD 参数）。

**换算方程的权威版本不在本文件**，在 `life-matters-reference-engine` 仓库 `docs/evidence/conversion.md`
（含 `evidence_type`/`evidence_raw_value` 溯源字段说明和已知实现细节，如 `hr` 的 `baseline_ref`
在基础换算路径上不校验目标类型）；`applies_to` 自动接入 dynamics 的校验顺序与生成表达式模板见同目录
`applies_to.md`。本文件只维护"建模者要填哪些 YAML 字段"，方程随 Loader 实现变化，避免两处维护、
两处漂移，发现两边不一致以 `conversion.md`/`applies_to.md`（对应实际 loader.py 代码）为准。

---

## 医学证据类型与变量映射

声明了 `evidence_type` 的变量由 Loader 在加载阶段自动换算，Simulator 只见换算后的值（与声明时
同名，无后缀）。8 种子类型的完整换算逻辑见 `life-matters-reference-engine` 仓库 `docs/evidence/conversion.md`，
YAML 示例见下方 Schema，决策背景见 `decisions/0040`（顶层节设计）与 `decisions/0137`（并入 `variables:`）。

患病率（Prevalence）直接设为对应 `state` 变量的初始 `value`，不需要单独声明 `evidence_type`。

---

## 变量与方程数据规范

所有 YAML 中的 `variables` 和 `equations` 条目须遵守：

1. **强制 `description`**：简洁说明该变量/方程的物理或医学意义。
2. **强制 `reference`**：所有数值（`value`）、范围（`bounds`）和动力学方程（`dynamics`）必须标注数据来源。格式不限，但须包含足够信息（DOI、PMID、简写引用或 URL）让读者在 30 秒内定位原始文献。暂无来源时填 `["TODO:SOURCE"]` 并在 `description` 中注明估算逻辑。
3. **可选 `locator`**：与 `reference` 配对，标注该数值在文献内部的精确位置（页码、图、表、方程、章节），格式不限但应具体，例如 `"Table 1"`、`"Figure 3"`、`"eq.3"`、`"p.1172"`、`"§6.2"`。能写出具体位置时优先写 `locator`，而不是把位置信息散落在 `description` 文字里；尚未核实具体位置时填 `"TODO:LOCATE"`，不要臆造页码/图表号。GUI 在 reference 列展示时会自动拼接为 `reference (locator)`。
4. **可选 `comments`**：记录多文献冲突时的选择理由或参数微调过程，不替代 `description` 和 `reference`。

---
## 时间与步长

步长分为两个独立概念，分别在不同字段声明（ADR 0104、ADR 0105）：

### equation.step_unit（当 dynamics 使用 step 时必填）

**仅当方程的 `dynamics` 表达式中使用了 `step` 时，`step_unit` 才是必填字段**，用于声明该方程中 `step` 符号所代表的时间单位。不含 `step` 的 `dynamics:` 方程（如纯代数赋值）无需声明 `step_unit`。

```yaml
equations:
  bp_dynamics:
    description: "..."
    step_unit: day        # minute | hour | day（dynamics 使用 step 时必填）
    dynamics:
      systolic_bp: "systolic_bp + (...) * step"

  performance_calc:
    description: "静态计算，无 step，无需 step_unit"
    dynamics:
      performance: p0 + fitness - fatigue
```

`step_unit` 是方程的属性，反映系数标定时假设的时间分辨率。跨模块 import 时，每条方程携带自己的 `step_unit`，引擎据此正确换算 `step` 的数值。

### simulation.step_size（必填）

仿真执行步长，独立于方程的 `step_unit`：

```yaml
simulation:
  step_size:
    value: 1
    unit: day             # minute | hour | day
```

`optimizer.step_size` 同格式，可选（缺省沿用 `simulation.step_size`）。

### 方程内符号

| 符号 | 含义 | 说明 |
|------|------|------|
| `step` | 当前方程的步长（单位 = `equation.step_unit`） | **唯一规范符号** |
| `t` / `time` | 当前仿真时间（单位 = `equation.step_unit`） | |
| ~~`step_size`~~ / ~~`dt`~~ | 同 `step` | **废弃**，禁止在新方程中使用；validator 检测到即报错 |

**`step` 的计算方式**：`step = simulation.step_size / equation.step_unit`（换算为相同时间单位后相除）。

| 示例 | `simulation.step_size` | `equation.step_unit` | 方程内 `step` 值 |
|------|----------------------|---------------------|----------------|
| 同单位 | 1 day | day | 1 |
| 粗步长 × 细单位 | 1 day | hour | 24（每步积分 24 个小时单位） |
| 细步长 × 粗单位 | 1 hour | day | 1/24（每步仅积分 1/24 天） |

---

## 方程步长规则

**根据变量类型决定是否乘 `step`：**

| 变量类型 | 方程类型 | 是否乘 step | 原因 |
|---------|---------|-----------|------|
| `state` | 速率（连续动力学） | **必须乘** | 效果与时间成比例 |
| `input` | 脉冲（pulse 驱动） | **不乘** | 一次性量，与步长无关 |
| `parameter` | 乘数系数 | 不适用 | 本身是系数 |

```yaml
# ✅ 速率类：state 更新必须乘 step
dynamics:
  insight:   insight + 0.069 * cognitive_efficiency * step
  nutrition: max(0, nutrition - 0.010 * step)

# ✅ 瞬时类：input 脉冲不乘 step
dynamics:
  stomach_carbs: stomach_carbs + carb_intake
```

### 跨 step_unit 的参数换算：线性除法 vs 开根（ADR 0121）

文献给的速率参数常以"天"为单位（如"半衰期 4-5 天"、"每年下降 1.5 mL/min"），但所在方程的 `step_unit` 可能是更细的 `hour`。**`step_unit` 机制只换算方程自身的 `step` 符号，不会对嵌入 dynamics 表达式里的字面参数值做单位换算**——这一步换算的责任在建模者，写错了引擎不会报错，只会悄悄把效果放大/缩小若干倍（典型是 day→hour 放大24倍）。

换算前先判断该参数所在的动力学项属于哪一类，**两类换算方式不同**：

| 类型 | 项的形式 | 换算方式 |
|------|---------|---------|
| **A：状态无关通量项** | `X: X + rate * f(其他变量) * step`（系数不依赖被更新的同一个状态变量） | **精确线性**：天速率 ÷ 24 = 时速率 |
| **B：自指数衰减/恢复项** | `X: X - k * (X - target) * step`（系数乘以"状态自身与目标值的差"，target 可以是0） | **开根**：$k_{hour}=1-(1-k_{day})^{1/24}$，线性除以24只是 $k_{day}$ 较小时的近似（如 $k_{day}=0.15$ 时偏差约8%） |

类型 B 是一阶线性 ODE $dX/dt=-k(X-\text{target})$ 的 Euler 离散形式——24个 hour 步的复合效应是乘法性的（$(1-k_{hour})^{24}$），不是线性叠加，所以不能直接除以24。两种算法的计算成本完全相同（一次幂运算或一次除法，模型加载时算一次），**类型 B 必须用开根方程，不接受线性近似**。

**不要简单把整条方程的 `step_unit` 改掉来"修复"类型 B 的参数**——`step_unit` 是整条方程的属性，如果同一条方程里混有其他已经按当前 `step_unit` 正确标定的项（常见情况），改 `step_unit` 会把那些项也错误地稀释。正确做法是只修改该参数自身的 `value`（连同 `unit`、`description` 一起更新留痕），保持方程的 `step_unit` 不变。详见 ADR 0121。

---

## 方程执行顺序（priority）

每个 step 内，方程按 `priority` **降序**排序后依次执行（数值越大越先执行；未声明默认为 0）。
排序是**全局一次性**的：按方程整体的 `priority` 排序，
单条方程内部按 `condition` → `dynamics` 的固定顺序求值。

**同 step 内顺序写入语义（非"快照"）**：每条方程算出的新值会立即写回模型变量
（含 `bounds` 裁剪），随即对**本 step 内后续执行的方程**可见。
即：`priority` 数值更大的方程先执行，其写回结果会被本 step 内 `priority` 数值更小的方程读到，
而不是读到上一 step 的旧值。若方程 B 需要读取方程 A 本 step 的最新结果，
应给 A 设置比 B 更大的 `priority`。

```yaml
equations:
  feed_intake:          # 先把奶量加入胃，处理溢奶
    priority: 10
    dynamics:
      stomach_volume: "stomach_volume + intake - max(0, stomach_volume + intake - capacity)"

  gastric_emptying:     # 后执行：读到 feed_intake 本 step 已更新的 stomach_volume
    priority: 0
    dynamics:
      stomach_volume: "stomach_volume - emptying_rate * stomach_volume * step"
```

---

## Euler 离散积分（永久决策）

**本框架永久采用统一 Euler 离散明文表达，直接写出下一时刻的值，不引入 RK4 等高阶积分器。**

```yaml
dynamics:
  blood_glucose: blood_glucose + (uptake - utilization) * step
  position:      position + velocity * step
  velocity:      velocity + (force - damping * velocity) * step
```

理由：生理/社会模型参数不确定性 ±10–50%，Euler 截断误差远低于此；离散事件（进餐、用药）破坏高阶积分器精度优势；明文表达所见即所得。

---
