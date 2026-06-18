# Layer 0 — Data curation

**Goal:** turn ~80 GB of raw, unlabeled, motion-triggered footage into a clean,
trustworthy **reference table** (your dataset) that Layers 1 and 2 can train on
and test against.

This is the gateway layer. Both later layers depend on what you build here. It is
also where a mixed-background team divides work naturally: some people set up
access and the automatic steps, others do the human labeling, someone owns the
schema.

The exact table format is in [`manifest-schema.md`](manifest-schema.md). The event
definitions and the fixed edge-case rules are in [`concepts.md`](concepts.md) and
[`../templates/labeling-guidelines.md`](../templates/labeling-guidelines.md).

The curation pipeline has a clear shape — **track people first, and let the
tracking drive everything else**:

```
inventory ─► detect & track people ─► drop clips with no people
          ─► timecode the active span (person present) ─► label the two events
          ─► check agreement ─► split
```

---

## Step 1 — Inventory

List the whole bucket and record, for every clip: its key/path, duration, fps,
resolution, and size. This is the first version of your `clips` table (one row
per video). Read metadata with `ffprobe` (from ffmpeg) or `opencv`/`PyAV`. You do
not need to download everything for this — listing plus reading headers covers
most of it.

## Step 2 — Detect and track people

Run a person detector on sampled frames of each clip and **track** each person
across frames so you get *tracklets* (a person's path through the clip with the
times they are visible). Good, light tools:

- **Ultralytics YOLO** (person class) for detection,
- a simple tracker on top — **ByteTrack** or **BoT-SORT** (both ship with
  Ultralytics) — to link detections into tracklets,
- **MediaPipe** as an even lighter alternative for presence detection.

Sampling one frame every second or two is plenty for this; you are looking for
*whether and when* a person is present, not fine motion yet. Record per clip how
many person tracklets you found.

## Step 3 — Drop the empty clips using the tracks

Because the camera is motion-triggered with active elements in the room, many
clips contain **no person at all**. Any clip where tracking finds **no person**
is a false trigger — mark it `has_person = False`, `usable = False`, and set it
aside. This is what shrinks 80 GB down to a workable set, and the person tracks
make the decision objective rather than a guess. Keep the flag rather than
deleting anything, so you have an audit trail and can revisit.

## Step 4 — Timecode the active span

For each kept clip, use the person tracks to find the **active span** — the time
window when a person is actually present (from the first frame a person appears
to the last). Record it as `active_start_s` / `active_end_s` in the `clips`
table. This lets you **cut away the useless parts later** — the dead footage
before anyone arrives and after the person leaves — so that Layers 1 and 2 only
process the part of each clip where something can happen. If a clip has people
coming and going in separate bursts, record the main span and note the others.

## Step 5 — Label the two events

Within the active spans, find the `pickup` and `putdown` events and record, per
event: the clip, the type, and the **interval** `[t_start, t_end]`. Apply the
fixed edge-case rules from [`concepts.md`](concepts.md) §5 (two items → two
events; occluded → exclude; multiple people → label all; brief/ambiguous → keep
with `confidence = low`; hard cases → set the `hard_case` flag). This produces
your `events` table (one row per event).

**Use a real annotation tool — not a text editor:**

- **CVAT** — <https://www.cvat.ai/> — strong for video, supports interpolation and
  events; self-hostable or cloud.
- **Label Studio** — <https://labelstud.io/> — flexible, has a video/timeline
  template, easy to start.
- **VIA (VGG Image/Video Annotator)** — <https://www.robots.ox.ac.uk/~vgg/software/via/> —
  zero-install single HTML file, good for temporal segments.
- **ELAN** — <https://archive.mpi.nl/tla/elan> — built for time-aligned annotation
  of media; excellent for marking event intervals on a timeline.

Pick **one** tool, have everyone use it, and export to the schema in
[`manifest-schema.md`](manifest-schema.md).

## Step 6 — Check agreement

Labels are only useful if they are consistent. Have two people independently label
a shared handful of clips and compare where they agree on event presence, type,
and timing. Where they differ, the definition was unclear — refine the wording in
your labeling guideline, then carry on. Record each labeler's name in the
`annotator` field.

## Step 7 — Split the data

Partition your labeled clips into **train / validation / test**, splitting **by
whole clip** (and by session/person where you can tell, using the tracklets) so
the same scene does not appear in two splits — see [`concepts.md`](concepts.md)
§6. Record the split in the `clips` table so it is fixed and reproducible, and
keep the test set aside until the final evaluation.

---

## What you hand to the next layers

- `clips.csv` — one row per video, with metadata, `has_person`/`usable` flags, the
  `active_start_s`/`active_end_s` span, and the `split`.
- `events.csv` — one row per labeled event, linked to a clip, with type, interval,
  `confidence`, and `hard_case` flag.
- The labeling guideline you actually used.

(See the templates folder for example files in the exact format.)
