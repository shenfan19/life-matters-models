# models/references — 来源模型构件

## 定位

`references/` 存放**各论文场景所依赖的基础生理/社会动力学子模型**，是整个模型生态的基础构件层。

每个文件描述一个独立的生理系统或机制（肾功能、运动疲劳、PK/PD 等），供上层场景（`papers/`、`scenarios/`）通过 `imports:` 机制组合调用。

**不在这里放的内容**：
- 完整可运行的仿真场景（放 `papers/` 或 `scenarios/`）
- 游戏内容（放 `stories/`）

## 目录结构

```
medical/
  disease/      慢性病进展模型（CKD、糖尿病、高血压等）
  fitness/      运动适应与疲劳（Banister 等）
  medicine/     药物动力学（PK/PD，一阶吸收/清除）
  nutrition/    营养素代谢（蛋白质、能量、水分）
  physiology/   生理基础（肾小球滤过、肌肉合成、ALT动力学等）
  psychology/   心理健康与认知模型
  surgery/      手术围手术期模型
social/
  conflict/     冲突与压力模型
  demography/   人口统计动力学
  economy/      经济收支模型
  law/          法律与政策约束
  psychology/   心理健康与认知模型
  technology/   技术扩散模型
```

## 使用方式

```yaml
# 在 papers/ 或 scenarios/ 的场景文件中通过 imports 引用
imports:
  - medical/fitness/banister_fitness_fatigue
  - medical/disease/ckd_renal_filtration
```

Loader 递归合并导入的子模型，根文件中的同名变量/方程覆盖子模型定义。

## 验证状态（sim validation）

每个"可独立分析的参考模型"（含 `type: input` 变量的模型，见 `docs/authoring/imports_and_organization.md` §"references/ 目录约定"）
都应有多个 `simulation.plans`，用仿真曲线相互对照来验证模型自身的方向性正确——而不是只跑一次
`--sim` 看不报错就算过关。方法：给同一模型写 2-3 个机制上应产生不同、可预期方向的方案，
跑 `--sim` 逐一核对曲线是否符合预期方向；核对通过后再跑 `--opt` 确认前沿非退化（不是所有解
挤在一点），最后才摘掉 `metadata.todo` 里的 `nosim`/`noopt` 标记。

**当前已完成本轮验证的模型**（2026-07-06）：

| 模型 | plans 数 | 验证要点 |
|------|---------|---------|
| `medical/physiology/banister_fitness_fatigue_2026.yaml` | 3 | 适应/疲劳双时间常数机制；发现 `bounds` 过窄导致 performance 恒为0 的模型级 bug 并修复 |
| `medical/disease/chronic/ckd_protein_muscle_2026.yaml` | 3 | 低蛋白护肾 vs 高蛋白保肌的方向性权衡；发现并触发了下述引擎级 bug 的排查 |
| `medical/nutrition/diet/mediterranean_diet_2026.yaml` | 3 | 依从性与红肉拮抗效应；修正 LDL 速率常数换算错误（3年误算成7年）；opt 前沿诚实退化为单点——唯一决策变量对 LDL 单调有益但不影响 CRP，两目标间无真实冲突，非 bug |
| `medical/fitness/individual/running_2026.yaml` | 3 | 配速-乳酸-疲劳-表现耦合；发现"单日模型只跑1小时"引擎 bug，并将 optimizer 目标从"末端表现"改为"消耗热量"以消除退化前沿 |
| `social/demography/population/population_growth_2026.yaml` | 3 | 生育政策通过出生率影响人口结构；模型原用 `step_unit: month`（当前 format 不支持），已改写为 day 级步长 |
| `social/economy/labor/labor_economic_2026.yaml` | 3 | 加班/休假/技能投资的收入-疲劳-生产力权衡；模型原用 `step_unit: week`（当前 format 不支持）且完全缺失 `simulation.plans`，均已补齐 |

其余 `references/` 下 62 个模型仍带 `metadata.todo` 的 `nosim`/`noopt` 标记（未在本轮验证范围内），
按同样方法逐一核实前先不要假定其 `optimizer.results` 或 `description.result` 数值可信。

**批量发布提示**：本项目"可发布"的唯一判据是 `metadata.todo` 是否存在/非空（ADR 0120，与文件名无关）。
当前 `references/` 下 `metadata.todo` 为空的文件共 13 个（上表 6 个 + 此前已无标记的 7 个，均未在
本轮验证范围内、也未按上述两个引擎 bug 复核过——批量发布前建议至少对这 7 个也跑一遍
`--sim`/`--opt` 确认没有中招）。查询命令（在 `models/references/` 下执行，需要 PyYAML）：

```python
import yaml, glob
for f in sorted(glob.glob('**/*.yaml', recursive=True)):
    data = yaml.safe_load(open(f, encoding='utf-8'))
    if isinstance(data, dict) and not (data.get('metadata') or {}).get('todo'):
        print(f)
```

### 典型示例：sim 验证了 LM 的哪些功能

举 3 个有代表性的（其余同类不逐一列出）：

1. **`running_2026.yaml` 的 `threshold_intervals` 方案**：验证了 sustained 区间"窗口总量"语义
   （ADR 0099，`time_start`≠`time_end` 时 `value` 按窗口内 step 数摊分）与非线性阈值机制的组合——
   RPE=8/配速4.5min/km 持续60分钟后，乳酸从0.5冲到18.5 mmol/L（远超4 mmol/L阈值区），运动表现从
   100分崩溃到2.1分，与模型自身文档描述的"40-50分钟后明显下降"完全吻合。
2. **`ckd_protein_muscle_2026.yaml` 的三方案对照**（0.6 / 0.8 / 1.2 g/kg/day）：验证了"多个 plan
   相互印证模型方向性"这一核心方法——蛋白摄入越低，GFR终值越高（43.06→42.01）但肌肉质量终值越低
   （24.36→35.28kg），两条曲线严格反向，与模型自身描述的核心临床权衡一致；也是本轮发现 pulse
   重置 clamp bug 的原始案例（该模型此前无 `nosim`/`noopt` 标记，看似"已验证"，实为引擎 bug 静默污染）。
3. **`labor_economic_2026.yaml` 的 `skill_investment` 方案**：验证了多方案对比中"短期代价、长期
   收益"的跨期权衡——相比均衡方案，该方案年末存款更低（238627 vs 249399元）但生产力系数翻倍
   （1.0→2.0），量化体现了模型问题陈述里"人力资本投资有短期机会成本"的核心张力。

## 贡献规范

见 [`docs/model_requirements.md`](../../docs/model_requirements.md)。每个组件文件必须包含：
- `metadata.description`（机制说明）
- `metadata.references`（文献来源）
- 每个变量的 `description` 和 `unit`

每个模型文件都欢迎任何用户参与编辑、补充参数来源或修复问题。
