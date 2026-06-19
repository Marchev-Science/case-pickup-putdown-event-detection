# The reference-table (manifest) schema

Your curated dataset is the backbone of the whole case. Keep it as **two linked
tables** plus a third for model output. CSV is fine to start; Parquet is nicer
once it grows. Example files in the exact format are in the
[`../templates/`](../templates) folder.

Two tables (rather than one giant one) keep clip-level facts separate from
event-level facts and avoid repeating clip metadata on every event row.

---

## Table 1 — `clips` (one row per video file)

| column | type | meaning |
|---|---|---|
| `clip_id` | string | stable unique id you assign (e.g. `clip_00012`) |
| `s3_key` | string | object key/path in the bucket |
| `duration_s` | float | clip length in seconds |
| `fps` | float | frames per second of the source |
| `width`, `height` | int | resolution in pixels |
| `n_person_tracks` | int | how many person tracklets were found (0 = empty clip) |
| `usable` | bool | kept for labeling/use |
| `active_start_s` | float | start of the span where a person is present (for trimming dead footage) |
| `active_end_s` | float | end of that span |
| `split` | enum | `train` / `val` / `test` |
| `session_id` | string | optional: recording session/person group, for leakage-safe splits |
| `notes` | string | free text (extra active spans, quality issues, oddities) |

## Table 2 — `events` (one row per labeled event = ground truth)

| column | type | meaning |
|---|---|---|
| `event_id` | string | unique id for the event |
| `clip_id` | string | foreign key → `clips.clip_id` |
| `type` | enum | `pickup` or `putdown` |
| `t_start` | float | event start, seconds from clip start |
| `t_end` | float | event end, seconds from clip start |
| `hard_case` | bool | set `True` for multiple-people / partially-obscured / unusual events to review later |
| `annotator` | string | who labeled it |
| `confidence` | enum/float | how sure (`high` / `med` / `low`, or 0–1) |
| `notes` | string | free text (what was picked, why it's a hard case, etc.) |

> Every event is stored as the **interval** `[t_start, t_end]`. Follow the fixed
> edge-case rules in [`concepts.md`](concepts.md) §5 when deciding what is one
> event vs. two, what to exclude, and when to set `confidence = low` or
> `hard_case = True`.

## Table 3 — `predictions` (one row per predicted event = model output)

Same shape as `events`, so evaluation is a direct comparison. Add a `score`.

| column | type | meaning |
|---|---|---|
| `pred_id` | string | unique id for the prediction |
| `clip_id` | string | foreign key → `clips.clip_id` |
| `type` | enum | `pickup` or `putdown` |
| `t_start`, `t_end` | float | predicted interval |
| `score` | float | model confidence (used for thresholding / PR curves) |
| `model` | string | which model/run produced it (e.g. `layer1_tridet_v3`) |

---

## Conventions everyone follows

- **Times are seconds from the clip's own start**, as floats, same unit
  everywhere.
- **`clip_id` is the join key** between all three tables.
- **One file = one `clip_id`**; never reuse ids.
- Keep an immutable copy of each labeled version (don't silently overwrite) so
  results stay reproducible.
- Keep the human ground-truth `events` table and any **VLM-suggested** labels in
  separate files. Evaluate only against the human-verified ground truth (see
  [`layer-2-vlm.md`](layer-2-vlm.md)).

## Evaluation, concretely

To score a model: for each clip, match `predictions` to `events` of the **same
type** whose **intervals line up** — either temporal IoU above a threshold
(e.g. `tIoU ≥ 0.5`) or interval midpoints within a tolerance (e.g. ±1 s).
Matched = true positive; unmatched prediction = false positive; unmatched
ground-truth = false negative. From those counts compute precision, recall, F1,
and a `pickup`↔`putdown` confusion count. (See [`concepts.md`](concepts.md) §7.)
