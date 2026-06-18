# Best practices and tools

Opinionated, practical defaults. You don't have to follow every one, but teams
that do tend to avoid the usual pitfalls. The theme throughout: **make your work
reproducible and your two detectors comparable.**

---

## Working habits

- **Decide the schema and evaluation before modeling.** If
  [`manifest-schema.md`](manifest-schema.md) is fixed and one person owns the
  evaluation code, Layers 1 and 2 stay comparable. Most wasted effort comes from
  skipping this.
- **Cap the labeling effort early.** Agree a target number of labeled events,
  reach it, move on. Come back only if you have time.
- **Fix random seeds** and keep a fixed train/val/test split. Don't peek at test.
- **One change at a time.** When something improves (or breaks), you want to know
  what caused it.
- **Notebooks for exploration, scripts/modules for anything you run twice.**

## Environment & reproducibility

- Pin your environment: `environment.yml` (conda/mamba) or `requirements.txt`
  (pip), or use **uv**/**pixi** for fast, reproducible installs.
- Keep configuration in **YAML/JSON config files** (frame rate, model, thresholds,
  paths), not hard-coded in scripts. Hydra is nice if you like it.
- Record, for every result, the config + code version that produced it (a git
  commit hash in the run log is enough).
- For data/version tracking, **DVC** is optional but handy; at minimum, never
  overwrite a labeled dataset in place — version it.

## Experiment tracking

- **Weights & Biases** or **MLflow** if you want dashboards; a disciplined CSV /
  markdown log of `{config, metrics, notes}` is perfectly fine and sometimes
  better for a short project.

## Video & data tooling

- **ffmpeg / ffprobe** — inspect, trim, transcode, sample frames. Indispensable.
- **OpenCV**, **PyAV**, **decord**, **torchvision.io**, **imageio** — read frames
  in Python (`decord` is fast for sampling; `PyAV` is robust).
- **pandas** / **pyarrow** — manage the manifest tables (CSV ↔ Parquet).

## Labeling

- **CVAT**, **Label Studio**, **VIA**, or **ELAN** (see
  [`layer-0-data-curation.md`](layer-0-data-curation.md)). Pick **one**, export to
  the agreed schema.

## Modeling toolkits

- **Track A (pose/hands):** MediaPipe, Ultralytics YOLO(-pose), RTMPose.
- **Track B (features → detector):** CLIP/SigLIP/DINOv2 or VideoMAE/I3D features;
  ActionFormer / TriDet / TemporalMaxer for temporal detection.
- **Lighter video nets:** TSM, X3D, MoViNet, MViT (small).
- **VLMs:** Hugging Face `transformers` + bitsandbytes (4-bit), **Ollama** /
  **llama.cpp** for easy local runs, **vLLM** for speed on a real GPU.

## Compute

- Do GPU-heavy steps (person tracking, feature extraction, training) on free
  cloud GPUs and **cache the outputs** so you never repeat them. See
  [`free-gpu-access.md`](free-gpu-access.md) for the platforms and for how a
  five-person team pools its free quotas by splitting one-time jobs across
  members' accounts.
- Persist anything important to your own storage — notebook disks vanish between
  sessions.
- Downsample fps/resolution and use the Layer 0 active spans before anything
  expensive; verify it doesn't hurt accuracy much.

## Git hygiene

- This case repo is **read-only** — your code lives in your own repos (see
  [`deliverables-and-repos.md`](deliverables-and-repos.md)).
- **Never commit data, large model files, or credentials.** Add a `.gitignore`
  for `data/`, `*.mp4`, `*.pt`/`*.onnx`, `.env`, and similar from day one.
- Small, frequent commits with clear messages; use branches/PRs if the team is
  comfortable with them.

## Reproducibility checklist (quick self-test)

- Could a teammate re-run your result from a clean checkout + the config?
- Is the test set untouched until the final evaluation?
- Are both detectors evaluated by the **same** code on the **same** split?
- Are credentials and large files kept out of git?
