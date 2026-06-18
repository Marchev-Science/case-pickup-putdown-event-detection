# Getting GPU time for free

The heavy steps in this case — running person tracking over many clips, extracting
video/image features, training the Layer 1 detector, running a small VLM in Layer
2 — go much faster on a GPU. You do not need to buy one. Several services give GPU
time away for free, and because you are a **team of (around) five**, you have a
real advantage: **each member has their own free quota on each platform**, so you
can pool a surprising amount of compute.

> Free-tier details (hours, hardware, limits) change often. Treat the numbers
> below as a snapshot and check each service's own page before you rely on it.

---

## The platforms

### Google Colab — easiest, notebook-based
Free **NVIDIA T4 (16 GB)** GPUs from a browser notebook. There is no fixed
published quota; availability is dynamic (often in the range of ~15–30 GPU-hours
per week per account), sessions can run up to ~12 hours, and they disconnect after
about 90 minutes of inactivity. Save anything important to Google Drive — the disk
is wiped when the session ends. Each **Google account** gets its own allocation.

### Kaggle Notebooks — the most predictable quota
Free **~30 GPU-hours per week** per account (sometimes more under a "floating"
quota), on **T4 ×2 (32 GB combined)** or **P100 (16 GB)**, with sessions up to ~12
hours and a clear remaining-quota counter. You verify your account with a phone
number to unlock the GPU. Persist outputs as Kaggle Datasets. Fast model downloads
from Hugging Face. Each **Kaggle account** gets its own ~30 h/week.

### Hugging Face — short GPU bursts and hosted inference
- **ZeroGPU on Spaces**: a shared pool of high-end GPUs that attaches only while
  your function runs. Free accounts get a small **daily** GPU allowance (a few
  minutes/day; PRO at ~$9/month raises this a lot). Gradio-SDK only. Best for
  **short inference bursts and demos**, e.g. running a small VLM on sample clips —
  not for long training jobs.
- **Inference Providers / Inference API**: call many hosted models over the API
  with a small monthly free credit — handy to try a VLM without installing
  anything.
- **Community GPU grants**: open/educational projects can apply for free
  persistent GPU on a Space.
Each **Hugging Face account** has its own daily ZeroGPU allowance.

### Lightning AI (Studios) — free monthly GPU hours
Cloud "Studios" (a persistent VS Code/Jupyter environment) with a block of **free
GPU hours each month** per account, plus persistent storage — good for slightly
longer development sessions than a notebook. Each account gets its own monthly
allotment.

### Amazon SageMaker Studio Lab — no AWS account, no card
A free Jupyter environment offering a **T4 GPU for a few hours per session** plus
persistent storage, with **no credit card and no AWS account** required (sign-up
approval can take a day or two). Check current availability before depending on
it.

### Paperspace Gradient (DigitalOcean) — free notebook GPUs
A free notebook tier with GPU instances when capacity is available. Useful as an
extra pool; availability varies.

### If you outgrow the free tiers
Cheap pay-as-you-go GPUs (e.g. RunPod, Vast.ai) or one-time cloud credits
(Google Cloud's new-user credit, GitHub Student Developer Pack, etc.) are
fallbacks — but for this case the free notebook tiers above are usually enough if
you pool and plan.

---

## Using five accounts well (the team multiplier)

You are genuinely several different people, each entitled to your own free account
— that is exactly how these services are meant to be used. (What platforms forbid
is *one* person farming many accounts to dodge limits; that is not what this is.)
Each person should sign up for the platforms above so the team has, in effect,
five Colab allocations, five Kaggle ~30 h/week quotas, five ZeroGPU daily
allowances, and so on.

To turn that into real throughput:

- **Split the one-time heavy jobs by data shard.** Person tracking and feature
  extraction run once over the dataset. Divide the clips into five chunks, each
  member runs their chunk on their own account, and you finish in roughly a fifth
  of the wall-clock time.
- **Share the results, not the raw work.** Write the outputs (tracklets, the
  manifest rows, extracted features) to a shared place everyone can read — a
  shared Google Drive, a Hugging Face dataset, your own bucket, or your solution
  repo for the small files. Nobody re-runs a chunk someone else already did.
- **Keep a simple coordination sheet:** who owns which chunk, which platform they
  used, and how much quota they have left. It avoids two people burning quota on
  the same thing.
- **Checkpoint often and save off-platform.** Free sessions disconnect; write
  intermediate results to durable storage (Drive / Kaggle Datasets / your bucket)
  so a disconnect costs minutes, not hours.
- **Match the job to the platform.** Long-ish training → Kaggle (predictable
  quota) or Lightning; quick feature-extraction sweeps → Colab; short VLM
  inference demos → Hugging Face ZeroGPU; no-card quick tests → SageMaker Studio
  Lab.
- **Cut the work before you spend GPU on it.** Downsample fps and resolution, and
  use the `active_start_s`/`active_end_s` spans from Layer 0 so you never process
  the dead parts of clips.

---

## A note on data

Your footage sits in the provided read-only bucket. On a free notebook, pull only
the **subset you need** for the job at hand (see
[`data-access.md`](data-access.md)), and persist your outputs to your own storage
— never to this case's bucket, and never commit large files or credentials to git.
