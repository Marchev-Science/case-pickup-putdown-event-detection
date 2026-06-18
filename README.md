# Case: Detecting Pickup and Putdown Events in Store Video

> A computer-vision case about finding two specific human actions inside raw,
> unlabeled video footage from a small self-checkout store.

This repository is the **reference description** of the case. It is meant to be
**read-only**: you do not commit your code, data, labels, models, results, or
any other output here. Those live in your own repositories and storage (see
[`docs/deliverables-and-repos.md`](docs/deliverables-and-repos.md)). Treat this
repo the way you would treat a problem statement handed to you on paper - read
it, copy out the templates you need, and build everything else elsewhere.

---

## The premise

A small self-checkout store has a single camera recording short video clips of
customers as they move through the aisles and explore the shelves - both
visually (looking) and physically (touching, lifting, returning items).

Your job is to build a computer-vision system that watches this footage and
**detects two events**:

1. **`pickup`** - a person removes an item from a shelf and takes it into their
   hand(s).
2. **`putdown`** - a person places a *previously taken* item back onto a shelf
   or surface.

For each event the system should say **what** happened (`pickup` or `putdown`)
and **when** it happened (a timestamp, or a short interval, inside the clip).

That is the whole task. It is deliberately narrow. You are **not** asked to
identify products, recognize individual people, count inventory, or detect theft.
Resist the temptation to broaden the scope - the value of this case is in doing
these two events *well*, on *messy real data*, with *modest hardware*. Precise
definitions of both events (including the tricky edge cases) are in
[`docs/concepts.md`](docs/concepts.md), and reading that file first will save
you a lot of confusion later.

---

## The data (and why it is part of the case)

The footage is delivered through a cloud bucket (~80 GB) to which you have
**read-only** access. It is **not** clean and **not** labeled. Specifically:

- The camera is **motion-triggered**, and the room has moving elements (e.g.
  reflections, lighting, automatic doors, screens). A large share of the clips
  are **false positives with no person in them at all**.
- Some clips are **long stretches where nothing relevant happens**.
- **Nothing is labeled.** There is no list of which clips matter, no timecodes,
  and no event annotations.

So the first real piece of work is **preparing your own dataset**: deciding
which clips are usable, finding where the events occur, timecoding them, and
labeling them into a clean **reference table**. This is not busywork - in
practice, getting from raw footage to trustworthy labels is most of what makes
a project like this succeed or fail.

How to reach the bucket from any operating system and with several different
tools (command line, Python, GUI apps, mounted drives) is documented in
[`docs/data-access.md`](docs/data-access.md). **Do not** try to download all
80 GB - the access doc explains how to work with a sensible subset.

---

## How the case is layered

The case is built as **layers**. Each layer is a complete, self-contained piece
of work that produces something useful on its own. Do the layers in order. If
you finish one and still have appetite, move to the next. There is no
expectation that every team reaches the last layer - a solid result on the
early layers is a real result.

| Layer | Name | What you produce | Doc |
|------|------|------------------|-----|
| **0** | **Data curation** | A clean reference table of clips and events (your dataset) | [`docs/layer-0-data-curation.md`](docs/layer-0-data-curation.md) |
| **1** | **Standard model** | A non-VLM model that detects the two events | [`docs/layer-1-standard-model.md`](docs/layer-1-standard-model.md) |
| **2** | **VLM solution** | A small vision-language model that detects the two events | [`docs/layer-2-vlm.md`](docs/layer-2-vlm.md) |
| **3** | **Stretch** *(optional)* | Extensions: localization, fusion, error analysis, streaming | [`docs/layer-3-stretch.md`](docs/layer-3-stretch.md) |

**Layer 0 is the gateway.** Layers 1 and 2 both need labeled data to learn from
and to test against, so everyone passes through Layer 0. After that, Layers 1
and 2 attack the *same* problem with two *different* families of method, which
lets you compare them on equal footing.

```
                 ┌─────────────────────────────────────────────┐
   raw footage   │  LAYER 0  curate → triage → timecode → label │  →  reference table
   (S3, 80 GB)   └─────────────────────────────────────────────┘     (clips + events)
                                     │
                 ┌───────────────────┴───────────────────┐
                 ▼                                         ▼
        ┌──────────────────┐                     ┌──────────────────┐
        │ LAYER 1          │                     │ LAYER 2          │
        │ standard (non-VLM)│   ── compare ──     │ small VLM        │
        │ event detector    │                     │ event detector   │
        └──────────────────┘                     └──────────────────┘
                 │                                         │
                 └───────────────────┬─────────────────────┘
                                     ▼
                          ┌──────────────────┐
                          │ LAYER 3 (optional)│
                          │ extensions        │
                          └──────────────────┘
```

---

## What "done" looks like

You will end up with, at minimum:

- A **reference table** (your curated dataset) describing clips and the events
  inside them - see the schema in [`docs/manifest-schema.md`](docs/manifest-schema.md).
- At least one **event detector** (Layer 1 and/or Layer 2) that takes a clip and
  outputs predicted events as `(type, time)`.
- A short, honest **comparison of how well it works**, using the evaluation idea
  described in [`docs/concepts.md`](docs/concepts.md) (precision/recall within a
  time tolerance). This is for *you* to understand your own system - it is a
  scientific measurement, not a score someone hands you.

Everything above lives in *your* repositories, never in this one.

---

## Repository map (this repo)

```
case-pickup-putdown-event-detection/
├── README.md                     ← you are here
├── LICENSE                       ← MIT
├── docs/
│   ├── concepts.md               ← terms + precise event definitions (READ FIRST)
│   ├── data-access.md            ← reaching the bucket on any OS / tool
│   ├── free-gpu-access.md        ← free GPU time, and pooling it across the team
│   ├── layer-0-data-curation.md  ← build your dataset
│   ├── layer-1-standard-model.md ← non-VLM detector (model proposals)
│   ├── layer-2-vlm.md            ← small-VLM detector (model proposals)
│   ├── layer-3-stretch.md        ← optional extensions
│   ├── manifest-schema.md        ← the reference-table format
│   ├── roles.md                  ← suggested team roles (use them or don't)
│   ├── best-practices.md         ← tools and working habits worth adopting
│   ├── deliverables-and-repos.md ← where your work goes (not here)
│   └── readme.md                 ← (placeholder)
└── templates/
    ├── labeling-guidelines.md    ← copy out and adapt
    ├── clips.example.csv
    ├── events.example.csv
    ├── predictions.example.csv
    └── readme.md                 ← (placeholder)
```

## Suggested reading order

1. This README.
2. [`docs/concepts.md`](docs/concepts.md) - so the whole team shares one
   definition of `pickup` and `putdown`.
3. [`docs/deliverables-and-repos.md`](docs/deliverables-and-repos.md) - so you
   set up your own repos before writing a line of code.
4. [`docs/data-access.md`](docs/data-access.md) - get one person reading the
   bucket.
5. [`docs/layer-0-data-curation.md`](docs/layer-0-data-curation.md) - start
   curating.
6. [`docs/free-gpu-access.md`](docs/free-gpu-access.md) - line up free GPU time
   (and pool the team's accounts) before the heavier steps.

## All files in this repo

**Top level**
- [README.md](README.md) - this file
- [LICENSE](LICENSE) - MIT license

**docs/**
- [docs/concepts.md](docs/concepts.md) - terms and precise event definitions (read first)
- [docs/data-access.md](docs/data-access.md) - reaching the bucket on any OS / tool
- [docs/free-gpu-access.md](docs/free-gpu-access.md) - free GPU time, and pooling it across the team
- [docs/layer-0-data-curation.md](docs/layer-0-data-curation.md) - build your dataset
- [docs/layer-1-standard-model.md](docs/layer-1-standard-model.md) - non-VLM detector (model proposals)
- [docs/layer-2-vlm.md](docs/layer-2-vlm.md) - small-VLM detector (model proposals)
- [docs/layer-3-stretch.md](docs/layer-3-stretch.md) - optional extensions
- [docs/manifest-schema.md](docs/manifest-schema.md) - the reference-table format
- [docs/roles.md](docs/roles.md) - suggested team roles
- [docs/best-practices.md](docs/best-practices.md) - tools and working habits
- [docs/deliverables-and-repos.md](docs/deliverables-and-repos.md) - where your work goes (not here)
- [docs/readme.md](docs/readme.md) - placeholder

**templates/**
- [templates/labeling-guidelines.md](templates/labeling-guidelines.md) - copyable labeling rulebook
- [templates/clips.example.csv](templates/clips.example.csv) - example clips table
- [templates/events.example.csv](templates/events.example.csv) - example events table
- [templates/predictions.example.csv](templates/predictions.example.csv) - example predictions table
- [templates/readme.md](templates/readme.md) - placeholder

## License

Released under the [MIT License](LICENSE). You are free to use, adapt, and build
on this case description.
