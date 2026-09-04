# Design Decision Records

每个文件记录一个设计决策，格式参考 [ADR](https://adr.github.io/)。

> **编号规则（2026-07-15 起）**：本仓库与 `life-matters-reference-engine` 仓库（`docs/reference_engine/decisions/`）
> 共用同一个全局编号序列，不再各自独立计数——新建 ADR 前先看两个仓库各自最新编号，取
> 两者最大值 + 1。2026-07-15 之前两边各自独立计数，曾出现 0128/0129/0130 同号但内容不同
> 的历史遗留（本仓库原 0128/0129 已改名为 0131/0132，`life-matters-reference-engine` 的 0128/0129/0130
> 维持不变，未追溯改动）。

## 索引

| # | 标题 | 状态 | 日期 |
|---|------|------|------|
| [0001](0001-simulator-two-column-layout.md) | 仿真器双列布局与多图方案 | ✅ 已实施 | 2026-03-28 |
| [0002](0002-left-panel-accordion.md) | 左侧面板 Accordion（VSCode 风格）| ✅ 已实施 | 2026-04-01 |
| [0003](0003-toolbar-flow-validate-mode-run.md) | 工具栏操作流：验证→模式→运行 | ✅ 已实施 | 2026-04-01 |
| [0004](0004-i18n-locale-files.md) | 多语言方案：JSON locale 文件 | ✅ 已实施 | 2026-04-01 |
| [0005](0005-localstorage-persistence.md) | 客户端状态持久化：localStorage | ✅ 已实施 | 2026-04-02 |
| [0006](0006-game-story-folder-format.md) | Game Story 文件夹格式与前端 Loader | ✅ 已实施 | 2026-04-04 |
| [0007](0007-unified-titlebar.md) | Sim / Game 顶栏统一 | ✅ 已实施 | 2026-04-04 |
| [0010](0010-game-hearthstone-layout.md) | Game 界面 Hearthstone 式布局重构 | ✅ 已实施 | 2026-04-04 |
| [0011](0011-game-6row-unified-cards.md) | Game 6行对称布局 + 统一卡牌尺寸 + 对方手牌机制 | ✅ 已实施 | 2026-04-04 |
| [0012](0012-neutral-theme-unified-colors.md) | Game 中性主题 + 双应用色彩 Token 统一 | ✅ 已实施 | 2026-04-05 |
| [0013](0013-relative-font-scale-selector.md) | 相对字号系统 + 字号选择器 | ✅ 已实施 | 2026-04-05 |
| [0015](0015-card-hand-mechanics.md) | 卡牌手牌机制：无回收 + 双重惩罚 + 永久牌 | ✅ 已实施 | 2026-04-05 |
| [0016](0016-game-terminology-and-fate-rule.md) | 游戏术语统一 + 命运机制规则 | ✅ 术语已实施；命运 AI 待定 | 2026-04-05 |
| [0017](0017-card-backs-story-assets.md) | 卡背图片 + Story 静态资产加载 | ✅ 已实施 | 2026-04-05 |
| [0018](0018-background-music-player.md) | 背景音乐 + MusicBar 播放控制条 | ✅ 已实施 | 2026-04-05 |
| [0019](0019-disclaimer-placement-and-content.md) | 免责声明：位置、内容与呈现规范 | ✅ 已实施 | 2026-04-07 |
| [0020](0020-app-naming-life-matters.md) | 应用命名：统一为 Life Matters，中文副名仅在 About 中显示 | ✅ 已实施 | 2026-04-07 |
| [0021](0021-about-contact-info-github-only.md) | About 弹窗联系信息：仅保留 GitHub，去除邮件与主页 | ✅ 已实施 | 2026-04-09 |
| [0022](0022-models-three-level-taxonomy.md) | Models 三层分类体系（medical/social → 学科 → 细分） | ✅ 已实施 | 2026-04-12 |
| [0023](0023-models-runnable-from-gui.md) | Models 可在 GUI 文件树中直接运行 + standalone 约定 | ✅ 已实施 | 2026-04-12 |
| [0024](0024-asteval-rebuild-over-clear.md) | asteval Interpreter 重建而非 symtable.clear() | ✅ 已实施 | 2026-04-12 |
| [0025](0025-story-editor-four-tab-layout.md) | StoryEditor 转换器四标签平铺 + 多条件结局设计器 + 通用卡牌库 | ✅ 已实施 | 2026-04-13 |
| [0026](0026-simulator-report-tab.md) | 仿真器报告标签：左列勾选 + 右侧折叠预览 + MD 导出 | ✅ 已实施；DOCX 待续 | 2026-04-13 |
| [0027](0027-hand-discard-and-card-backs.md) | 手牌弃置机制与界面重组 | ✅ 已实施 | 2026-04-15 |
| [0028](0028-card-drag-drop-zones.md) | 卡牌区域拖拽系统 | ✅ 已实施 | 2026-04-15 |
| [0029](0029-responsive-layout-and-deck-config.md) | 响应式游戏布局、draw_per_turn 与 copies 牌组配置 | ✅ 已实施 | 2026-04-15 |
| [0031](0031-scenario-to-story-semi-auto-generation.md) | Scenario-to-Story 半自动生成工作流 | ✅ 已实施 | 2026-04-15 |
| [0033](0033-2026-04-14_game_弃牌机制设计决策.md) | 弃牌机制设计（暂存区 + 双区布局） | ✅ 已实施 | 2026-04-14 |
| [0034](0034-2026-04-15_game_手牌数量设计.md) | 手牌数量设计（hand_size + draw_per_turn） | ✅ 已定稿 | 2026-04-15 |
| [0035](0035-2026-04-19_sim_报告生成设计.md) | Sim 报告生成：章节顺序、含义列、CSV 分离、图表内嵌 | ✅ 已实施 | 2026-04-19 |
| [0036](0036-2026-04-19_game_动画与结算时序设计.md) | Game 动画与结算时序设计 | ✅ 已实施 | 2026-04-19 |
| [0037](0037-2026-04-19_game_i18n多语言覆盖层设计.md) | Game i18n 多语言覆盖层设计 | ✅ 已实施 | 2026-04-19 |
| [0038](0038-2026-04-20_sim_regimen-k4-input-scheduling.md) | Regimen K×4 输入调度：时刻/摄入量/执行日/有效期 | ✅ 已实施 | 2026-04-20 |
| [0040](0040-2026-04-22_sim_医学证据类型与变量映射.md) | Sim 医学证据类型与变量映射（evidence 8 子类型） | ✅ 已实施 | 2026-04-22 |
| [0041](0041-2026-04-22_project_命名规范下划线优先.md) | 项目命名规范：snake_case 下划线优先 | ✅ 已实施 | 2026-04-22 |
| [0042](0042-2026-04-23_project_mod-to-model-rename.md) | mod → model 全面重命名；保留 sim_xxx 不改 | ✅ 已实施 | 2026-04-23 |
| [0043](0043-2026-04-25_game_battlefield-tension-framework.md) | 战场张力框架：battle_progress/danger_accumulation 归 Game-native；origin 字段；命运牌模式 | ✅ 已实施 | 2026-04-25 |
| [0044](0044-2026-04-30_sim_schedule作为simulation-input子类型.md) | `simulation.schedules`：时间驱动输入归属 `simulation` 块；pulse 模式；离散 input 不写零值点规则 | ✅ 已实施 | 2026-04-30 |
| [0045](0045-2026-04-30_sim_MC概率仿真与随机参数架构.md) | MC 概率仿真架构：parameter 分布表达式、多 run 引擎、半透明曲线渲染、Opt 内环均值评估、种子管理 | ✅ 已实施 | 2026-04-30 |
| [0046](0046-2026-04-30_sim_步长设计-step_size元数据与step公式符号.md) | 步长最终方案：`metadata.step_size.{value,unit}`；方程用 `step`；simulation 去掉 step/step_unit | ✅ 已实施 | 2026-04-30 |
| [0049](0049-2026-05-02_sim_Optimizer异步Job系统设计.md) | Optimizer 异步 Job 系统设计 | ✅ 已实施 | 2026-05-02 |
| [0050](0050-2026-05-04_sim_InputEvent扁平化与交互状态颜色规则.md) | InputEvent 扁平化与交互状态颜色规则 | ✅ 已实施 | 2026-05-04 |
| [0052](0052-2026-05-04_sim_schedule格式统一与opt-regimen支持.md) | Schedule 格式统一（扁平列表）& optimization.regimen 支持 | ✅ 已实施 | 2026-05-04 |
| [0053](0053-2026-05-03_sim_date_range调度字段与YAML-schedule优先级修复.md) | `date_range` 日期区间字段；YAML Schedule 优先于 GUI Regimen | ✅ 已实施 | 2026-05-03 |
| [0054](0054-2026-05-04_sim_unified-apply-regimens.md) | 仿真/优化 Regimen 执行函数统一：删除 `_apply_regimen_events` | ✅ 已实施 | 2026-05-04 |
| [0055](0055-2026-05-04_project_docs-go-public-private-split.md) | `docs/` 公开发布 / `go/` 内部不发布 分界规则 | ✅ 已实施 | 2026-05-04 |
| [0056](0056-2026-05-04_project_three-tier-validation-framework.md) | 三层验证框架：层1数值精度 / 层2文献对标 / 层3优化合理性 | ✅ 框架已实施，脚本待写 | 2026-05-04 |
| [0057](0057-2026-05-04_project_models-paper-directory.md) | `models/published/paper1-3/` 论文专用场景目录 | ✅ 已实施 | 2026-05-04 |
| [0062](0062-2026-05-06_project_models-directory-rename.md) | models 目录重命名规范（source/ → in_process/，scenarios/ → published/） | ✅ 已实施 | 2026-05-06 |
| [0063](0063-2026-05-07_sim_resolved-imports-and-output-selection.md) | Resolved imports 与仿真输出选择规则 | ✅ 已实施 | 2026-05-07 |
| [0064](0064-2026-05-07_project-edit-refresh-run-snapshot.md) | 编辑态刷新源文件，运行态固定快照 | ✅ 已实施 | 2026-05-07 |
| [0065](0065-2026-05-08_sim_structured-description.md) | metadata.description 支持结构化与自由文本 | ✅ 已实施 | 2026-05-08 |
| [0066](0066-2026-05-08_sim-simulator-decomposition-and-result-workspaces.md) | Simulator 拆分与 Sim/Opt 结果工作区 | ✅ 已实施；OPT/SIM 分离重构待续 | 2026-05-08 |
| [0067](0067-2026-05-15_sim_optimizer-algo-preset-slider-ui.md) | 优化算法参数预设（快速/标准/精细）与滑动条联动 UI | ✅ 已实施 | 2026-05-15 |
| [0068](0068-2026-05-15_sim_formula-precompile-to-python-function.md) | 方程预编译：asteval 运行时解析 → 加载时生成 Python 函数 | ✅ 已实施 | 2026-05-15 |
| [0069](0069-2026-05-15_sim_run-history-auto-archive.md) | 运行历史自动存档：sim/opt 完成后自动存档，历史抽屉加载/删除 | ✅ 已实施 | 2026-05-15 |
| [0070](0070-2026-05-15_sim_asteval-as-safety-sandbox-constraint.md) | asteval 作为方程安全沙箱：禁止用 Python eval() 直接替代（补录核心约束） | ⭐⭐ 核心约束 | 2026-05-15 |
| [0071](0071-2026-05-15_sim_ui-rounded-cards-settings-gear-drag-sort.md) | 全局圆角卡片面板 + 设置齿轮 Popover（字号/语言）+ 区块拖拽排序 | ✅ 已实施 | 2026-05-15 |
| [0072](0072-2026-05-15_project_gui-only-no-cli.md) | GUI-only：放弃 CLI 作为正式接口（补录核心约束） | ⭐⭐ 核心约束 | 2026-05-15 |
| [0073](0073-2026-05-16_sim_multi-plan-simulation.md) | 多方案仿真：Plan 术语、数据模型、MC 逐方案独立运行 | 待实现 | 2026-05-16 |
| [0074](0074-2026-05-16_sim_gui-working-state-priority.md) | GUI Working State 优先级高于 YAML Schedule（反转 ADR 0053 GUI 变量规则） | 待实现 | 2026-05-16 |
| [0074](0074-2026-05-16_sim_single-tab-group-and-builder-tab.md) | 单层标签组导航：动态 Builder Tab + 统一目录树双模式 | ✅ 已实施 | 2026-05-16 |
| [0075](0075-2026-05-17_model_remove-type-standalone-fields.md) | 移除 type/standalone 顶层字段 | ✅ 已实施 | 2026-05-17 |
| [0076](0076-2026-05-17_sim_yaml-simulation-plans.md) | YAML simulation.plans 预置多方案 | ✅ 已实施 | 2026-05-17 |
| [0077](0077-2026-05-17_sim_session-model-import.md) | Session 模型导入：原子上传（UUID + 内联解析 + 即时删除）→ localStorage | ✅ 已实施（2026-05-18 重写） | 2026-05-17 |
| [0078](0078-2026-05-18_project_scs-mode-design.md) | SCS_MODE：云端部署写操作保护 + 前端行为适配 + 合并→session model | ✅ 已实施 | 2026-05-18 |
| [0079](0079-2026-05-18_sim_workspace-layout-4-6-split.md) | Sim/Opt 工作区布局：4:6 百分比分列 | ✅ 已实施 | 2026-05-18 |
| [0080](0080-2026-05-20_sim_optimizer-schedule-tiers-T2T3T4.md) | 优化器调度粒度分层设计（T2/T3/T4：时间窗 / 星期模式 / 起始日） | ✅ 已实施 | 2026-05-20 |
| [0081](0081-2026-05-20_sim_lm-score-health-span-metric.md) | LM Score：Life Matters 健康时长核心指标（可恢复 / 不可逆双模式） | ✅ 已实施 | 2026-05-20 |
| [0082](0082-2026-05-21_sim_lock-unlock-refresh-behavior.md) | 锁定/解锁/切换模型的会话状态设计（两层分离架构）D3–D5 由 0085 取代 | ✅ 已实施（部分取代） | 2026-05-21 |
| [0083](0083-2026-05-22_sim_optimizer-evaluation-time-window.md) | Optimizer 评估时间窗独立配置（start_date / end_date / step_size） | ✅ 已实施 | 2026-05-22 |
| [0084](0084-2026-05-23_sim_sim-opt-separation.md) | Sim / Opt 完全分离：InputEvent / OptInput 独立类型；OptSetupTab 新组件 | ✅ 已实施 | 2026-05-23 |
| [0085](0085-2026-05-25_sim_remove-lock-free-switch-running-indicator.md) | 移除锁机制、自由切换模型、双箭头运行指示器、云端运行拦截 | ✅ 已实施 | 2026-05-25 |
| [0086](0086-2026-05-26_project_lmml-rename-from-lmf.md) | 格式命名：LMF → LMML（Life Matters Model Language） | ✅ 已实施 | 2026-05-26 |
| [0087](0087-2026-05-27_sim_schedules-plans-coexistence.md) | `simulation.schedules` 与 `plans` 共存语义：plans 优先，papers/ 禁止混用 | ✅ 已实施 | 2026-05-27 |
| [0088](0088-2026-05-28_sim_optimizer-schedules-unified-format.md) | optimization.schedules 统一格式：决策变量与固定背景量合并列表 | ✅ 已接受 | 2026-05-28 |
| [0089](0089-2026-05-30_sim_session-refactor-warm-start-dirty-active-model.md) | Session 精化：useSession 分离、userEdited 追踪、Warm-start Dirty 检测 | ✅ 已实施 | 2026-05-30 |
| [0090](0090-2026-05-31_sim_remove-second-step-unit.md) | 移除 second 步长单位，统一 minute/hour/day | ✅ 已接受 | 2026-05-31 |
| [0091](0091-2026-06-01_project_cli-batch-tool.md) | `sim_cli/`：批量仿真 CLI 工具 | ✅ 已实施 | 2026-06-01 |
| [0092](0092-2026-06-05_model_input-variable-bare-unit-rule.md) | `type: input` 单位规范：裸单位（事件量），禁止速率单位（/day 等） | ✅ 已实施 | 2026-06-05 |
| [0093](0093-2026-06-05_sim_runtime-log-panel.md) | Sim/Opt 运行时日志面板：内容分层（模型信息、NaN/bounds 警告、完成统计）与实现 | ✅ 已实施 | 2026-06-05 |
| [0096](0096-2026-06-06_model_filename-quality-markers.md) | 模型文件名质量标记约定（_nosim / _noopt / _noref） | ⚪ 已被 0101 取代 | 2026-06-06 |
| [0097](0097-2026-06-08_model_description-3-fields.md) | papers/ 模型 description 简化为三字段（brief / problem / method） | ✅ 已实施 | 2026-06-08 |
| [0098](0098-2026-06-11_sim_optimizer-schedule-sustained-mode.md) | optimization.schedules 新增 mode: sustained（子日步长持续输入） | ✅ 已实施 | 2026-06-11 |
| [0099](0099-2026-06-11_sim_sustained-value-step-invariance.md) | sustained 模式 value 语义修正：窗口总量 / N_steps（step-size 不变性） | ✅ 已实施 | 2026-06-11 |
| [0100](0100-2026-06-11_sim_unify-pulse-sustained-time-interval.md) | 统一 pulse/sustained 为时间区间 [start,end)；GUI 取消 full day/time/sustained 三态 | 🟡 部分实施（papers 术语已补充说明，未做全文改写） | 2026-06-11 |
| [0101](0101-2026-06-14_model_hold-suffix-todo-field.md) | 文件名质量标记统一为 `_HOLD` + `metadata.todo` 任务列表（取代 0096） | ⚪ 文件名部分被 0120 取代 | 2026-06-14 |
| [0102](0102-2026-06-14_model_formula-priority-execution-order.md) | 澄清 equation `priority` 执行顺序（数值越大越先执行）与同 step 内顺序写入语义 | ✅ 已实施 | 2026-06-14 |
| [0103](0103-2026-06-14_model_metadata-log-field.md) | 新增 `metadata.log`：模型内改进历史记录 | ✅ 已实施 | 2026-06-14 |
| [0104](0104-2026-06-16_model_step-unit-per-formula-and-sim-step-size.md) | 步长设计重构：per-equation `step_unit` + `simulation.step_size`（取代 `metadata.step_size`） | ✅ 已实施 | 2026-06-16 |
| [0105](0105-2026-06-16_model_step-unit-conditional-and-deprecate-dt.md) | `step_unit` 改为条件必填 + 废弃 `dt`/`step_size` 动力学符号 | ✅ 已实施 | 2026-06-16 |
| [0107](0107-2026-06-16_model_output-variables-import-overwrite.md) | `output_variables` / `output_types` import 行为统一为覆盖（取代并集） | ✅ 已实施 | 2026-06-16 |
| [0108](0108-2026-06-21_project_lm-icon-design.md) | LM 品牌图标：黑白对半心形，无边框；sim 端配色为品牌绿 | ✅ 已实施 | 2026-06-21 |
| [0120](0120-2026-06-23_model_drop-hold-filename-suffix.md) | 废除 `_HOLD` 文件名后缀，状态判定仅看 `metadata.todo`（部分取代 0101） | 🟢 已实施 | 2026-06-23 |
| [0121](0121-2026-06-25_model_step-unit-parameter-conversion-linear-vs-root.md) | 跨 step_unit 参数换算：线性除法（状态无关通量项）vs 开根（自指数衰减项） | ✅ 已采纳 | 2026-06-25 |
| [0125](0125-2026-07-05_model_test-valid-invalid-split.md) | `models/test/` 拆分为 `valid/`+`invalid/`：新增 11 个错误检测 fixture，覆盖循环 import/evidence 冲突/方程未声明变量等校验 | ✅ 已接受 | 2026-07-05 |
| [0126](0126-2026-07-09_model_regimen-semantics-scope-decision.md) | regimen 语义完备性讨论范围拍板：pulse-decay 是模型完备性非引擎问题；覆盖/累加不改引擎，只用 baseline+增量惯例改具体文件；sustained 判断规则文档收尾 | ✅ 已接受 | 2026-07-09 |
| [0127](0127-2026-07-09_model_input-unified-sustained-window-defaults.md) | input 变量统一为 sustained，不再有独立 pulse 模式；窗宽默认规则：都不写=全天，只写起点=单step，都写=显式区间 | ✅ 已实施 | 2026-07-09 |
| _（0128–0130 保留给 life-matters-reference-engine 仓库 `docs/reference_engine/decisions/` 的引擎侧 ADR，见下方说明——两仓库自 2026-07-15 起共用一个编号序列）_ | | | |
| [0131](0131-2026-07-13_model_sustained-value-per-day-not-per-span.md) | sustained value 语义修正：每个匹配日独立满额，取代 0099 的"整跨度总量"（同批取代 0126 第3条） | ✅ 已实施 | 2026-07-13 |
| [0132](0132-2026-07-14_model_sustained-delivery-total-vs-level.md) | sustained regimen 新增 `delivery: total\|level`，区分"总量摊分"（训练负荷类）与"恒定水平"（睡眠时长类） | ✅ 已实施 | 2026-07-14 |
| [0133](0133-2026-07-15_model_delivery-judgment-principle-and-day-lumped-map.md) | `delivery` 判断规则（系数/瞬时读取 vs 累加）+ "day-lumped map" 反模式识别：sleep_hours 类变量靠 `step_unit:day`+`delivery:level` 打补丁，非真正逐步可积，修复留给独立 task | ✅ 判断原则已定；反模式修复未实施 | 2026-07-15 |
| [0134](0134-2026-07-15_model_test-renamed-to-test_validation-and-valid-prefix.md) | `models/test/` 改名为 `models/test_validation/`（与 life-matters-reference-engine `tests/`→`test_verify/` 对称）；`valid/` 下 35 个文件加 `test_valid_` 前缀，`invalid/` 保持既有 `test_invalid_*` 命名 | ✅ 已接受 | 2026-07-15 |
| [0135](0135-2026-07-21_model_test-validation-split-into-test_fixtures-and-validation.md) | 0134 的"validation"命名名实不符（`valid`/`invalid` fixture 实际测的是 verify）：`models/test_validation/` 拆分为 `models/test_fixtures/`（引擎 fixture，`valid`/`invalid` 不变）+ 新建 `models/validation/`（真正的文献对标结果，与 fixture 无关） | ✅ 已接受 | 2026-07-21 |
| [0136](0136-2026-07-21_model_test-verify-to-test_verification-and-validation-to-test_validation.md) | life-matters-reference-engine `test_verify/` 改名 `test_verification/`（呼应内含的 `verification_report.md`）；`models/validation/` 因此改回 `models/test_validation/`（0135 否决该名字是因为当时名实不符，现内容已纯净、否决理由不再成立，与 `test_verification/` 对称配对） | ✅ 已接受 | 2026-07-21 |
| [0137](0137-2026-07-24_model_evidence-merged-into-variables.md) | 顶层 `evidence:` 节并入 `variables:`：`type` 保持 3 值不变，新增正交字段 `evidence_type` 表达 8 种文献效应量子类型（否决把子类型编码进 `type` 本身导致 3→11 值膨胀的方案），取代 ADR 0040 的顶层节设计 | ✅ 已接受 | 2026-07-24 |
| [0138](0138-2026-07-30_project_model-inclusion-criteria-and-coverage-inventory.md) | 模型纳入标准（var/equ/sim/opt 四问粗筛）与学科覆盖盘点表；顺带修复 `docs/LM_format_1.0.md` 多处滞后于当前实现的内容 | ✅ 已接受 | 2026-07-30 |
| [0139](0139-2026-07-30_project_ai-generated-content-and-author-responsibility-boundary.md) | AI 生成内容声明与作者/模型库责任边界：作者负责格式规范/引擎/S1 精选案例，模型库其余内容为 AI 辅助生成、邀请专家核对的开放资源 | ✅ 已接受 | 2026-07-30 |
| _（0140 保留给 life-matters-reference-engine 仓库）_ | | | |
| [0141](0141-2026-08-05_model_method-field-restored-and-debug-history-folder.md) | `papers/` description 恢复 `method` 字段（说明模型融合了哪些机制，体现耦合而非堆叠）；反复调试产生的历史版本移入同目录 `history/`（gitignored），主文件只留最终版本 | ✅ 已实施 | 2026-08-05 |
| [0142](0142-2026-08-06_model_description-list-structure-and-references-annotations.md) | `papers/` description 的 `problem`/`method`/`result` 并列事实改用列表；来源文献的具体贡献从 `problem` 搬进 `metadata.references` 的可选 `{citation, description}` 对象；引用统一句末作者年份夹注，不用数字编号 | ✅ 已实施 | 2026-08-06 |
| [0143](0143-2026-08-06_model_description-migration-no-longer-deferred.md) | 废止 0142"非目标"里"不追溯批量重写"的表述：此后任何原因编辑 `papers/` 模型都应顺带迁移 description/references 到 0142 定义的格式；references 的 `{citation, description}` 写法从"可选"升级为推荐默认 | ✅ 已实施 | 2026-08-06 |
| [0144](0144-2026-08-08_project_formulas-renamed-to-equations-and-veso-mnemonic.md) | LM format 顶层 `formulas:` 字段更名为 `equations:`，与规范正文已在用的 differential/dynamic equation 表述对齐；四要素简写 var/for/sim/opt 改为 var/equ/sim/opt，助记符 V.F.S.O. 改为可连读的 V.E.S.O.；规范仍处 Draft 未冻结发布，不构成破坏性变更，`LM_format_1.0.md` 保持 v1.0 | ✅ 已实施 | 2026-08-08 |
| [0145](0145-2026-08-15_project_lm-modeling-agent-four-step-pipeline.md) | LM 建模协作固化为四步流水线（灵感/起草/诊断/调整，加收尾报告约定），完整指令存本仓库 `agents/`，reference_engine 端只放 `.claude/agents/` 薄封装 stub；权限边界按步骤收敛，仅 Step 3.2 能编辑模型副本且绝不碰正式文件；任务报告存内部 `life-matters-home/agent_reports/` | ✅ 已实施 | 2026-08-15 |
| _（0146-0149 见 reference-engine 仓库索引，或保留候选未创建，详见对应仓库记录）_ | | | |
| [0150](0150-2026-08-20_model_ratings-scale-changed-to-0-1-and-veso-scoring-fields.md) | `metadata.ratings` 评分尺度从 1-5 整数改为 0-1 连续量表（五锚点 0/0.25/0.5/0.75/1）；字段按技术类（VESO：`variable`/`equation`/`simulation`/`optimization`）与非技术类（`importance`/`innovation`/`confidence`）两分法重组精简，原六个通用+三个类型专属字段合并为四个通用技术类+三个非技术类；适用范围扩展到 `models/plan/`；agent 文档改为运行时读取 `ratings.md` 而非各自复制量表定义；不追溯迁移 S2/S3/S4/scenarios/references 下已有评分 | ✅ 已实施（规范本身，模型文件迁移为后续任务） | 2026-08-20 |
| [0151](0151-2026-08-23_model_references-top-level-block-and-mivesor-order.md) | `references:` 从 `metadata.references` 提升为顶层块，与 `imports` 同级；顶层键顺序统一为 M I V E S O R；顺带修复 `LM_format_1.0.md` 里 ADR 0104/0105（`metadata.step_size`→`simulation.step_size`+逐公式`step_unit`）从未同步进规范文档的历史遗留；引擎无需改动（`references`/`checksum` 均从未被 loader/validator 读取）；现有模型文件迁移为后续任务，见 `skill_agent/lm-model-calibration.md` | ✅ 已实施（规范本身，模型文件迁移为后续任务） | 2026-08-23 |
| [0152](0152-2026-08-25_project_optimizer-field-renamed-to-optimization.md) | LM format 顶层 `optimizer:` 字段更名为 `optimization:`，补齐 ADR 0144 已定的 VESO 助记符（Variables, Equations, Simulation, **Optimization**）与实际字段名之间的落差，`simulation:` 装的是仿真实验配置而非"仿真器"工具本身，`optimizer:` 同理；只改外部可见的字段名（YAML、规范、论文、GUI 文案、API JSON 字段），engine 内部模块文件名/类名/属性名/API 路由路径保留 `optimizer` 不变；规范仍处 Draft 未冻结发布，不构成破坏性变更，`LM_format_1.0.md` 保持 v1.0 | ✅ 已实施 | 2026-08-25 |
