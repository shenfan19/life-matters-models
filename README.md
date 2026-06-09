# Life Matters · 模型库

> LM format（Life Matters format）格式的 YAML 模型库。  
> 仿真引擎见 → **[life-matters](https://github.com/shenfan19/life-matters)**

---

## 这是什么

本仓库是 Life Matters 项目的模型内容库，收录基于公开文献的生理、营养、疾病和社会动力学模型。  
每个模型是一个 YAML 文件，按 LM format 格式编写，可由 LM 仿真引擎直接运行和优化。

LM format 是一种开放的 YAML 格式标准，类似 SBML / CellML，但专注于：
- **个体尺度**的健康与行为动力学（分钟～年）
- **行为干预调度**（饮食、运动、用药时序）
- **多目标 Pareto 优化**（搜索最优干预方案）

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
  model.md      YAML Schema 参考（变量、公式、调度、优化器）
  quickstart.md 30 分钟写出第一个模型
  model_ratings.md 模型质量评分体系
  decisions/    YAML 格式和模型库结构的架构决策记录（ADR）
```

---

## 文件名质量标记

| 后缀 | 含义 |
|------|------|
| 无后缀 | sim ✓、opt ✓（或无 optimizer 块）、所有参数有文献来源 |
| `_nosim` | 仿真引擎无法运行（解析错误、变量引用错误等） |
| `_noopt` | sim 通过，optimizer 块存在但运行失败 |
| `_noref` | 缺乏文献来源（存在 `TODO:SOURCE`） |

---

## 快速开始

**阅读格式规范**：[docs/quickstart.md](docs/quickstart.md) → [docs/model.md](docs/model.md)

**运行模型**：需要配套仿真引擎，见 [life-matters](https://github.com/shenfan19/life-matters)

**批量测试**（在 life-matters 仓库下执行）：

```bash
# 测试 references/ 下全部模型
FILTER_BROKEN=false bash script/test_batch.sh

# 只测有问题的模型（修复模式）
bash script/test_batch.sh
```

---

## 文档索引

| 文档 | 内容 |
|------|------|
| [docs/LM_format_1.0.md](docs/LM_format_1.0.md) | LM format 格式规范全文（学术版） |
| [docs/model.md](docs/model.md) | YAML Schema 完整参考（变量类型、公式、调度、优化器） |
| [docs/quickstart.md](docs/quickstart.md) | 入门指南：30 分钟写出第一个模型 |
| [docs/model_ratings.md](docs/model_ratings.md) | 模型质量评分体系（metadata.ratings 字段说明） |
| [docs/DECISIONS.md](docs/DECISIONS.md) | 格式与库结构架构决策索引 |

---

## 贡献

见 [CONTRIBUTING.md](CONTRIBUTING.md)

---

## Acknowledgments

This project was developed with AI coding assistance, primarily [Claude Code](https://claude.ai/code) (Anthropic), for code generation, automated testing, and documentation.

## License

CC BY 4.0 · Copyright (c) 2026 Fan Shen
