# Layer 1 — The standard (non-VLM) model

**Goal:** build an event detector **without** a vision-language model. Input: a
clip. Output: predicted events as `(type, time)`, written in the same format as
your ground-truth `events` table so you can compare directly.

The hardware reality shapes everything here: **laptops, maybe with a GPU, plus
Google Colab for short bursts.** Full-resolution, full-fps video deep learning
is out of reach. The proposals below are chosen to fit that constraint. Pick
**one baseline** and, if time allows, **one stronger approach** — don't try them
all.

---

## The two-track strategy

```
            ┌──────────────────────────────────────────┐
  clip ───► │ TRACK A  pose / hand-region + simple logic │ ──► events   (start here)
            └──────────────────────────────────────────┘
            ┌──────────────────────────────────────────┐
  clip ───► │ TRACK B  features → temporal detector      │ ──► events   (stronger)
            └──────────────────────────────────────────┘
```

### Track A — Pose / hand-region baseline (start here; very accessible)

This track needs almost no GPU, is interpretable, and lets non-CV team members
contribute real logic.

Idea: a `pickup`/`putdown` is fundamentally a **hand interacting with a shelf
region and an object moving with the hand**. So:

1. Detect **hands / body keypoints** per frame with a light model:
   - **MediaPipe Hands / Pose** (runs on CPU, very fast),
   - **Ultralytics YOLO-pose**, or **RTMPose**.
2. Track when a hand **enters a shelf region** (you can mark shelf regions once
   per camera view) and **comes back out**.
3. Decide `pickup` vs `putdown` from **what the hand carries** and the **time
   direction**: empty hand in → full hand out ≈ `pickup`; full hand in → empty
   hand out ≈ `putdown`. "Carrying" can be approximated by a small object/box
   detector or by appearance change in the hand region.
4. Smooth the per-frame signal and threshold it into event times.

This will not be perfect, but it gives a working, explainable detector and a
strong baseline to beat. It is a great fit for a team without CV specialists.

### Track B — Pre-extracted features + a temporal detector (the SOTA route)

The key trick that makes SOTA reachable on weak hardware: **separate feature
extraction from detection.**

1. **Extract clip features once** with a pretrained video (or image) backbone,
   sampling at low fps. Do this on Colab's GPU; you only do it once and then
   reuse. Backbone options, lightest first:
   - **CLIP / SigLIP / DINOv2** per-frame image features (cheap, surprisingly
     strong, run on modest GPUs),
   - **VideoMAE** features,
   - **I3D** features (the classic baseline these detectors were built for).
2. **Train a temporal action detector on those features.** Because it consumes
   pre-computed features rather than raw video, it trains fast on a single GPU.
   Strong, well-maintained choices:
   - **ActionFormer** — <https://github.com/happyharrycn/actionformer_release>
   - **TriDet** — <https://github.com/dingfengshi/TriDet>
   - **TemporalMaxer** — <https://github.com/TuanTNG/TemporalMaxer>
   These are SOTA-level on standard untrimmed-video benchmarks and are designed
   exactly for the "find the segments in a long video" problem.
3. The detector outputs candidate segments with a type and score; threshold and
   convert to your `(type, time)` event format.

### A lighter middle option — per-frame/clip classifier + smoothing

If Track B's repos feel heavy, train a small head to classify short windows into
`{pickup, putdown, background}`, then smooth the probabilities over time and pick
peaks. Efficient backbones meant for this:
- **TSM (Temporal Shift Module)** — very cheap, adds temporal modelling to a 2D
  CNN,
- **X3D**, **MoViNet** (designed for mobile/streaming), **MViT** (small variant).
This sits between Track A and Track B in effort and usually in accuracy too.

---

## Making it fit on a laptop / Colab

- **Downsample aggressively:** 2–8 fps and a small resolution are usually enough
  for these events. Test the effect — more fps is not always better.
- **Extract features once, cache them**, then iterate on the cheap detector/head.
- **Work on windows around events** when training, to fight class imbalance
  (events are rare).
- **Mixed precision** and small batch sizes on Colab; keep clips short.
- Prefer **pretrained** backbones; you do not have the data or compute to train
  one from scratch.

For where to get GPU time for the feature-extraction and training steps without
paying — and how to pool it across the team — see
[`free-gpu-access.md`](free-gpu-access.md).

---

## Evaluating it

Run the evaluation from [`concepts.md`](concepts.md) §7 on your **validation**
set while developing, and only at the very end on the **test** set: match
predicted and true event intervals of the same type (temporal IoU or midpoint
tolerance), then report precision / recall / F1 plus the `pickup`↔`putdown`
confusion. Treat **threshold choice as part of the model**: pick thresholds on
the validation set, never on the test set. Save predictions to a `predictions`
table in the schema from [`manifest-schema.md`](manifest-schema.md).

---

## Two details that decide whether this works

- **Preserve temporal direction.** A `pickup` and a `putdown` look like each other
  played backwards, so the *order of motion in time* is what separates them. Build
  features and windows that keep temporal order (consecutive frames in sequence,
  not a bag of shuffled frames), or the model will flip the two types.
- **Get a baseline running before reaching for a bigger model.** A working Track A
  detector you fully understand teaches you more — and gives Layer 2 something
  concrete to compare against — than a half-trained Track B.
