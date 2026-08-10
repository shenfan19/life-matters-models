# 建模实践指南索引

`LM_format_1.0.md` 是正式的、版本化的格式规范，定义一个 YAML 文件要满足什么条件才算合法的 LM file，是可以独立引用的基准文件。这个目录是在那份规范之上，这一个具体项目关于怎么写好一个 LM 模型的实践指南，随建模实践持续调整，不追求版本化，也不重复维护 `LM_format_1.0.md` 已经写清楚的字段定义，涉及具体字段结构时直接指向规范对应章节。

目录名叫 `authoring` 而不是 `model`，是为了跟仓库根目录下存放实际模型 YAML 文件的 `models/` 目录区分开，两个名字长得太像会让人翻错地方；`authoring` 对应"写模型"这个动作本身，`LM_format_1.0.md` 的引言里也用 author 描述同一件事，用词是一致的。

## 目录

- [methodology.md](methodology.md)：LM 的核心方法论，耦合而非堆叠的判断标准，以及模型纳入标准与学科覆盖盘点表，写新模型或审查已有模型该不该保留某个机制时先看这份。
- [description_writing.md](description_writing.md)：`metadata.description` 与 `metadata.references` 的写作规范，`problem/method/result/limitations` 四字段怎么分工、来源文献的贡献说明怎么写。这份文件是自包含的，可以单独交给另一个协作者或另一个 AI 会话，作为写模型 description 的素材。
- [ratings.md](ratings.md)：`metadata.ratings` 评分体系，`topic_importance`/`framework_demand`/`evidence_quality`/`validation_confidence` 等通用字段与各模型类型专属字段的打分标准。
- [bookkeeping.md](bookkeeping.md)：`metadata.todo`/`metadata.log`/`history/` 目录/`reviewed` 字段，模型文件从起草到发布的状态标记与改动追溯约定。
- [variables_and_equations.md](variables_and_equations.md)：变量三种类型与 `evidence_type` 换算规则，方程的 `step_unit`/`priority`/Euler 离散积分等执行细节，比规范本身的字段定义更详细，含设计理由和已知陷阱。
- [regimens_and_optimizer.md](regimens_and_optimizer.md)：`simulation.plans[*].regimens` 与 `optimizer` 的完整用法，含正反例、`lm_score` 健康时长核心指标、T1-T4 决策变量层级、`optimizer.results` 内嵌格式。
- [imports_and_organization.md](imports_and_organization.md)：`imports` 合并规则与模型分类目录约定。

## 与 LM_format_1.0.md 的分工

字段合不合法、有哪些取值、必填还是可选，以 `LM_format_1.0.md` 为准；这个目录里的文件默认这些字段定义已经成立，补充的是这个项目里怎么把这些字段用好、哪些写法是推荐的、哪些是已知会踩坑的。两边内容一旦出现冲突，以 `LM_format_1.0.md` 为准，并且应该视为这个目录的文档需要修正，而不是规范本身需要迁就旧写法。
