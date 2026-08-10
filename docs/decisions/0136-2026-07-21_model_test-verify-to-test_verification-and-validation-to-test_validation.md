# 0136 — `test_verify/` 改名为 `test_verification/`，`models/validation/` 改回 `models/test_validation/`

**日期**：2026-07-21
**状态**：✅ 已接受

---

## 背景

同一天（2026-07-21）稍早的 ADR 0135 把名实不符的 `models/test_validation/`（内容其实是
verification 用的 `valid`/`invalid` fixture）拆分成 `models/test_fixtures/`（fixture）+
新建的 `models/validation/`（真正的文献对标/优化合理性结果），并在决策正文里明确讨论过、
否决了"把 `test_fixtures` 改叫 `test_verification`"的方案——理由是这会跟 life-matters-reference-engine
的 `test_verify/` 撞得更严重。

当天晚些时候，在实际更新 `validation_report.md` 的仿真结果时，用户提出两个独立的命名疑问：
(1) 能否把这次新产出的验证 CSV 放进 `test_fixtures/`；(2) `test_verify/` 能否改名为
`test_verification/`，好和它里面的 `verification_report.md` 对齐，也让"validate/verify"
两侧的目录名更对称。

第(1)个问题很好回答：不行，会破坏 ADR 0135 当天刚刚建立起来的"fixture 数据 vs validation
结果"边界，`test_fixtures/README.md`、`validation_report.md` 开篇都专门写明"两者是两回事、
互不依赖"，混进去等于抹掉这条线。

第(2)个问题触发了本 ADR：**如果 `test_verify/` 改名为 `test_verification/`，ADR 0135 当时
"避免和 test_verify 撞名"这条否决 `test_fixtures→test_verification` 的理由本身也随之改变**
——`test_verify` 这个名字将不复存在，`test_fixtures` 和 `test_verification` 不会撞名。但
`test_fixtures` 这个名字本身已经准确描述内容（可复用的测试 YAML 数据），不需要再改，
ADR 0135 那部分决策不受影响，不用重新讨论。

真正需要重新讨论的是 `models/validation/`：ADR 0135 把它命名为不带 `test_` 前缀的裸名字，
主要是因为当时 `test_validation` 这个名字刚刚因为"名实不符"（装着 fixture 而非 validation
结果）被否决掉，直接复用会造成"同一个名字，此刻却装着完全不同的东西"的混淆——所以退而求其次
选了裸名字。但现在的情况不同：**`models/validation/` 里现在只装真正的 validation 内容**
（`validation_report.md` + `reports/` CSV，ADR 0135 建好之后没有再混入别的东西），"名实不符"
这个否决理由已经不成立了。如果同时把 `test_verify/` 改名为 `test_verification/`，`models/
test_validation/`（对，改回这个名字）与 `test_verification/` 会形成"validation ↔
verification"的整齐配对，比裸名字 `models/validation/` 更贴合两仓库一直在用的"V&V 术语对称
命名"惯例（ADR 0134 最早就是这个动机）。

## 决策

### 1. `test_verify/` → `test_verification/`（life-matters-reference-engine 仓库）

改名，`README.md`/`verification_report.md`/`errors/`/`models/` 子目录内容不变，只改目录名
本身；所有引用该路径的文件同步更新（见下方"影响范围"）。

### 2. `models/validation/` → `models/test_validation/`（life-matters-models 仓库）

**不是重新打开 ADR 0135 关于"要不要把 fixture 和 validation 内容分开放"的判断**——那个判断
（`test_fixtures/` 只装 fixture、真正的 validation 结果单独放一个目录）本次完全保留，两块内容
继续物理分离在两个目录里。本次只改"真正的 validation 结果那个目录该不该带 `test_` 前缀"这一
个更小的问题：ADR 0135 建立时因为名字刚被否决过一次而选了裸名字，现在名实不符的前提已经解除
（该目录自建立起只装过 validation 内容），且 `test_verify→test_verification` 改名让"两个
`test_` 前缀目录对称配对"这个 ADR 0134 就想要的效果第一次真正成立，所以改回来。

`validation_report.md`、`reports/` 子目录内容不变，只改父目录名；所有引用该路径的文件同步
更新。

## 影响范围

**life-matters-reference-engine 仓库**：`pytest.ini`（`testpaths`）、`.gitignore`（路径注释）、
`scripts/check_hardcoded_constants.py`（`EXCLUDE_DIR_PARTS`）、`test_verification/` 目录内
`README.md`/`verification_report.md`/`errors/README.md`/`models/README.md` 及其下 3 个
`test_*.py` 的自引用注释、根 `README.md`、`docs/reference_engine/{DECISIONS.md,cli.md,
evidence/conversion.md,impl.md,mc.md}`、`reference_engine/scripts/validate_banister{,_step_grid}.py`
的模块 docstring、`gui/e2e/specs/run-simulation.spec.ts` 的注释。历史 ADR 0056 正文里的
`test_verify` 引用按"历史记录不做追溯性改写"惯例不动。

**life-matters-models 仓库**：`models/test_fixtures/{README.md,fixture_catalog.md,invalid/README.md}`
及 `valid/`、`invalid/` 下引用 `test_verify/errors/`、`test_verify/verification_report.md`
的 fixture YAML 注释（`test_invalid_*.yaml` 8 个 + `test_valid_banister_v1_*.yaml` 系列 7 个
+ `banister_step_convergence_grid_POINTER.yaml`）；`models/test_validation/validation_report.md`
自身对 `test_verify/verification_report.md` 的 4 处引用，以及对 `models/validation/reports/`
CSV 路径的 3 处引用（本次改名前一天由另一次验证任务新写入，路径需要跟着父目录改名同步更新）。
`models/test_fixtures/README.md` 里"和 `models/validation/`、`test_verification/` 的关系"一节
额外补充了一句说明：本目录曾经短暂用过 `test_validation` 这个名字（ADR 0135 拆分前），当天下午
本 ADR 又把新建的 validation 结果目录改回同一个名字——两次是不同的目录，只是先后用过同名字，
避免读者误以为改名被撤销。

**life-matters-home**：`process/model_validation_workflow.md` 的 `models/validation/`、
`test_verify/` 路径引用；`tasks/2026-07-21_report_validation.md`（同一天更早的验证任务记录，
按"不追溯改写"原则保留原始路径描述，改为末尾追加一条说明当前路径已变更）。历史归档文件
`tasks/archive/2026-07-17_task_docs-code-drift-engine-cli-gui.md` 按惯例不动。

**历史 ADR 正文不动**：`docs/decisions/README.md` 里 0134、0135 两行索引摘要描述的是这两个
ADR 当时的决策内容，本次不追溯改写，仅新增本 ADR 的索引行。

## 结果

- 改名：`test_verify/` → `test_verification/`（life-matters-reference-engine）
- 改名：`models/validation/` → `models/test_validation/`（life-matters-models）
- 同步：两仓库 + life-matters-home 共约 35 处路径引用（详见上）
- ADR 0135 的核心判断（fixture 与 validation 结果物理分离到两个目录）保持不变，本次只调整
  validation 结果目录是否带 `test_` 前缀这一项

## 未决

- `docs/reference_engine/decisions/0056-2026-05-04_project_three-tier-validation-framework.md`
  正文里的 `test_verify` 引用未同步（历史记录惯例不动），下次有读者从该 ADR 点进去时可能需要
  自行心算一次改名映射，不阻塞本次改名。
