# ADR 0151 — `references:` 提升为顶层块；M I V E S O R 顶层键顺序

**日期**：2026-08-23
**状态**：已采纳

---

## 背景

`references:` 此前是 `metadata:` 的一个子字段，与 `name`/`version`/`tags`/`authors`/`updated` 这类纯管理性识别信息放在同一个容器里。这些管理性字段大多可以随时增删而不影响模型的科学有效性，但 `references:` 不同：它是"每条引用必须能追溯到真实文献"这条项目级硬性规则的落地位置，是模型可信度与学术可核查性的直接支撑，性质上更接近 `imports:`——都是模型文件对外部世界的强绑定声明，一个绑定其他 LM file，一个绑定文献来源。把 `references:` 埋在 `metadata:` 内部，容易被当作和 `tags`/`updated` 同等重要性的可选装饰信息，在起草和审查模型文件时被忽视。

## 决策

### 1. `references:` 提升为顶层块

从 `metadata.references` 移动为顶层 `references:`，schema 本身不变，仍是 `{citation, description}` 对象的列表：

```yaml
references:
  - citation: "KDIGO 2020 Clinical Practice Guideline for Diabetes Management in CKD. Kidney Int."
    description: "GFR decline rate and protein restriction threshold."
  - citation: "Bauer J et al. (2013) Sarcopenia in CKD. NDT."
    description: "Muscle loss rate under CKD."
```

### 2. 顶层键顺序统一为 M I V E S O R

`metadata` → `imports` → `variables` → `equations` → `simulation` → `optimization` → `references`，读作 M I V E S O R。V.E.S.O. 四个核心动作居中，`metadata`/`imports` 作为前置的身份与组合声明，`references` 作为收束在最后的文献支撑表，呼应论文正文"先讲模型本身、末尾附文献表"的阅读习惯，而不是让读者在还没看到任何机制之前先面对一堆没有上下文的引用条目。

### 3. `references:` 条目与 `variables`/`equations` 的 `reference:` 字段的关系不变

本 ADR 只改变 `references:` 的顶层位置，不改变其 schema，也不要求 `variables.<name>.reference` / `equations.<name>.reference` 这两个已有的、每条目自带的自由文本引用字段指向本块内的某个条目——两者继续独立存在，互不强制关联。让 `reference:` 字段改为指向本块内引用 key（从而让"引用是否真实存在"变成可脚本核查的机器可验证约束）是一个有价值的后续方向，但涉及引擎读取逻辑变更与向后兼容策略，是一个独立的设计决策，不在本 ADR 范围内。

## 引擎影响

无需改动。核查 `reference_engine/src` 全目录，`references`/`checksum` 均未被 loader.py、validator.py 或任何其他模块读取或校验——`metadata.references` 此前只是随 `metadata` 一起被解析进内存的普通字典内容，从未被引擎按路径专门访问。移动到顶层后同样不会被专门访问，纯粹是文档与模型文件的组织约定变化，不影响任何现有 `--sim`/`--opt` 行为。

## 取舍

**放弃**：`references` 与其余管理性字段共享同一个容器带来的简洁性，顶层键数量从五个（含隐式的 `references` 增至六个）。

**获得**：`references` 在结构上获得与 `imports` 同等的顶层地位，直接反映它在模型可信度体系里"强绑定、不可随意丢弃"的角色，不再被淹没在可随时增删的管理性字段之间；为后续"`reference:` 字段指向本块内引用 key、机器可核查引用真实性"这类增强预留了清晰的结构位置。

## 关联工作

- 现有模型文件的 `metadata.references` 需要迁移为顶层 `references:`，是纯机械搬移，不改变引用内容本身；批量执行方式见 `skill_agent/lm-model-calibration.md`。
- `LM_format_1.0.md` 同步更新：§1.1 顶层结构示例改为 M I V E S O R 顺序；§1.2 Metadata Block 移除 `references` 子块；新增独立的 References Block 一节；§8.1 Required Fields 的 `metadata.references` 改为 `references`；相应章节编号顺延。
- 本次改写顺带核查并修复了 `LM_format_1.0.md` 里另一处独立的历史遗留：ADR 0104/0105（2026-06-16，已采纳）早已把 `step_size` 从 `metadata.step_size` 移到 `simulation.step_size` + 逐公式 `step_unit`，但该决定从未同步进 `LM_format_1.0.md`（版本历史表里甚至没有对应的变更记录行），文档 Terms 表、Metadata Block 示例、Equation Expression Language、Simulation Block、Optimizer Independence Principle 表、优化器 YAML 示例注释、§6.1 都还停留在 ADR 0104 之前的旧字段路径，本次一并修正，是文档追平既有 ADR 决定，不是本 ADR 新增的设计决策。
