# test_fixtures Fixture 全览

> 本文件是 `models/test_fixtures/valid/` 与 `models/test_fixtures/invalid/` 下每一个 YAML fixture 的导览：按 LM format 的功能领域（evidence、import、mc、optimization、lm_score、schedule/plan、equation、step_size、回归锁定、结构校验）分组，说明每个文件具体测的是什么、为什么要单独测、以及它和相邻文件的关系。目的是替代"文件名 + 一行简介"式的浅层索引，把设计意图讲清楚，读者不需要逐个打开 YAML 也能理解这套 fixture 集合的完整逻辑。
>
> 这些文件不代表真实临床或社会场景（`valid/README.md`、`invalid/README.md` 已说明），也不是本文档要验证的对象——本文档只解释"每个 fixture 在测什么、为什么这么设计"，实际的加载/运行断言由 `test_verification/` 下的 pytest 用例负责（角色划分见 `models/test_fixtures/README.md`）。引擎级 pytest 套件本身的作用范围、数值精度验证，见 life-matters-reference-engine 仓库的 `test_verification/verification_report.md`；文献对标/优化合理性验证是完全不同的另一件事，见 [`models/test_validation/validation_report.md`](../test_validation/validation_report.md)。

## 目录

1. Evidence 证据子类型
2. Import 模块化组合
3. MC 蒙特卡洛与概率参数
4. Optimizer 优化器四层级与附加机制
5. lm_score 累积/闭锁语义
6. Schedule / Plan 时间表与执行计划
7. Equation 方程执行机制
8. Step size 与跨尺度一致性
9. 回归锁定类 fixture
10. 结构性错误检测（未归入以上领域的 invalid 项）

---

## 1. Evidence 证据子类型

`evidence_type` 是 LM format 里"文献效应量 → 引擎可直接使用的有效值"的转换层，声明在 `variables:` 条目上（不再是独立的顶层 `evidence:` 节——evidence 本质上仍是变量，独立成节反而制造两套平行命名空间，故并入 `variables:`，声明 `evidence_type` 的条目 `type` 必须是 `parameter`）。8 种子类型（`rr`/`or`/`hr`/`ard`/`cohens_d`/`ir`/`beta`/`pk`）各自的换算方程不同，需要的辅助字段也不同（`baseline_ref`、`baseline_prevalence`、`population_sd` 等）。如果把 8 种混在一个复杂模型里一起测，一旦某个换算结果不对，很难判断是哪种子类型的转换逻辑出错，还是模型自身的动力学方程写错了。因此先给每种子类型各写一个只含"一个 evidence 变量 + 一两个展示/累积变量"的最小文件单独验证换算是否正确，全部通过后再用一个综合文件确认 8 种子类型能否在同一模型里共存、互不干扰。

- **`test_valid_evidence_ard.yaml`**：绝对风险差（`ard`）子类型，`effective = value`，不需要辅助字段。验证换算后的有效值能被 `applies_to` 自动接入动力学（`stroke_risk_reduction_auto`）与手写动力学（`stroke_risk_reduction`）两条路径，且两者数值必须逐步一致——这是检验"自动接线"和"手写方程"两条实现路径等价的关键点。
- **`test_valid_evidence_beta.yaml`**：回归系数（`beta`）子类型，同样 `effective = value`，但强调换算出的年化系数要能正确折算为日速率（÷365）后驱动一个连续状态变量（`systolic_bp`）。
- **`test_valid_evidence_cohens_d.yaml`**：标准化效应量（`cohens_d`）子类型，`effective = d × population_sd`，是唯一必须依赖 `population_sd` 辅助字段才能算出有量纲结果的子类型，专门检验这个乘法换算和单位对齐（效应量本身无量纲，乘上 `population_sd` 后单位与目标变量一致）。
- **`test_valid_evidence_hr.yaml`**：风险比（`hr`）子类型，`effective = baseline_ir × HR`，是需要 `baseline_ref` 指向同文件内一个 `ir` 证据条目的两个子类型之一（另一个是 `or`）——单独测是因为"引用另一个 evidence 条目做基线"这件事本身有独立的出错空间（基线找错、单位不对齐）。
- **`test_valid_evidence_ir.yaml`**：发生率（`ir`）子类型，`effective = value`，通常本身就是被别的子类型引用的基线，因此也需要单独确认它作为"被引用方"时数值和单位是稳定的。
- **`test_valid_evidence_or.yaml`**：比值比（`or`）子类型，换算方程 `effective = OR / ((1 − p₀) + p₀ × OR)` 是 8 种里最复杂的一个（需要 `baseline_prevalence` p₀ 做非线性变换），单独测是为了在最小文件里把这一步换算的正确性和后续 `applies_to` 接线分开验证。
- **`test_valid_evidence_pk.yaml`**：PK/PD 参数（`pk`）子类型，`effective = value`，重点不是换算本身（本就是直接值），而是验证换算出的速率常数能否在小时级步长的消除方程里直接使用、不需要额外的单位转换因子。
- **`test_valid_evidence_rr.yaml`**：相对风险（`rr`）子类型，`effective = value`，同时对比了"手写动力学乘一个普通 parameter 基线"与"`applies_to` 自动接线到一个 `ir` evidence 基线"两种用法在同一文件里并存但走不同基线、因此数值不应相等的情形——用来确认两条路径不会被意外混用同一个基线。
- **`test_valid_evidence_types.yaml`**：综合文件，一次性验证全部 8 种子类型能同时存在于同一模型、各自的换算值互不干扰、且都能被各自的动力学方程正确引用。这是隔离测试通过后的收尾集成测试。

对应的错误检测（`invalid/`）：
- **`test_invalid_evidence_missing_baseline_ref.yaml`**：`rr`/`or` 使用 `applies_to` 却省略必需的 `baseline_ref`——8 个正例都规规矩矩填了这个字段，这个反例专门确认漏填时加载器会可靠拒绝，而不是静默地把 baseline 当成 0 或抛出无关的错误。
- **`test_invalid_evidence_name_collision.yaml`**：evidence 并入 `variables:` 后，`evidence` 与 `variables` 各自命名空间的重名问题已结构性消失（只剩一个命名空间），改为验证配套的新约束——声明了 `evidence_type` 的条目 `type` 必须是 `parameter`；这个反例故意把 `type` 写成 `state`，确认加载器会拒绝而不是把非 parameter 变量悄悄当成换算结果。

## 2. Import 模块化组合

LM 模型可以通过 `imports` 组合多个子模型文件，典型场景是"多个高层模型共享同一个基础生理组件"。这类组合最容易出错的地方不是单个文件本身，而是**多份 import 共同作用时的行为**：同一个基础组件被两条路径间接引用时会不会被重复加载、override 的优先级顺序对不对、循环引用会不会让加载器卡死。三个正例构成一条两层 + 菱形依赖的完整链路：

- **`test_valid_import_base.yaml`**：纯组件，不含任何 input，只提供心血管相关的状态变量和动力学，专门设计成"只用来被 import，不单独跑仿真"，测试 import 链的最底层。
- **`test_valid_import_layer.yaml`**：中间层，import `base` 并覆盖其中一个参数（静息心率 65→58 bpm，模拟训练有素者），同时自己也会被更上层的文件 import——它的存在制造出"`base` 可以通过两条路径被间接引用"的菱形依赖场景。
- **`test_valid_import_top.yaml`**：顶层，同时直接 import `base` 和 `layer`（而 `layer` 内部也 import 了 `base`），验证加载器在这种菱形依赖下：① 对 `base` 去重（只解析一次，不会把它的变量和方程重复合并两遍）；② override 优先级正确（顶层 > 中间层 > 基础层），顶层对 `layer` 已覆盖过的参数再次覆盖时，最终生效的是顶层的值。

对应的错误检测：
- **`test_invalid_import_circular_a.yaml` + `test_invalid_import_circular_b.yaml`**：两个文件互相 import 对方，构成最简单的循环依赖。之所以需要两个文件而不是一个自我引用的文件，是因为循环必须由"至少两个节点互相指向"才能真实触发图遍历意义上的环——这两个文件必须成对存在，任何一个被单独加载都应该在解析到另一个时立刻检测到环并报错，而不是无限递归或静默丢弃环上的一侧。
- **`test_invalid_import_escapes_root.yaml`**：相对路径 import 向上跳出 `models/` 根目录一层，验证加载器把"import 路径必须留在 models 根目录内"当作硬边界来强制执行，而不是仅仅检查目标文件是否存在。

## 3. MC 蒙特卡洛与概率参数

LM 的概率建模分两个独立的层次：仿真侧的参数不确定性（同一个 `parameter` 声明成分布，多次采样得到一族轨迹）和优化器侧的鲁棒优化（每个候选解在不确定性下被多次评估、取聚合目标函数）。两者用的是同一套分布语法，但作用对象不同，因此分两个文件分别验证：

- **`test_valid_mc_distributions.yaml`**：验证 `normal()`、`uniform()`、`lognormal()` 三种分布形式在"确定性模式"（取分布均值，得到单一轨迹）和"MC 模式"（`runs=5`，独立重采样三个参数各 5 次，轨迹应可见地发散）下都能正确工作，同时确认优化器在 MC 模式下的 Pareto 前沿会因为参数不确定性而比非 MC 版本更宽——这是"分布语法本身"能否被仿真器和优化器同时正确消费的验证。
- **`test_valid_opt_inner_mc.yaml`**：专门验证 `optimization.mc.runs`（内层鲁棒优化）——每个候选剂量在多个 `absorption_rate` 采样下被重复评估，优化器应该使用跨次运行的平均目标函数，而不是被某一次随机采样的噪声牵着走。区分"仿真侧 MC"和"优化器侧 MC"是必要的，因为二者面对的是不同问题：前者回答"这个方案在不确定性下的轨迹分布是什么样"，后者回答"哪个方案在不确定性下平均表现最好"。

## 4. Optimizer 优化器四层级与附加机制

LM 的优化器决策变量按复杂度分为四个层级（Tier），从"只搜数值"到"数值 + 时间/模式/日期同时搜索"。四个层级各自的解码逻辑（把优化算法内部的连续/离散向量翻译回可读的用药/训练方案）是完全不同的代码路径，因此逐层单独验证，再用一个组合文件确认它们能在同一个决策向量里混用：

- **`test_valid_opt_t1_single.yaml`**（T1，单变量单目标）：只搜一个数值（蛋白质摄入量），用梯度类求解器 L-BFGS-B 而非 NSGA-II，验证这条求解器路径本身能端到端跑通、约束在最优点正确生效。
- **`test_valid_opt_t1_pareto.yaml`**（T1，多变量 Pareto）：三个数值型输入 + 一个硬约束（每日总剂量上限）+ 两个目标（最大化疗效、最小化毒性），验证多目标场景下 Pareto 前沿呈现符合直觉的单调权衡，且约束边界内的解才会出现在前沿上。
- **`test_valid_opt_t2.yaml`**（T2，时间窗口）：数值和"一天中的具体时刻"同时作为决策变量，早餐窗口按小时粒度、午餐窗口按 15 分钟粒度分别离散化，验证不同粒度的时间槽位都能正确解码回具体的钟点时间。
- **`test_valid_opt_t3.yaml`**（T3，星期模式）：数值和"一周中训练在哪几天"同时作为决策变量，从预定义的候选日期组合里选择，验证模式下标能正确解码回具体的星期几组合。
- **`test_valid_opt_t4.yaml`**（T4，起始日期窗口）：数值和"干预从哪天开始"同时作为决策变量，验证日期偏移量能正确解码回具体的日历日期，且优化器能在"更早开始+更温和剂量"与"更晚开始+更激进剂量"之间找到正确的权衡。
- **`test_valid_opt_all_tiers.yaml`**：四个层级同时出现在同一个 7 维混合决策向量里（外加一个不参与优化的固定背景变量），是验证各层级解码逻辑不会互相踩到对方维度索引的收尾测试——这类"维度错位"问题只有在多个层级混合时才会暴露，单独测每个层级测不出来。
- **`test_valid_opt_metric_min.yaml`**：验证 `objectives[].metric: min`（`final`/`max`/`min`/`mean` 四种聚合方式里唯一没被其他文件覆盖的一种）——用一个单调递减的库存曲线，让"最大化历史最低库存"这个目标必须取轨迹最小值而非终值，专门排除聚合方式选错导致优化目标名不副实的情况。
- **`test_valid_opt_results.yaml`**：验证三个機制的组合：优化评估窗口比可视化仿真窗口短；`optimization.results` 里预置的历史 Pareto 解在加载时被前端识别并可用于热启动；硬约束和软约束同时存在时都能正确显示和生效。这三者组合在一起是因为它们都属于"结果不是从头算出来的，而是要正确处理已有状态"这一类问题。

对应的错误检测：
- **`test_invalid_optimization_missing_method.yaml`**：`optimization` 块声明了目标/初始点/算法却漏填必需的 `method` 字段，验证这个必填校验能在加载阶段就拦下，而不是等到真正求解时才因为找不到方法而报出无关的错误。

## 5. lm_score 累积/闭锁语义

**`test_valid_lm_score.yaml`**：`lm_score` 有两种语义，累积型（健康天数在指标达标时累加、失代偿时暂停、恢复达标后继续累加）和闭锁型（一旦越过不可逆阈值就永久归零，即使后续指标恢复正常也不再累加）。这个文件专门验证两种语义能在同一个高血压管理模型里独立且正确地共存：血压在阈值内累积健康天数；一旦血压瞬时突破卒中阈值，闭锁型指标立即永久冻结，但累积型指标在血压恢复后应继续正常累加——用同一份轨迹同时检验"暂停/恢复"和"永久冻结"两种不同的时间语义不会互相干扰，是这个字段两种模式唯一需要关注的边界情形。

## 6. Schedule / Plan 时间表与执行计划

- **`test_valid_schedules_phased.yaml`**：验证 `date_range` 和 `days`（星期筛选）两种时间过滤器同时作用于一个 4 阶段周期化训练计划（基础期/强化期/巅峰期/减量期），且同一步内多条日程条目对同一变量的贡献需要正确累加（脉冲求和），而不是互相覆盖。这是验证"多个时间过滤条件叠加"这一组合场景，而不是单独测某一种过滤器。
- **`test_valid_plans.yaml`**：验证 `simulation.plans` 在**没有**全局 `schedules` 块时能独立加载——三个命名方案（保守/均衡/激进）各自定义独立的热量赤字和运动日程，互不污染，且 `output_variables`/`output_types` 的并集规则在多方案场景下正确生效。这与 `test_valid_schedules_phased.yaml` 互补：一个测"多重时间过滤器的叠加"，一个测"多个独立方案之间的隔离"。
- **`test_valid_sustained_mode.yaml`**：验证 `mode: sustained` + `time_range` 这种"持续窗口"式日程条目（区别于默认的单次脉冲事件）在小时步长模型上的行为：默认方案用普通脉冲事件（`--sim` 基线），优化器起点则改用持续窗口（`--opt`），验证同一份日程语法在两种运行模式下都能被正确解析和执行，包括跨越午夜的窗口（夜间恢复窗口跨 20:00–次日08:00）。

对应的错误检测：
- **`test_invalid_date_range.yaml`**：`simulation.end_date` 早于 `start_date`。这个检查刻意不在结构校验阶段做，而是延迟到真正发起仿真/会话时才检查——因此这个 fixture 本身能正常通过结构校验和加载，只有在调用 `run_simulation`/`run_simulation_mc`/`start_session` 时才会报错，用来确认"日期先后顺序"这类依赖运行时上下文的校验没有被遗漏在某条调用路径之外。

## 7. Equation 方程执行机制

- **`test_valid_equation_condition.yaml`**：验证 `equations[].condition` 的互斥分支——两条 `dynamics` 方程作用于同一个状态变量（室温），条件互斥（`< setpoint` 与 `>= setpoint`），模拟一个开关式温控器。核心要验证的是"每一步恰好有一个分支被触发"：不能两个分支都不触发（状态卡住不变），也不能两个分支都触发（状态被双重更新）。
- **`test_valid_equation_edge_cases.yaml`**：覆盖方程表达式解析里几类容易被引擎的表达式求值路径漏掉的边界写法——链式比较嵌在三元表达式里（`1.0 if 0 < level < 10 else 0.0`）、`and` 短路求值保护除法（避免除以零）、以及一个设计成必然在某一步触发 `ZeroDivisionError` 的方程，用来验证引擎对单条方程的求值异常是按 step 捕获并冻结在最后一次成功值（不会让整个仿真崩溃，也不会让 NaN/inf 悄悄传播到下游变量）。这个文件同时记录了一个此前没有被任何测试显式验证过的执行顺序细节：同一 step 内方程按 `priority` 降序执行，且高 priority 的方程读到的是低 priority 方程**本步尚未更新前**的值——这不是一个曾经的缺陷，而是当前执行模型的既定行为，值得作为文档参考单独记下来，供后续任何依赖方程执行顺序的模型设计参考。

对应的错误检测：
- **`test_invalid_equation_undefined_var.yaml`**：`dynamics` 表达式引用了一个从未声明的变量，验证基于 AST 的未定义变量检测能在加载阶段抓住这类拼写错误或遗漏声明，而不是等到运行时才因为找不到变量而报出更难定位的错误。
- **`test_invalid_equation_deprecated_dt.yaml`**：`dynamics` 使用了废弃符号 `dt` 而非现行的 `step`，验证校验器会主动拒绝旧符号，而不是静默兼容导致新旧写法混用、单位含义不一致。

## 8. Step size 与跨尺度一致性

LM 允许不同模型以不同的 `step_size`（分钟/小时/天）各自运行，也允许一个模型 import 另一个使用不同步长的模型。这类跨尺度组合最容易出的错，是"方程该乘 `step` 的地方漏乘、不该乘的地方多乘"，以及"高层模型的步长意外污染了被 import 组件本应保持的步长"。

- **`test_valid_step_hour.yaml`**：小时级步长端到端验证——一个单室 PK 模型，每日固定时刻口服给药，30 天/720 步，验证血药浓度呈现"每日达峰-衰减"并在几天内趋于周期性稳态；同时验证 Nelder-Mead 单目标求解器作为 NSGA-II 之外的另一条求解路径能正确收敛。
- **`test_valid_step_minute.yaml`**：分钟级步长验证——一天三餐、3 天/4320 步，验证日程条目能在亚小时精度下正确触发，基础代谢清除方程正确乘 `step`（速率量）而三餐的脉冲输入不乘 `step`（瞬时量），以及 `output_types: [state, input]` 过滤器只输出指定类型的变量。
- **`test_valid_step_cross_import_base.yaml`** + **`test_valid_step_cross_import_top.yaml`**：核心跨尺度一致性检验——`base` 以天为步长、体重按每天精确衰减 1%，有已知的解析轨迹（day0→100.0、day1→99.0、day2→98.01、day3→97.03）；`top` 以小时为步长运行并 import `base`，验证 `base` 内部的日衰减方程仍然只在每 24 小时的边界上触发一次、且触发时与 `base` 独立运行时的解析轨迹完全一致——既不能因为被小时级模型 import 就被逐小时错误地重复触发（72 倍超算），也不能反过来因为步长不匹配而完全不触发。`top` 自身另有一个独立的小时级 PK 变量同步验证小时级引擎本身工作正常，与跨尺度的日变量互不干扰、分别提供交叉确认。
- Banister 步长收敛系列（`test_valid_banister_v1_analytical.yaml` 及其 5 个步长变体 `_step_{12h,6h,3h,1h,30min}.yaml`）同样属于 step_size 相关的 fixture，但它们的用途是数值精度层面的步长收敛性检验（协议 V4），并非本节讨论的"跨尺度组合是否正确"这类结构性问题——完整方法论和当前执行结果见 life-matters-reference-engine 仓库的 `test_verification/verification_report.md` 第2节，此处不重复。

对应的错误检测：
- **`test_invalid_step_size.yaml`**：`simulation.step_size.value` 为 0，验证校验器要求步长必须为正数，避免除零或死循环之类的下游后果。

## 9. 回归锁定类 fixture

以下两个文件不对应某个"新特性"，而是把此前发现过的一类具体行为，用最小化模型固定下来，防止后续改动无意间再次破坏同样的行为。二者的写法和普通 valid fixture 相同（结构合法、能正常加载和跑通），只是断言的重点是"某个具体数值必须精确等于预期"，而不是"结构或功能是否可用"。

- **`test_valid_pulse_reset_bounds_floor.yaml`**：验证一个下界大于 0 的 input 变量（`bounds[0] > 0`）在脉冲事件触发时，读到的值必须精确等于日程里声明的 `value`，而不是 `bounds[0] + value`。这类变量在每一步都会先被清零、再叠加当前触发事件的贡献值；这个文件锁定的是"清零"这一步不能被下界钳制逻辑影响，否则每次触发都会被下界值污染，越是设了较高下界的变量越容易被这个问题放大。
- **`test_valid_same_day_duration.yaml`**：验证 `start_date == end_date`（仿真区间只有一个自然日）时，仿真必须覆盖完整的 24 小时，而不是退化成 0 或 1 小时的桩窗口。这类边界情况容易在"用日期差计算总时长"的实现里被忽略——同一天的日期差是 0，如果不做特殊处理会导致当天较晚时刻（该 fixture 设在 18:00）的日程事件永远不会被触发。

## 10. 结构性错误检测（未归入以上领域的 invalid 项）

**`test_invalid_yaml_not_dict.yaml`**：顶层 YAML 内容本身是一个列表而不是映射（mapping）。这是所有校验逻辑生效的前提条件——加载器必须先确认 `yaml.safe_load()` 返回的是一个 dict，再往下解析 `variables`/`equations`/`imports` 等字段；如果这一层不做防御，遇到列表或标量类型的顶层内容会在更深的解析代码里抛出令人费解的 `AttributeError`，而不是一条指向根本原因的清晰错误信息。这是唯一一个不属于以上任何功能领域、而是保护"所有其他校验能否正常开始"这一前提的 fixture。

---

## 新增一个 fixture

按功能领域的哪一类新增，具体步骤与命名约定见 `valid/README.md`（正例）和 `invalid/README.md`（反例，含"新增一个错误检测用例"清单）。新增后应在本文件对应章节补一条说明，交代它测什么、为什么需要单独测、与同章节已有文件的关系——而不是只在文件名列表里加一行。
