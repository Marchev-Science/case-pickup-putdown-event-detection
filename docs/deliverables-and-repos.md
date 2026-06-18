# Where your work goes (this repo is read-only)

This repository is the **problem statement**. Nothing you produce belongs here.
Keeping the case description separate from your solution keeps it reusable and
keeps your work clean. Set this up **before** you start coding.

---

## What goes where

| Artifact | Where it belongs |
|---|---|
| Raw footage (~80 GB) | Stays in the **provided bucket** (read-only). Mirror only the subset you use to local/Colab disk. |
| Your code (curation, models, evaluation) | **Your own solution repo(s)** on GitHub. |
| Your reference table (`clips`, `events`) | Small CSV/Parquet — fine to keep **in your solution repo** (it's small) or in your own storage. **Not** here. |
| Extracted frames / features / clips you cut | **Your own storage** (local, Drive, or your own bucket). Not in git. |
| Trained models / weights | **Your own storage**. Too large for git; not here. |
| Predictions, metrics, plots, write-up | **Your own solution repo** (predictions/metrics are small and worth versioning). |
| Credentials / bucket keys | **Nowhere in git.** Use env vars or a git-ignored secrets file. |

## A suggested repo layout (in *your* account/org)

You don't need many repos. A clean single solution repo works well:

```
your-solution-repo/
├── README.md                 # what you built + how to reproduce
├── environment.yml           # pinned environment
├── configs/                  # YAML configs (fps, model, thresholds, paths)
├── src/                      # importable code
│   ├── data/                 # access, triage, manifest building
│   ├── labeling/             # export → schema, agreement checks
│   ├── standard_model/       # Layer 1
│   ├── vlm/                  # Layer 2
│   └── eval/                 # the shared evaluation (used by both layers)
├── notebooks/                # exploration
├── manifest/                 # clips.csv, events.csv  (small, versioned)
├── results/                  # predictions.csv, metrics, plots (small, versioned)
└── .gitignore                # data/, *.mp4, *.pt, .env, model weights, ...
```

If you prefer to separate concerns, a second repo just for the **manifest +
labeling guideline** is reasonable — but keep raw data and big artifacts out of
git either way.

## What you'll have produced (not a checklist to be graded)

By the end you will naturally have:

- a curated **reference table** (your dataset),
- at least one **event detector** (Layer 1 and/or Layer 2),
- an **evaluation** of it on a held-out test set,
- and, if you did both detectors, an honest **comparison** of the two paradigms.

That set of artifacts *is* the solution. Keep it reproducible, keep it in your
own repos, and leave this one untouched.
