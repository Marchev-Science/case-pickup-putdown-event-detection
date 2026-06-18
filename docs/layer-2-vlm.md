# Layer 2 — The VLM solution

**Goal:** solve the *same* detection task with a **vision-language model (VLM)** —
a model you can prompt in natural language about images/frames — and compare it
to your Layer 1 detector. Same input (a clip), same output format
(`(type, time)` events), same evaluation.

The constraint is the same: **laptops, maybe a GPU, plus Colab.** So the focus is
**small** VLMs that run locally (quantized) or on a free/Pro Colab GPU, with
API-based VLMs allowed as a convenience for specific steps.

---

## Small VLMs worth trying

Pick **one or two**; do not survey the whole zoo. Versions move fast — check each
project for the latest small release before committing.

**Video-capable small VLMs (preferred — they accept multiple frames natively):**
- **SmolVLM2** (Hugging Face) — explicitly small (sub-billion to ~2B), video
  support, designed to run on modest hardware. A natural first choice.
- **Qwen2.5-VL (2B / 3B / 7B)** — strong video understanding, handles frame
  sequences and timestamps well; the 2–3B sizes are laptop/Colab-friendly when
  quantized.
- **InternVL2.5 (1B / 2B / 4B)** — multi-image and video, small variants run
  locally.
- **LLaVA-OneVision (0.5B / 7B)** — image + multi-frame video.
- **MiniCPM-V** (≈8B) — capable but heavier; treat as the upper bound of "small".

**Image-only tiny models (use frame-by-frame or on frame pairs):**
- **Moondream** — tiny, fast image VLM; reason over sampled frames or
  before/after pairs.
- **Phi-vision** — compact multi-image model.

**Run them with:**
- **Hugging Face `transformers`** + **bitsandbytes** 4-bit quantization (Colab or
  a GPU laptop),
- **Ollama** or **llama.cpp** (GGUF) for easy local, even CPU-ish, inference,
- **vLLM** if you have a capable GPU and want speed.

Most of these run comfortably on free cloud GPUs — see
[`free-gpu-access.md`](free-gpu-access.md), and remember a five-person team has
five separate free quotas to pool.

**API VLMs** (you may have access to general assistant tools): fine to use for
**bootstrapping labels** or as a strong baseline to compare against, but the
*solution* for this layer should center on a **small, open VLM you run
yourself** — that is the point of the layer.

---

## The pipeline

```
clip ─► sample frames (low fps) ─► group into windows ─► prompt VLM per window
     ─► parse JSON ─► merge/deduplicate across windows ─► events (type, time)
```

1. **Sample frames** at low fps (e.g. 1–4 fps). VLMs cost grows with frames, so
   be frugal.
2. **Window** the frames (e.g. a few seconds of frames per call) with their
   timestamps, so the model can place events in time. Overlap windows slightly so
   events on a boundary aren't lost.
3. **Prompt** the model with the operational definitions from
   [`concepts.md`](concepts.md) and ask for **structured output** (JSON) listing
   events with type and timestamp. Force it to answer "no events" explicitly.
4. **Parse** the JSON robustly (models sometimes wrap it in prose or add stray
   text — strip and validate).
5. **Merge** duplicates across overlapping windows and map back to clip-relative
   timestamps.

### A prompt sketch (adapt freely)

> You are shown frames sampled from store footage, each labelled with its time in
> seconds. Detect only two events: `pickup` (a person removes an item from a shelf
> into their hand) and `putdown` (a person returns a previously held item onto a
> shelf). Ignore touching without removing, browsing, and empty scenes. Respond
> with **only** a JSON array of `{"type": "...", "time": <seconds>}`; return `[]`
> if there are no events.

Then **parse** the array, validate types and ranges, and convert to your events
schema.

---

## Two ways to use the VLM (don't confuse them)

- **As the detector (the actual Layer 2 solution):** prompt it to find events;
  evaluate its output against your ground truth.
- **As an annotator / bootstrapper (optional helper for Layer 0):** have a VLM
  propose draft labels that humans then correct. This can speed up curation —
  but **never evaluate a model on labels that the same model produced**, or you
  measure nothing. Keep VLM-assisted labels clearly separated from human-verified
  ground truth.

---

## Evaluating and comparing

Use the **identical** evaluation as Layer 1 ([`concepts.md`](concepts.md) §7) on
the **same** test set, so the comparison is fair. Then write up the trade-offs
you actually observed, for example:

- accuracy (precision/recall/F1, type confusion),
- speed / cost per clip,
- engineering effort to get a working pipeline,
- interpretability and failure modes (e.g. hallucinated timestamps, made-up
  events, ignoring temporal order).

A clear, honest comparison of the two paradigms on your data is one of the most
valuable outcomes of the whole case.

---

## Getting good output from a small VLM

A few habits make these models far more usable for this task:

- **Label every frame with its time** in the prompt and keep windows short, so the
  model places events on real timestamps instead of inventing plausible-looking
  ones.
- **Feed frames in order and check direction.** Telling `pickup` from `putdown`
  means reading the *direction* of motion in time; verify on a few clips that the
  model actually gets the direction right.
- **Parse tolerantly.** Models wrap JSON in stray prose — strip and validate it,
  and keep a count of responses you could not parse so you know how often it
  happens.
- **Sweep the frame budget.** Too few sampled frames miss brief events; too many
  slow everything down. Try a few rates and pick the best trade-off on your
  validation set.
- **Record the quantization** you used (e.g. 4-bit); it changes behaviour, and
  others need it to reproduce your numbers.
