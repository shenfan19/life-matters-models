# 0103 - Adding metadata.log: an In-Model Change-History Record

**Date**: 2026-06-14
**Status**: Implemented (documentation added; adopted gradually in models as needed, no mandatory backfill)
**Category**: Model specification / engineering convention

---

## Background

`metadata.todo` (ADR 0101) records pending items and is forward-looking; but after a model has gone through several rounds of edits, why it ended up the way it currently is exists only in the git history. Problems with relying on git log:

1. A cross-file bulk commit (a rename, a bulk migration) buries a single model's real edit history inside a large number of unrelated diffs.
2. When an AI or a human reviews a model, piecing together why a parameter has the value it does requires `git log --follow -- <file>` followed by reading diff after diff, which is costly and easy to miss something in.
3. A YAML file itself may be copied, extracted into another repository, or shared with a collaborator, and once separated from its git context, its change history is completely lost.

The `evidence` field on `metadata.todo` has already proven the value of writing a diagnostic conclusion into the file itself instead of re-deriving it later; `metadata.log` applies the same idea to changes that have already been completed.

---

## Decision

### Add metadata.log: a change-record dictionary keyed by timestamp

```yaml
metadata:
  log:
    "2026-06-14_10-30-00":
      change: "A one-sentence description of what changed this time"
      reason: "Why it changed this way"
```

- Key: a `YYYY-MM-DD_HH-mm-ss` timestamp (local time). This format was chosen because the string's lexicographic order matches chronological order, and since a YAML dict already preserves insertion order, the two together guarantee a readable order with no extra sorting field needed.
- Value: two fields, `change` (what changed) plus `reason` (why it changed). No further fields are introduced; what changed can already be verified from a git diff, so `metadata.log`'s core value is filling in the motivation or background a git diff cannot show; an extra field (such as a related ADR or a todo item number) is left to be referenced naturally inside the `reason` text (for example, "to resolve the quality item in `metadata.todo`: ..."), rather than given its own field.
- Appending: a new entry is appended to the end of the dict (so the chronologically newest entry is last), and existing entries are never edited or deleted.
- When to record: record an entry for a change with semantic effect on the model (a parameter value, a formula, a constraint, a structural adjustment, an update to literature backing, or resolving a `metadata.todo` item); a pure formatting change, a spelling fix, or comment polish does not need to be recorded.
- No mandatory backfill: an existing model does not need its history log written retroactively; entries can simply start from the next meaningful change.

### Relationship to metadata.todo

- `metadata.todo`: forward-looking, what still needs doing.
- `metadata.log`: backward-looking, what has been done and why.
- When resolving an item in `metadata.todo`, an entry describing the resolution can be appended to `metadata.log` before removing the item from `metadata.todo` (once todo is empty, the `_HOLD` suffix is removed per ADR 0101).

### Not a publication gate

`metadata.log` is an optional field, and its absence does not affect the "publishable" status determined by `_HOLD`/`metadata.todo` (similar to `reviewed: true`; see `docs/model.md` for detail).

---

## Related

- ADR 0101 - the `metadata.todo` structured task list (the design idea behind its `evidence` field is the precedent for this decision)
- `docs/model.md` - the complete YAML schema and field reference
