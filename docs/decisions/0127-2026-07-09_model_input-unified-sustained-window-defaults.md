# 0127 — input 变量统一为 sustained（不再有独立的 pulse 模式），窗宽按显式规则默认

**日期**：2026-07-09
**状态**：✅ 已实施
**类别**：仿真引擎 / regimen 语义

---

## 背景

ADR 0126 记录了 regimen 语义完备性讨论的三处范围拍板，其中问题2（同一变量多条 regimen
覆盖/累加）决定不改引擎、只用 baseline+增量惯例改具体文件；讨论过程中确认了两个互补的
备选方向（内部设计待办记录方向2）：一个是给 `type: input` 增加显式白名单（未采纳为本次范围，见 ADR 0126）；另一个是
retire "pulse" 作为独立命名概念，把它并入 sustained（`N_steps=1` 是同一条 ADR 0099 规则的
特例，这个等价关系在 ADR 0099/0100 就已成立），窗宽由建模者显式声明或按规则默认。

本 ADR 记录第二个方向的落地决策：**不再有独立的 pulse 模式，所有 `type: input` 变量统一按
sustained 处理，区别只在窗宽**。

讨论中曾质疑"窗宽统一"是否会牺牲已验证的药代动力学（PK）曲线形状——实测
`nicotine_plasma`（20支/天，`eta_abs=1.8`，`k_nic=2.77/day`，t½=6h）三种窗宽：

| 窗宽 | 峰值 | 谷值 | 日均 |
|---|---|---|---|
| 单点脉冲（旧默认） | 38.0 | 2.3 | 13.0 |
| 醒着的16小时铺开 | 17.7 | 6.6 | 13.0 |
| 摊平全天24h | 13.0 | 13.0 | 13.0 |

结论：日均值三种情况完全一致（ADR 0099 总量不随分摊方式变化的性质严格成立），峰谷振荡在
合理窗宽下依然存在（不是"抹平成常量"）；窗宽该选多宽是建模者自己的科研判断（真实抽烟行为
也不是单点集中、也不是全天匀速），步长/窗宽选得不合理导致失真是建模者的科研设计责任，不是
引擎该防的事——这是"自然保障"，不需要额外机制去防。已验证的 PK 模型（`nicotine_plasma`/
`thiazide_level`/`allopurinol_level`）继续选窄窗（1 个 step）即可，数值零改动。

## 决策

### 1. 废除"pulse 默认收缩到某个约定时刻"的隐式行为

原引擎行为（`schedule_runner.py::_normalize_time_interval`、`loader.py`、
`optimizer_engine.py` 三处独立实现、互不一致）：`time_start`/`time_end` 缺省时分别退化为
`'08:00'`（`schedule_runner.py`/`optimizer_engine.py`）或 `'00:00'`（`loader.py`）——一个
建模者从未声明过的、纯属引擎实现细节的"约定时刻"。这正是 ADR 0126 讨论中指出的问题：
day-rate 输入（如 `caloric_deficit`）被强行套进"某个时刻触发"的框架，制造了本不该存在的
"该阶段有没有生效"的歧义。

### 2. 新的窗宽默认规则（`resolve_time_interval`，`schedule_runner.py`）

| 写法 | 生效窗口 | 适用场景 |
|---|---|---|
| `time_start`/`time_end` 都不写 | 全天 `["00:00","24:00")` | day-rate 输入，没有自然的"触发时刻" |
| 只写 `time_start` | `time_end = time_start`（单 step 窗口） | 离散事件（进食、给药），数值上等价于旧的 pulse |
| 两个都写 | 显式区间 | 子日步长模型的"持续强度"类输入 |

三种写法共用同一套引擎机制（ADR 0099 的 `value/N_steps` 累计规则），不是三个分支——这也是
"统一"的字面含义：不存在"pulse 分支"和"sustained 分支"两套独立代码路径，只有一个函数
`resolve_time_interval` 决定窗宽多大。

### 3. 代码改动

新增 `schedule_runner.py::resolve_time_interval(entry) -> (time_start, time_end)`，实现上面
的三段默认规则，替换原来分散在三个文件、彼此不一致的 `.get('time_start', '08:00'/'00:00')`
写法：

- `schedule_runner.py`：`precompute_sustained_divisors`/`apply_schedules` 内部调用改用
  新函数（原 `_normalize_time_interval` 改名并重写为公开函数）。
- `model_structure/loader.py::_parse_schedule_entries`（`simulation.plans[*].regimens`
  解析路径）：改为调用 `resolve_time_interval`。
- `optimizer_engine.py`（`optimizer.startpoint.regimens` 解析路径，`fixed_events_map`
  构建 + `_build_regimen_events` 的 `d0` 解码，共 3 处）：改为调用 `resolve_time_interval`。

四处默认逻辑合并为一处，消除了原本 `loader.py`（'00:00'）与 `optimizer_engine.py`（'08:00'）
两条路径互不一致的隐藏 bug 面。

### 4. 术语：文档不再使用"pulse"作为独立命名的模式

`docs/model.md` 全文改写：不再把"pulse"和"sustained"并列为两种 input 类型；统一表述为
"sustained，窗宽可窄到 1 个 step"。历史文档（`mode: sustained` ADR 0098 旧格式、`optimize.time`
旧字段等已标注废弃的段落）保留原始表述，供迁移旧 YAML 参考，不追溯改写。

## 结果

- 已实施：`reference_engine/src/schedule_runner.py`（新增 `resolve_time_interval`，替换
  `_normalize_time_interval`）、`reference_engine/src/model_structure/loader.py`、
  `reference_engine/src/optimizer_engine.py`（三处默认逻辑改用新函数）。
- 已验证：`tests/` 全量 23 个 pytest 通过（含 `test_schedule_runner.py`/
  `test_sim_cli_consistency.py`/`tests/errors/` 全部错误检测 fixture）；CLI 冒烟测试
  （`hypertension_gout_sim.yaml --sim`，5 plan × 5 MC run）跑通，无回归。
- 已更新：`docs/model.md` 多处改写（regimens 字段说明新增"窗宽默认规则"表、"统一区间表示"
  章节改写、"窗宽是同一范畴的量"章节改写），不再把 pulse 作为独立模式介绍。
- **全库验证（2026-07-09 追加，纠正下面这条曾经的错误陈述）**：曾以为"完全不写时间"这个
  写法此前从未被合法使用过，**核实后是错的**——实测扫描全部 205 个 `models/**/*.yaml`
  文件，`optimizer.startpoint.regimens` 里有 **18 处**真实实例（`ckd_protein_opt_*`×4 个
  `dietary_protein`、`infant_breastfeeding_opt_*`×3 个文件各 2 处 `breast_milk`、
  `bergman_glucose_opt_*`×3 个 `exercise_met_min`、`masld_insulin_opt_*`×3 个
  `exercise_met_min`、`test_opt_t2.yaml` 2 处），全部同一结构：T2（`optimize.time_start`
  1 维搜索）激活、base 条目完全不写 `time_start`/`time_end`。逐一验证（真实跑一次
  `--opt`，pop=4/gen=1）确认**行为不变**：这类条目最终的 `time_end` 由
  `_shift_time(搜索到的 time_start, _width_min)` 算出，旧默认下 `_width_min=0`
  （pulse，宽度0）、新默认下 `_width_min=1440`（全天），但 `_shift_time` 对 1440 分钟
  取模 `% 1440`，两者结果都是"搜索到的时刻，宽度0"——这是 24 小时对模运算的巧合，不是
  设计保证，记录在案供以后排查同类问题时参考。除此之外，全库 205 个文件（papers 58 +
  test 45 + scenarios 27 + references 75）真实跑一遍 `--sim`/`--opt`
  （对 `--opt` 用极小 pop/gen 压缩验证时间），**ADR 0127 造成的失败为 0**——发现的 17
  处失败全部是预存、与本次改动无关的问题（`references/medical` 断链 import + 内容 bug 14
  处、`scenarios/` 两个文件的旧版 optimizer schema 从未迁移、`test/valid` 一个孤立
  fixture 缺 `optimize:` 块），已记录到内部任务
  `2026-07-09_task_reference-library-broken-imports-audit.md`，
  不在本 ADR 范围内处理。

## 未决

- ADR 0126 已记录的方向1（`consumed_by` 白名单，解决"读错物理量"类 bug，如 `bp_dynagmics`
  该读 `body_weight` 却读了 `caloric_deficit`）与本 ADR 是互补机制，不是本 ADR 的一部分，
  仍在内部设计 backlog。
- `docs/model.md` 中历史/已废弃段落（`mode: sustained` ADR 0098 旧格式说明、`optimize.time`
  旧字段映射表）保留原始"pulse"表述，未追溯改写——如果这些段落将来需要整体清理，是独立的
  文档维护任务，不在本 ADR 范围。
- "持续量"输入（如 `lm_score` 这类累积型指标，是否也该纳入本次窗宽统一讨论）在
  2026-07-09 讨论中被识别为一个新问题，明确记录为不在第一版解决，见内部任务
  `2026-07-09_task_cumulative-quantity-input-design.md`。
- 全库验证顺带发现的 17 处预存失败（`references/medical` 断链 import 等，与本 ADR 无关）
  未修复，见内部任务 `2026-07-09_task_reference-library-broken-imports-audit.md`。
