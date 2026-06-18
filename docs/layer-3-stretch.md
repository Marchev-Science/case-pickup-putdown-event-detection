# Layer 3 — Stretch (optional)

Only for teams who have a solid Layer 0 plus at least one of Layers 1/2 working
and still have appetite. **None of this is expected.** Pick at most one or two
directions; depth beats breadth. Keep the scope tight — the same discipline that
made the core case tractable applies here.

---

## A. Spatial localization — *where* on the shelf
Beyond *when* an event happens, report *where* (which shelf region, or a bounding
box around the hand/item at the moment of the event). Mark shelf regions once per
camera view and attach the nearest region to each detected event.

## B. Standard ↔ VLM fusion
Combine your Layer 1 and Layer 2 detectors. Simple, effective ideas: use one to
**propose** candidate moments and the other to **confirm** type; or take the
union/intersection of their events; or have the VLM adjudicate only the
low-confidence cases from the standard model. Measure whether fusion actually
beats the better single model.

## C. Error analysis (high value, low compute)
Systematically categorize your mistakes: occlusion, two-people overlap,
touch-without-take, `pickup`↔`putdown` flips, timing-off-by-tolerance. A clear
taxonomy of *why* the model fails is often more insightful than a slightly higher
F1, and it needs almost no extra compute.

## D. Robustness / ablations
Vary one thing at a time and report the effect: frame rate, resolution, backbone,
window length, temporal tolerance, training-set size. This turns "it works" into
"we understand when and why it works."

## E. Streaming / online detection
Re-frame the detector to run **causally** — emitting events as frames arrive,
without seeing the future — which is closer to how a real store camera would use
it. Discuss the latency/accuracy trade-off.

## F. Active learning / smarter labeling loop
Use a first model to surface the clips/moments it is most unsure about, label
those next, retrain, and see whether targeted labeling improves results faster
than random labeling. Connects nicely back to Layer 0.

## G. Item identity (only if genuinely ahead)
Identify *what kind* of item was picked or put down. This expands scope
considerably and overlaps with full product recognition — attempt only if the
core task is firmly solved, and keep it bounded.

---

Whatever you choose, evaluate it with the **same** measurement discipline as the
core layers and report honestly what changed.
