<img src="icon.svg" width="48" height="48" alt="Life Matters icon" />

# Life Matters · 模型库

> Life Matters（LM）项目的模型内容库，是整个项目的根基：LM format 格式规范与基于公开文献的模型内容都发布于此。  
> 仿真与优化引擎见 → **[life-matters-reference-engine](https://github.com/shenfan19/life-matters-reference-engine)**（LM Reference Engine，LM format 的参考实现）

---

## 免责声明

本项目中的历史与医学场景基于公开学术文献，仅用于健康决策教育目的。所有模拟内容不代表对历史人物的道德评判；历史数据经简化处理，不构成医学建议；仿真结果为模型推演，非历史事实重现。

---

## 这是什么

本仓库是 Life Matters 项目的根基仓库，收录 LM format 格式规范本身，以及基于公开文献的生理、营养、疾病和社会动力学模型。  
每个模型是一个 YAML 文件，按 LM format 格式编写，可由 LM Reference Engine 直接运行和优化。

LM format 是一种开放的 YAML 格式标准，类似 SBML / CellML，但专注于：
- **个体尺度**的健康与行为动力学（分钟～年）
- **行为干预调度**（饮食、运动、用药时序）
- **多目标 Pareto 优化**（搜索最优干预方案）

核心能力：

1. 把医学/社会学文献里的统计结论（OR、HR、Cohen's d 等）转化为可运行的 YAML 动力学模型
2. 在统一框架内同时运行异尺度模型（分钟–小时–天–年）
3. 对行为干预方案（Regimen）做多目标 Pareto 优化
4. 把多篇文献的参数装进同一框架，检验它们是否互相自洽（Simulation-as-Validation）

一个 LM file 由四个顶层机制组成，合起来读作 V.E.S.O.：`variables`（可迁移的数值证据）、`equations`（把证据接成随时间演化的动力学）、`simulation`（跑出轨迹）、`optimizer`（在决策空间里搜索权衡）。四问判断一个候选话题是否落在这个范围内，详见 [`docs/LM_format_1.0.md`](docs/LM_format_1.0.md) Scope 一节的 Inclusion Test。

本仓库的模型不是逐篇复现单一研究结论，而是把多篇独立文献各自验证过的机制放进同一个模型，让原本互不知晓彼此存在的机制产生真实的相互作用，显现出单篇论文各自的建模范围内看不到的权衡。判断一个模型该纳入哪些机制、又该剔除哪些机制的方法论详见 [`docs/authoring/methodology.md`](docs/authoring/methodology.md) 开篇的"LM 的核心方法论：耦合，不是堆叠"一节。

---

## 内容可信度声明

本仓库的定位接近一个面向 AI 时代的、可计算的科学参考库：模型内容由 AI 大量参与生成，这是这个时代无法回避的现实，让这类内容变得可核对、可纠错，是这个仓库存在的意义之一。为此明确两条边界：

- **作者本人负责**：LM format 格式规范，即 [`docs/LM_format_1.0.md`](docs/LM_format_1.0.md)；配套仿真与优化引擎，见 [life-matters-reference-engine](https://github.com/shenfan19/life-matters-reference-engine)；以及少数已标注为"强验证"的核心示范模型，判定标准为 `validation_confidence` 不低于 4，完整名单见 [`docs/authoring/methodology.md`](docs/authoring/methodology.md) 的纳入标准盘点表与 [`models/test_validation/validation_report.md`](models/test_validation/validation_report.md)。
- **模型库其余内容**：大部分模型文件由 AI 辅助生成与初步复核，**尚未经过相关领域专家核实**，仅供方法论演示与测试参考，不构成临床或科学结论。每个模型 `metadata.ratings.validation_confidence` 字段标注当前验证程度，量表为 1 到 5 分，定义见 [`docs/authoring/ratings.md`](docs/authoring/ratings.md)；纳入标准盘点表和验证报告如实记录了哪些学科、哪些模型已验证，哪些仍待评估，请据此判断可信度，不要默认已发布的模型就是已核实的。

如果你是相关领域的专家，发现某个模型的参数、机制或结论有误，欢迎提交 issue 或 PR 指出——这正是模型以开放、可核对的 YAML 格式发布而非锁在私有工具里的原因。

---

## 仓库结构

```
models/
  references/   基于文献的参考组件模型
    medical/      生理、营养、疾病、药理
    social/       经济、冲突、心理、人口
  papers/       与论文绑定的完整场景（含优化结果）
  scenarios/    组合场景（开发中）
  temp/         未验证草稿（gitignore）
output/         批量测试输出（gitignore）
docs/
  LM_format_1.0.md   格式规范全文
  model/        建模实践指南，索引见 model/README.md（方法论、写作规范、评分、regimens/optimizer 等）
  quickstart.md 30 分钟写出第一个模型
  decisions/    YAML 格式和模型库结构的架构决策记录（ADR）
```

每个模型目录的具体内容、验证状态和贡献规范见各自的 README：[`models/references/README.md`](models/references/README.md)、[`models/papers/README.md`](models/papers/README.md)、[`models/scenarios/README.md`](models/scenarios/README.md)。

---

## 状态标记

模型是否"可发布"只看 `metadata.todo` 字段，与文件名无关：

- **无 `metadata.todo`（或为空）= 已确认通过、可发布**：sim ✓、opt ✓（或无 `optimizer` 块）、所有参数有文献来源。
- **有 `metadata.todo`** = 存在待处理事项，详情见 [docs/authoring/bookkeeping.md](docs/authoring/bookkeeping.md)「状态标记」一节（`type: nosim/noopt/noref/quality/other` + 诊断证据）。

---

## 快速开始

**阅读格式规范**：[docs/quickstart.md](docs/quickstart.md) → [docs/LM_format_1.0.md](docs/LM_format_1.0.md) → [docs/authoring/README.md](docs/authoring/README.md)

**运行模型**：需要配套的 LM Reference Engine，见 [life-matters-reference-engine](https://github.com/shenfan19/life-matters-reference-engine)

**批量测试**（在 life-matters-reference-engine 仓库下执行，详见 [cli.md](https://github.com/shenfan19/life-matters-reference-engine/blob/main/docs/cli.md)）：

```bash
python cli/batch.py --input-dir models/references
```

---

## 文档索引

| 文档 | 内容 |
|------|------|
| [docs/LM_format_1.0.md](docs/LM_format_1.0.md) | LM format 格式规范全文（学术版） |
| [docs/authoring/README.md](docs/authoring/README.md) | 建模实践指南索引（写作规范、评分、regimens/optimizer 等） |
| [docs/quickstart.md](docs/quickstart.md) | 入门指南：30 分钟写出第一个模型 |
| [docs/authoring/ratings.md](docs/authoring/ratings.md) | 模型质量评分体系（metadata.ratings 字段说明） |
| [docs/DECISIONS.md](docs/DECISIONS.md) | 格式与库结构架构决策索引 |

---

## 贡献

见 [CONTRIBUTING.md](CONTRIBUTING.md)

---

## Acknowledgments

This project was developed with AI coding assistance, primarily [Claude Code](https://claude.ai/code) (Anthropic), for code generation, automated testing, and documentation.

## License

CC BY 4.0 · Copyright (c) 2026 Fan Shen
