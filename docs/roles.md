# Suggested team roles

A team of five works best when ownership is clear — but **these roles are
suggestions, not assignments.** Split the work however suits your people. Many
will wear more than one hat, and that's expected. The one thing that is *not*
optional: **everyone labels at least some data.** Labeling is how the whole team
builds a shared, correct understanding of the two events, and it is too much work
to leave to one person.

The roles below map cleanly onto the layers, and deliberately include entry
points for members **without** a computer-vision background.

## A natural split for five people

- **Data & access lead** — owns the bucket access, the inventory, the triage
  automation, and the integrity of the `clips` table. Comfortable with the
  command line / Python. *(Layer 0)*

- **Annotation & quality lead** — owns the labeling tool, the labeling
  guidelines, the edge-case decisions, inter-annotator agreement, and the
  `events` table. **No CV background needed** — this is about clarity, judgment,
  and consistency. *(Layer 0)*

- **Standard-model engineer** — owns the non-VLM detector: feature extraction,
  the temporal model or the pose/hand-region baseline, thresholds. *(Layer 1)*

- **VLM engineer** — owns the small-VLM pipeline: frame sampling, prompting,
  JSON parsing, running the model locally/Colab. Prompt-writing and careful
  output-wrangling matter as much as CV here, so this **suits a strong
  generalist**. *(Layer 2)*

- **Integration & evaluation lead** — owns the shared schema, the evaluation
  code (used identically by both detectors), reproducibility, and the final
  comparison and write-up. The glue that keeps Layers 1 and 2 comparable.
  *(spans all layers)*

## Why this shape works for a mixed team

- The heaviest CV-specific work is concentrated in two roles (standard-model and
  VLM engineers); the other three are accessible to people from statistics,
  economics, software, or other backgrounds.
- Layer 0 gives everyone a low-barrier, high-value way to contribute from the
  start.
- A single owner of the schema and evaluation prevents the classic failure where
  two sub-teams build great things that can't be compared.

## However you organize

Choose roles by interest and strength, rotate if you like, and make sure two
things are always owned by *someone*: **the data/labels** and **the evaluation**.
If those two are solid and shared, the rest follows.
