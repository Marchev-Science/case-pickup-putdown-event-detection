# Labeling guidelines (template — copy into your own repo and use)

This is the rulebook every annotator follows so that labels are **consistent**.
It mirrors the definitions and the fixed edge-case rules in
[`../docs/concepts.md`](../docs/concepts.md) — keep the two in sync.

---

## The two events

- **`pickup`** — a person removes an item from a shelf/surface into their hand(s);
  the item leaves its resting place and is carried. The action runs from the hand
  starting to take the item until the item is in hand.
- **`putdown`** — a person returns a *previously held* item onto a shelf/surface,
  releasing it so it rests there. The action runs from the hand approaching the
  surface until the item is released and settled.

## Label these as NON-events (do not annotate)

- Touching/inspecting an item without removing it.
- Reaching, browsing, or walking past with nothing taken.
- Restocking / new goods being placed that were never "taken".
- Clips with no person (these are dropped during person tracking).

## Timing

- Record times in **seconds from the start of the clip**.
- Store every event as an **interval `[t_start, t_end]`**: `t_start` = the action
  begins, `t_end` = the item is settled (carried, or released and resting).

## Edge-case rules (apply exactly as written)

- **Two items grabbed at once** → **two events** (one `pickup` per item).
- **Pick up then immediately put back** → **two events**: a `pickup` then a
  `putdown`.
- **Item or hand occluded by the body / out of frame** → **exclude** (do not add
  it to the events table).
- **Multiple people acting at the same time** → **label all** the events.
- **Very brief or ambiguous motion** → keep it, but set **`confidence = low`**.

## Confidence

Use `high` / `med` / `low` (or 0–1). Anything you are unsure about → `low`, with a
note.

## The `hard_case` flag

Set **`hard_case = True`** for events that are awkward but still labelable —
multiple people in the shot, partial obscuring, or unusual motion — so they can be
found and reviewed later. (Fully occluded events are excluded, not flagged.)

## Output format

Export to the `events` table in
[`../docs/manifest-schema.md`](../docs/manifest-schema.md):
`event_id, clip_id, type, t_start, t_end, hard_case, annotator, confidence, notes`.

## Agreement routine

- Each annotator records their name in `annotator`.
- Two people independently label a shared subset; compare presence, type, and
  timing. Where you differ, the rule was unclear — clarify the wording here, then
  continue.
