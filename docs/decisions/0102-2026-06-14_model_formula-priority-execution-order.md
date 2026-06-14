# 0102 — 澄清 formula `priority` 执行顺序与同 step 内的更新可见性

**日期**：2026-06-14
**状态**：✅ 已实施（文档澄清 + 修正一处模型注释/priority 错配）
**类别**：模型规范 / 仿真引擎语义澄清

---

## 背景

`docs/model.md` 与 `docs/LM_format_1.0.md` 此前都写"`priority` 小值先执行（lower = first）"，
但实际引擎实现（`sim_engine/src/model_structure/simulation.py` `_build_formula_cache`/`step()`，
以及 `sim_engine/src/simulator_engine.py` `solve_ode`）均使用：

```python
sorted(formulas.items(), key=lambda x: x[1].priority, reverse=True)
```

即**数值越大越先执行**，与文档描述相反。

这不是空谈差异：`models/papers/s1/infant_breastfeeding.yaml` 中
`gastric_emptying_and_growth`（priority 10）的注释写明"在 `stomach_intake_and_overflow`
（priority 0）之后执行，读取溢奶修正后的胃内奶量"——但按 `reverse=True` 的实际排序，
priority 10 的公式**先于** priority 0 执行，与作者描述的生理顺序（先溢奶修正、再胃排空）相反。

另外，`LM_format_1.0.md` §3.3 原描述"先执行所有 `formula:` 块，再执行所有 `dynamics:` 块"
也与实现不符：引擎对排序后的每条公式做**单次遍历**，在同一条公式内部按
`condition → dynamics → formula(dict) → formula(string)` 的固定顺序求值，
不是按字段类型做两轮全局遍历。

`step()` 中每条公式的 `dynamics`/`formula(dict)` 写回是**立即生效**的
（`var.value` 与 `asteval.symtable` 同步更新），因此本 step 内**后执行**的公式
会读到**先执行**公式刚写入的新值——这是 Gauss-Seidel 式的顺序更新，不是
对上一 step 状态的快照（Jacobi 式）。这一点此前完全未文档化。

## 决策

**保留代码现状（`reverse=True`，数值越大越先执行），更新文档与受影响模型以匹配代码行为**，
不改动引擎代码（多个已发布模型的 `priority` 取值已隐含基于该行为调参，改代码影响面更大且无独立收益）。

### 1. 文档修正

- `docs/model.md`：
  - YAML schema 注释改为"数值越大越先执行"。
  - 新增"公式执行顺序（priority）"小节，说明全局单次排序 + 单条公式内部固定求值顺序
    + 同 step 顺序写入可见性，并给出 feed_intake/gastric_emptying 示例。
- `docs/LM_format_1.0.md`：
  - §3.1 示例注释 `(lower = first)` → `(higher = first)`。
  - §3.3 Execution Order 重写：单次遍历、`condition → dynamics → formula(dict) →
    formula(string)` 求值顺序、Gauss-Seidel 顺序写入语义、`bounds` 在每次写入时
    逐变量裁剪（而非 step 末尾统一裁剪）。

### 2. 修正受影响模型

`models/papers/s1/infant_breastfeeding.yaml`：交换
`stomach_intake_and_overflow`（0→10）与 `gastric_emptying_and_growth`（10→0）的
`priority` 值，使实际执行顺序与作者描述的生理顺序（先溢奶修正、再胃排空，
读取修正后的胃内奶量）一致；同步更新注释中的 priority 数字。

`maternal_sleep_tracking`（20）与 `lm_score_accumulation`（30）相对两者的执行顺序
（仍排在两者之前）不受此次交换影响。

## 影响与验证

- 未改动 `sim_engine` 代码，已有测试/仿真结果不受影响。
- `infant_breastfeeding.yaml` 的两条公式执行顺序发生变化（修正前 gastric_emptying 先于
  stomach_intake_and_overflow 执行；修正后顺序相反），数值结果可能随之变化——
  这是本 ADR 的预期修正（使模拟符合作者描述的生理因果顺序），需要在该模型下次跑
  `--sim`/`--opt` 时关注 `baby_stomach_volume`/`baby_weight`/`spit_up_volume` 轨迹是否
  仍合理（目前未发现该模型有 `_HOLD`/`metadata.todo` 标记）。
- 全仓库 `grep -rn "priority.*(先执行|之后执行|之前执行|before|after)"` 仅命中此一处，
  其余模型的 `priority` 注释不依赖执行顺序描述，无需调整。

## 关联

- `docs/model.md` — 公式步长规则节后新增"公式执行顺序（priority）"小节
- `docs/LM_format_1.0.md` §3.1, §3.3
- `models/papers/s1/infant_breastfeeding.yaml`
