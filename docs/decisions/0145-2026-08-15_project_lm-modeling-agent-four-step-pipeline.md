# 0145 — LM 建模协作 Agent 四步流水线与文档存放

**日期**：2026-08-15
**状态**：✅ 已接受

---

## 背景

LM 建模工作此前依赖用户手动完成"找方向 → 起草 YAML → 诊断 → 提出并验证候选修改"这一整套流程，每次都要重新说明规范、重新解释诊断维度，协作指令没有沉淀下来。随着 Claude Code 的 subagent 机制可用，具备条件把这套协作流程固化为可复用、可移植的文档，供 AI 助手按标准化步骤执行，同时保留人工核实作为最终把关。

## 决策

采纳四步流水线，完整指令文档存放在本仓库 `agents/` 目录：

- **Step 1** `lm-modeling-inspiration.md`：灵感发现。基于学科覆盖盘点表（`docs/model.md`）或用户给定方向做文献检索，产出候选建模方向清单，不创建任何文件。
- **Step 2** `lm-modeling-design.md`：起草与结构性修改。把一个方向或结构性改动需求写成可运行的 model YAML 草稿，草稿一律落在 `models/temp/`，不碰正式模型文件。
- **Step 3.1** `lm-modeling-diagnosis.md`：诊断。对一个 LM 模型做全面诊断——决策变量边界、可行域结构性冲突、仿真轨迹的生理边界违反、Pareto 前沿形态、引用缺口、自报的已知问题——产出统一诊断表。不要求模型必须有 `optimization:` 块，sim-only 模型同样可诊断；不具备编辑能力。
- **Step 3.2** `lm-modeling-advisor.md`：研究与执行。基于诊断表研究文献依据、提出候选建模修改，实际编辑模型副本、跑 life-matters-reference-engine 验证并按结果迭代。绝不编辑正式模型文件，只编辑模型目录下的 `temp_probe/`、`temp_advisor/` 副本。
- **Step 4** `lm-agent-report.md`：任务报告约定，不是独立触发的 agent，是前四份文档共同遵守的收尾规则——任一 agent 实际产出了会被后续引用/依赖的东西之后，按模板记录前因后果、任务、环境、过程、结果、可复现的期望、后续状态。

Step 3.1 与 3.2 共同构成"调整"阶段，通常配套使用；Step 1 与 Step 2 各自独立。

### 文档存放位置：完整指令在 models 仓库，reference_engine 仓库只放触发入口

完整指令统一维护在本仓库 `agents/*.md`，不绑定任何特定 AI 工具或框架——文档本身设计成可以整段读给任意支持长上下文指令的 AI 助手照做，不依赖 Claude Code 专有机制。

`life-matters-reference-engine/.claude/agents/` 下放对应四份文档的薄封装 stub，每份只做一件事：指向本仓库 `agents/` 目录下的同名文档并读取执行；若姊妹仓库不存在则如实告知用户无法继续。该仓库 `.claude/` 目录整体被 `.gitignore` 排除，这些 stub 不进入版本库——真实内容的单一事实来源始终是本仓库的 `agents/` 目录，Claude Code 只是众多可能的执行环境之一。

### 默认调用引擎验证，明确说明才退化为纯文本分析

Step 2/3.1/3.2 默认调用 life-matters-reference-engine 做验证（Step 2 是语法/可运行性验证，Step 3.1/3.2 是数值验证）。退化为纯文本分析（只读 YAML、逻辑推理、网络检索，不运行任何计算）需要用户在对话里明确说明；调用环境中若根本不存在 `life-matters-reference-engine` 仓库（两仓库须以兄弟目录形式存在），同样自动退化并说明原因。Step 1 不涉及引擎，只做检索。

### 临时文件约定

各文档在模型自己的目录下按需新建 `temp_probe/`、`temp_advisor/` 子目录存放临时副本；Step 2 的草稿统一放 `models/temp/`。`.gitignore` 新增 `**/temp_*/` 规则，任何深度、任何 `temp_` 前缀后缀的目录均被排除，避免探针/草稿副本污染正式模型文件或被误提交。

### Step 4 报告存放于内部仓库

Step 4 产出的实际任务报告存 `life-matters-home/agent_reports/`，不进公开仓库。与 `models/test_validation/validation_report.md` 的先例一致：先在内部积累真实使用记录，等这套 agent 流程有了可信的使用轨迹，再决定摘取哪些片段对外公开；公开仓库目前没有任何文件引用该目录，不预先放置占位文件。

## 影响范围

- 新增 `agents/README.md`、`agents/lm-modeling-inspiration.md`、`agents/lm-modeling-design.md`、`agents/lm-modeling-diagnosis.md`、`agents/lm-modeling-advisor.md`、`agents/lm-agent-report.md`。
- `.gitignore` 新增 `**/temp_*/` 规则。
- `life-matters-reference-engine/.claude/agents/` 新增四份本地 stub（不进版本库）。
- `life-matters-home/agent_reports/` 作为 Step 4 报告的内部存放目录，首份报告已回溯记录 2026-08-15 当天 ibs_diet/masld_insulin/bergman_glucose 三个模型的 Step 3.2 实际运行。

## 结果

- 建模协作流程从"每次口头重新说明"变为四份可复用、可移植的标准化文档，权限边界（谁能创建文件、谁能编辑正式文件、谁只能诊断不能改）显式写入各自文档而非依赖临时约定。
- 引擎验证默认开启，保证草稿/候选修改在交回用户前已过语法或数值检验，而不是停留在纯文本推理层面。
- 正式模型文件的编辑权限保持收敛：四份文档中仅 Step 3.2 能编辑模型副本，且明确排除正式文件；最终是否采纳仍需人工核实。

## 未决

- Step 4 报告是否需要以及何时摘取片段对外公开，本 ADR 不预先决定，留待后续视使用轨迹判断。
