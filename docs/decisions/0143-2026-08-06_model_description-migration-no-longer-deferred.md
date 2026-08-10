# ADR 0143 — description 列表结构与 references 贡献说明的迁移不再是一次性范围安排

## 状态

✅ 已实施

## 日期

2026-08-06

## 背景

ADR 0142 把 `papers/` 模型 description 的四字段列表写法与 `metadata.references` 的 `{citation, description}` 两字段写法定为写作规范，但落地时只重写了 `s1/infant_breastfeeding` 目录下两个文件，并在"非目标"一节明确写着"不追溯批量重写 `models/papers/` 下其余模型……历史模型的迁移是后续独立工作"。

这一表述当时指的是那次任务自身的范围安排——ADR 0142 的决策过程本身还在三种写法之间反复试验，只在 `infant_breastfeeding` 两个文件上验证新写法是否可行，尚不适合直接推广。但这句话留在 ADR 正文里之后，被后续任务当成了一条可以引用的长期例外：一次只做机械性引用核查的任务（S2 masld_insulin/ibs_diet、S3 smoking_stress、S4 hypertension_gout 四个模型）因为这句"非目标"表述，没有顺带把接触到的这四个模型的 `description` 迁移到新格式，`references` 也没有改成两字段写法，即使那次任务本身就在给这些模型补充引用。这与"每篇来源文献具体贡献了什么，理应随手写清楚"的初衷相悖——补引用的同时不把贡献说明写清楚，等于把同一件事分两次做。

## 决策

1. **废止 ADR 0142"非目标"一节里"不追溯批量重写"的表述**，该表述今后不再作为跳过迁移的理由。
2. **触发时机明确为**：此后任何原因编辑某个 `papers/` 模型文件——哪怕只是核对引用、修一个字段、修一处 bug——都应顺带把该模型的 `description` 迁移到 ADR 0142 定义的四字段列表结构，并把 `metadata.references` 迁移到 `{citation, description}` 两字段写法，不必等待、也不再需要"专门的批量迁移任务"这个前置条件。
3. **`references` 的 `{citation, description}` 写法从"可选注释"升级为 `papers/` 模型的推荐默认写法**：每篇来源文献具体贡献了什么机制或数据，是这个模型相对单篇源文献的价值所在，也是审阅者核对"耦合是否成立"最直接的依据，理应随手写清楚，不是可有可无的锦上添花。
4. 尚未迁移的历史模型不因这条 ADR 立即被批量重写；迁移仍按接触到该文件的先后顺序自然发生。但从这条 ADR 之后，"这次任务范围不含迁移"不再是默认可以省略的理由，只有用户在具体任务里明确说明本次范围不含迁移时才可以跳过。

## 影响

- `docs/authoring/description_writing.md` 增补"触发时机"说明，并把 `references` 对象写法的措辞从"可选"改为"推荐默认"。
- 后续任何触碰 `models/papers/` 模型的任务，缺省应包含 description/references 迁移，除非用户另有说明。
- `models/papers/` 下除 `s1/infant_breastfeeding` 外的其余模型（S1 的 banister/ckd_protein/diuretic_tradeoff/fatty_liver/homair_ogtt，S2/S3/S4 全部模型）陆续按这条规则迁移。

## 非目标

- 不改变 ADR 0142 关于四字段结构本身、列表写作规则、引用格式（句末括号夹注、不用数字编号）的实质内容，这些规则不变，只是移除了它们"不必追溯应用"的例外。
- 不要求一次性并发批量重写所有历史模型；迁移按任务接触到的模型逐个进行。
