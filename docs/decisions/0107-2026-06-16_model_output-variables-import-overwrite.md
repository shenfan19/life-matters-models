# ADR 0107 — output_variables / output_types import 行为统一为覆盖（取代并集）

## 状态

✅ 已实施

## 日期

2026-06-16

## 背景

ADR 0063 为 `output_variables` 和 `output_types` 设计了特殊规则：根模型未定义时，取所有 imports 的并集。其他所有字段（`start_date`、`end_date`、`step_size` 等）都遵循 deep merge（后 import 覆盖前，根模型最终覆盖）。

这条例外规则造成两个问题：
1. **行为不一致**：模型作者需要记住两套规则。
2. **并集可能引入噪音**：模型 A import 了 B 和 C，B 的 `output_variables` 是 `[x]`，C 是 `[y]`，结果变成 `[x, y]`——但 A 的作者可能只想要 C 的 `[y]`（因为 C 是更完整的模型）。模型作者对每个模型有明确意图，并集反而可能带来混乱。

## 决策

`output_variables` 和 `output_types` 不再特殊处理，改为与所有其他字段相同的行为：

- **根模型未定义**：继承最后一个 import 的值（deep merge 自然结果，后 import 覆盖前）。
- **根模型显式定义了任一输出字段**：根模型定义优先；若仅定义其中一个，另一个从 import 继承的值同时清除（防止两种过滤机制意外混用）。
- **两个字段都不存在或都为空**：输出所有变量。

## 实施

- `sim_engine/src/model_structure/loader.py`：删除 `imported_output_variables` / `imported_output_types` 并集收集逻辑及 `_append_unique` 静态方法；`merge_dicts` 自然处理覆盖继承。
- `model.md`：更新"输出变量选择规则"一节。
- ADR 0063：添加修订说明。

## 影响

- 有多个 import 且各自定义了 `output_variables` 的模型，行为从"并集"变为"最后一个 import 的值"。实践中这类模型几乎不存在（通常只有一个 import，或根模型自己定义了输出变量）。
- 逻辑更简单，规则更统一，模型作者无需记忆特例。
