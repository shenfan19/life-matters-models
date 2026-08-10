# ADR 0144 — `formulas:` 顶层字段更名为 `equations:`，四要素助记符改为 V.E.S.O.

## 状态

✅ 已实施

## 日期

2026-08-08

## 背景

LM format 的四个顶层机制此前分别叫 `variables`、`formulas`、`simulation`、`optimizer`，简写成 var/for/sim/opt。这个简写在讨论里一直不直观——"for"既不是一个能连读成词的助记符，也容易被误认成英文介词 for，需要额外解释才能记住它指的是哪个字段。

重新梳理这四个要素时发现，规范正文本身描述这个字段用的词从来就是 differential equation、regression equation、kinetic equation、dynamic equations，而不是 formula——多数条目是控制 dynamics 的微分方程，是随时间演化的关系，不是一次性代入数值算结果的静态公式（如 BMI 公式）。`equations` 才是和规范自己已经在用的描述语言对齐的字段名，不是引入新含义。改名之后四要素简写可以读作 V.E.S.O.（Variables, Equations, Simulation, Optimization），是一个能连读发音的助记符，比 V.F.S.O. 更容易记忆和对外传播。

版本号规则（`LM_format_1.0.md` §9.2）里 major bump 只在"破坏旧文件兼容性"时触发，其保护对象是已经依赖某个冻结版本的外部使用者。`LM_format_1.0.md` 版本历史里 v1.0 的每一条记录都标注为 Draft，规范至今没有对外冻结发布，也就没有"已经依赖旧字段名的外部消费者"需要保护——这次改名是在编辑草案，不是对已发布契约做破坏性变更，因此不需要 major bump，仍是 v1.0（changelog 新增一行记录本次改动，见该文件 §9.1）。

## 决策

### 1. `formulas:` 顶层字段更名为 `equations:`

所有 LM file（`life-matters-models` 与 `life-matters-game` 两仓库 `models/` 下共 241 个文件）的顶层 `formulas:` 键改为 `equations:`。`life-matters-game` 的卡牌 YAML（`source.formula` 溯源标注字段，362 个文件）同步改为 `source.equation`，与 `reference_engine/src/routes/converter.py` 生成这批文件时写入的字段名保持一致。

### 2. 四要素简写改为 var/equ/sim/opt，助记符改为 V.E.S.O.

`docs/authoring/methodology.md` 纳入标准盘点表的列名、`LM_format_1.0.md` Inclusion Test 的四问、`life-matters-home/process/lm_nomenclature.md`、各仓库 README 的介绍语言同步改为 V.E.S.O. 表述。

### 3. 代码、GUI、i18n 同步改名

`reference_engine` 后端：`Formula` 类改名 `Equation`，模型对象的 `.formulas` 属性、相关变量名、API 响应字段同步改名。`gui` 前端：组件、类型定义、四语言 i18n 文件（`en`/`zh-CN`/`zh-TW`/`fr`，含 i18n key 本身）同步改名，`fr.json` 一并修正了 "de équation" 应作 "d'équation" 的省音问题。`life-matters-game` 前端（`StoryEditor.tsx`/`StoryEngine.tsx`）同步改名。

### 4. 测试夹具改名

`models/test_fixtures/` 下 4 个文件名含 `formula` 的 fixture（`test_invalid_formula_undefined_var.yaml` 等）改名为 `equation` 对应名，文件内 `metadata.name` 与交叉引用同步更新；`life-matters-reference-engine` 侧引用这些文件路径的 3 个测试文件（`test_structural_errors.py` 等）同步更新路径与函数名。

## 改写范围的边界

`life-matters-models`/`life-matters-reference-engine`/`life-matters-game` 三仓库内，除下面明确列出的例外，所有文件——含 model YAML 的 `metadata.log`/`todo`/`change` 字段、两仓库索引文件（`DECISIONS.md`/`decisions/README.md`）描述历史 ADR 内容的摘要文字——都按本次改名统一改写，不保留"这里曾经叫 formula"式的更改痕迹。`life-matters-home` 仓库仅论文草稿（`paper/*.md`）与 `validation/validation_report.md` 按同一标准统一改写；`home` 下其余内容（`tasks/`、`process/` 除已改的活文档外、`personal/`、`outreach/` 除 slides 示例外）不要求追溯改写。

例外（保留原状，不属于遗漏）：

- **两仓库 `docs/decisions/` 下已接受的历史 ADR 正文与文件名本身**（如 0068 `formula-precompile-to-python-function.md`、0102 `formula-priority-execution-order.md`、0104、0106）——ADR 是本项目的决策记录载体，保留其原始文字与文件名；索引文件里指向这些 ADR 的链接文字（如"0068 方程预编译"）已改写为新术语，但链接目标（文件名）不变，因此索引行文字与其指向的文件名不完全一致，属预期行为。0106 索引摘要"移除 `formula:` 字典形式"例外保留，因为该行描述的是一个已被移除、从未叫过 `equation` 的独立旧字段（与本次 `formulas`/`equations` 复数块改名是两回事），改写会产生事实错误。`LM_format_1.0.md` §9.1 版本历史里，专门记录"字段从 formula 改名"这件事本身的行（2026-08-08 新增行）同理保留旧名，其余历史行的措辞已更新。
- **`models/**/history/` 下的调试历史快照**（ADR 0141 引入，`.gitignore` 排除，不随仓库发布）——这是刻意保留的改动前原样副本，与"文档里的更改记录"是不同性质的东西，本次改动过程中曾被误改，已核实并恢复原状。
- **`life-matters-home/paper/c_paper_s1_cn.md` 一处讨论已废弃的独立 `formula:` 字符串字段（ADR 0106 移除的旧写法，与本次 `formulas`/`equations` 改名是两个不同字段）的审阅批注区块**——同上，改写会产生事实错误，予以保留。
- **`life-matters-home/process/model_copyright_safety.md` 引用《著作权法》第 5 条原文"通用数表、通用表格和公式"**，以及同文件里泛指"公式作为思想不受版权保护"的表述——这是法律条文引用与知识产权语境下的通用词，与本次改名的 schema 字段无关，且该文件不在 `home` 的改写范围（`paper`/`validation_report.md`）内。
- **`life-matters-home/outreach/lm_slides_v2_global.md` 等三份 slides 里 `formula: "-0.08 * ..."` 示例**——这是 ADR 0106 移除的旧版单数 `formula:` 字典写法的过时示例，本身已与当前 schema 不一致，是独立于本次改名的既有遗留问题，且该文件不在 `home` 的改写范围内，未一并修复。
- **`reference_engine/src/routes/files.py` 里 `f.equation`（原 `f.formula`）读取一个 `Equation`/`Formula` dataclass 上不存在的属性**——这是改名前就存在的既有 bug（该 dataclass 从未定义过 `formula`/`equation` 字段），本次只做了同名改名，未修复该 bug，超出本次改名范围。

## 已知环境限制

本次改动后 `life-matters-reference-engine` 仓库本地 `models/` 目录为空（与本次改名无关的既有环境问题，改名前后同样失败），导致本地 `pytest` 大部分用例因"模型未找到"而无法真正跑通验证；已通过 `git stash` 对照确认该失败与本次改名无关。GUI（`gui/`）与游戏前端（`life-matters-game/game/`）的 `tsc --noEmit` 类型检查均已通过。

## 结果

- 改名：`formulas:` → `equations:`（LM format 字段），四要素简写 var/for/sim/opt → var/equ/sim/opt，助记符 V.F.S.O. → V.E.S.O.
- 覆盖：`life-matters-models`、`life-matters-reference-engine`、`life-matters-game`、`life-matters-home` 四仓库的 YAML、后端、前端、i18n、规范文档、建模指南、README、论文草稿、outreach 材料
- `LM_format_1.0.md` 仍为 v1.0（Draft 状态未发布，不构成对已发布契约的破坏性变更），§9.1 新增一行 changelog 记录本次改动
