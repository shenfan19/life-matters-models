# models/papers — 论文专用仿真场景

## 定位

`papers/` 存放**与学术论文直接对应的正式仿真场景**，用于复现论文数值、执行对照实验、生成 Pareto 前沿图。

每个场景文件经过文献参数校准，包含完整的 MC 和 Opt 配置，可复现论文中报告的定量结果。

## 目录结构

```
papers/
  s1/   LM format 根论文（格式/规范/模型库）
  s2/   MASLD-HUA 临床应用论文
  s3/   IHIO 系统科学范式论文
  s4/   数字健康生态基础设施论文
```

| 目录 | 论文定位 | 目标期刊 | 状态 |
|------|---------|---------|------|
| `s1/` | LM format 根论文 | SoftwareX | 撰写中 |
| `s2/` | 临床应用（MASLD-HUA） | JMIR Formative Research | 撰写中 |
| `s3/` | IHIO 系统科学范式 | JBI / IEEE JBHI | 撰写中 |
| `s4/` | 数字健康基础设施 | 系统工程理论与实践（中文） | 待撰写 |

## 文件清单

| 文件 | 案例描述 | 状态 |
|------|---------|------|
| `s1/banister.yaml` | Banister 双室适应-疲劳验证 | ✅ 可运行 |
| `s1/ckd_protein_noref.yaml` | CKD 蛋白质-肌肉权衡 | 待参数补全 |
| `s1/fatty_liver_periodization.yaml` | 脂肪肝运动进阶时机 Pareto 优化 | ✅ 可运行 |
| `s1/infant_breastfeeding.yaml` | 新生儿母乳喂养节奏 Pareto 优化 | ✅ 可运行 |
| `s2/bergman_glucose.yaml` | Bergman 最小模型血糖-胰岛素 | ✅ 可运行 |
| `s2/ibs_diet.yaml` | 肠易激综合征饮食管理 | ✅ 可运行 |
| `s2/masld_insulin_noref.yaml` | MASLD×胰岛素抵抗联合仿真 | 待参数补全 |
| `s3/burnout_allostatic_noopt.yaml` | 工作应激-恢复行为跨域 | opt 待修复 |
| `s3/sleep_schedule_noopt.yaml` | Borbély 双过程睡眠优化 | opt 待修复 |
| `s3/smoking_stress_noref.yaml` | 工作压力-吸烟-健康跨域 | 待参数补全 |
| `s4/ckd_protein_pareto_nosim.yaml` | CKD 完整 Pareto 前沿对照（继承 s1） | sim 待修复 |
| `s4/hypertension_gout_3obj_nosim.yaml` | 高血压痛风三目标扩展（继承本目录） | sim 待修复 |
| `s4/hypertension_gout_noref.yaml` | 高血压合并痛风药物冲突 | 待参数补全 |
| `s4/postpartum_recovery_nosim_noopt.yaml` | 产后多维恢复 Pareto 权衡 | sim+opt 待修复 |
| `s4/sodium_lifestyle_bp.yaml` | 钠摄入×体力活动血压代偿 | ✅ 可运行 |

## 验证要求

每个场景须通过对应层级的验证协议，详见 [`docs/validation.md`](../../docs/validation.md)：

| 场景 | 验证层 |
|------|--------|
| `s1/banister` | 层1（解析解）+ 层2（Morton 1990 Fig.3） |
| `s1/ckd_protein_noref` | 层2（GFR下降速率、KDIGO自然史） |
| `s4/hypertension_gout_noref` | 层2（HCTZ效应量，Law 2009） |
| s3 系列 | 层3（可行域结构 + IHIO 论点验证） |
| s4 系列 | 层3（Pareto合理性 + 对照实验） |

## 与 temp/ 的区别

`models/temp/` 存放调试用快速场景，**临时目录，后续清理删除**。凡需要长期保留和论文引用的场景，应移入此目录并完善参数文献来源。
