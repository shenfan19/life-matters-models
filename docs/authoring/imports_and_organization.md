# Imports 与模型组织

## Imports 与输出选择

`imports` 只支持显式路径：

- `papers/paper2/ckd_protein_a4_p2`：从 `models/` 根目录出发，省略 `.yaml`，不写 `models/` 前缀。
- `references/medical/physiology/glucose_regulation_2026_mw`：同上，深层路径写全即可。
- `./local_component`、`../paper1/foo`：从当前 YAML 所在目录出发。

裸名字检索已禁用，例如 `imports: ckd_protein_a4_p2` 不递归搜索 `models/`，须写出完整相对路径。

### 合并顺序与覆盖规则

**合并顺序：**

1. imports 按列表顺序加载，**靠后的覆盖靠前的**
2. **当前文件始终覆盖所有 imports**，无论 imports 列表怎么写
3. 循环 import 自动报错（A → B → A 不允许）
4. 同一文件被多次 import（菱形依赖：A → B、C，B → D，C → D）时只加载一次，不重复叠加

**覆盖机制（deep merge）：**

合并算法是全字段递归深合并——**不只是 `simulation` 和 `optimization`，所有顶层块（`variables`、`equations`、`metadata`、`simulation`、`optimization`）都适用相同规则**：

| 情况 | 结果 |
|------|------|
| 根模型和 import 都定义了同名 variable/equation | **根模型的版本完全替换** import 的版本（深合并：子字段也按 root 优先） |
| 只有 import 定义的 variable/equation | **保留**，根模型不影响它 |
| 根模型和 import 都有 `simulation.start_date` | **根模型的值覆盖** import 的值 |
| import 有 `simulation.plans`，根模型没有 | **保留** import 的 `plans` |

典型用法：component 模型（`references/` 下）通常有自己的 `simulation` 块用于独立运行，import 后根模型的 `simulation` 会覆盖其起止日期和步长——这是预期行为，component 的仿真配置仅作组件独立运行用。

**输出变量选择规则：**

- 根模型**未定义** `simulation.output_variables` 和 `output_types`：继承最后一个 import 的输出选择（与其他字段的 deep merge 行为一致）
- 根模型**显式定义了任一**输出字段：根模型定义优先；若仅定义其中一个，另一个从 import 继承的值同时清除
- 两个字段都不存在或都为空：输出所有变量

GUI 读取模型时会显示 resolved model：变量、方程、输出变量、`simulation` 和 `optimization` 都包含 imports 合并后的结果。模型页会标出各字段来自哪个 YAML（provenance）。
- `output_types` 只支持 `input`、`parameter`、`state`。
- `output_variables` 中不存在的变量会被跳过，并在 API/GUI 中给出 warning；不会再生成零值曲线。

---

## 模型分类体系

三层目录：`models/references/{L1}/{L2}/{L3}/file.yaml`

| L1 | L2 | 说明 |
|----|----|----|
| medical | physiology / nutrition / fitness / disease / medicine / surgery / psychology | 生理与医学 |
| social | economy / conflict / law / psychology / technology / demography | 社会经济与社会学 |
| environmental | climate | 环境科学 |
| risk | actuarial | 精算与风险 |

完整 L3 细分见 `docs/decisions/0022-models-three-level-taxonomy.md`。

### `references/` 目录约定

`references/` 下的模型分两类，optimization 要求不同：

| 类型 | 特征 | optimization 要求 |
|------|------|--------------|
| **可独立分析的参考模型**（fitness、disease、nutrition 等）| 有自己的 `input` 变量和 `simulation.plans[*].regimens`，可直接运行 | **应有** `optimization` 块 |
| **深层生理组件**（physiology/ 下的 `_mw` 系列，如 `digestive_system`、`insulin_system`、`glucose_regulation` 等）| 无 `input` 变量，主要为 `import` 的积木，单独运行无生理意义 | **不需要** `optimization` |

判断原则：若模型的 `variables` 中没有 `type: input` 的变量，说明它是纯组件，不需要 optimization。

---
