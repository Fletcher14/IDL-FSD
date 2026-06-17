# IDL.Co — Full Self-Driving Simulator

> A closed-loop autonomous-driving research stack built from scratch: procedural
> world generation, BEV perception, graph-based route planning, trajectory
> optimization, and a rigorous, leakage-free validation methodology.

**Status:** active R&D · **Author:** Emmanuel Fletcher · **Org:** Innovation Drive Line Corp (IDL.Co)

> **Note:** This is a portfolio repository. It documents the architecture, methodology,
> and validated results of the project. The proprietary source code, models, and
> datasets are not published here. See [LICENSE](LICENSE).

---

## What this is

A from-scratch self-driving simulator and perception/planning stack, developed as
an independent research effort. The system runs a full closed loop — perceive →
plan → control → act — on a large procedurally generated world, and is validated
with methodology designed to avoid the data-leakage pitfalls that inflate results
in many imitation-learning setups.

The emphasis throughout is **honest evaluation**: every performance claim is measured
against held-out, never-seen data, with the test conditions matching deployment.

---

## Architecture



**Highlights:**
- **Procedural world:** ~441 km^2 road network — highway ring, radial spokes,
  interchange ramps, ~1,786 lanes, 17 destinations. Deterministic by seed.
- **Perception:** compact CNN (~1.05M parameters) producing a road-occupancy map
  and an agent-detection head, trained by imitation on recorded driving, evaluated
  under a sensor-realistic observation model (occlusion + noise on the input, clean
  ground-truth targets).
- **Route planning:** analytic grid-crossing graph with KDTree proximity stitching
  and A* over type-aware lane costs; forward-progress re-sync to prevent follower
  drift on a large graph.
- **Motion planning:** candidate trajectory sampling scored by a multi-term cost
  function (collision, occupancy, detection, speed, lateral, progress), with the
  hot scoring loop migrated to C++ (pybind11) for throughput.
- **Velocity prediction (in progress):** a perception-only multi-object tracker
  that clusters dense detections into objects, estimates world-frame velocity with
  ego-motion compensation, and projects agent motion forward for predictive
  avoidance.

---

## Methodology — why the results are trustworthy

The project treats **evaluation rigor as a first-class feature.** Key practices:

- **Leakage-free validation by construction.** A dedicated world seed is reserved
  from training and used only for evaluation, guaranteeing the test maps are
  genuinely novel. Data provenance is stamped per recording session.
- **Identical train/eval conditions.** The evaluation harness mirrors the training
  validation loop exactly — same dataset pipeline, same loss computation, same
  sensor-observation regime — so comparisons are apples-to-apples.
- **Head-to-head model comparison.** New and prior models are scored on the *same*
  held-out novel map under identical conditions; improvement is reported only when
  measured this way, not against a model's own easier holdout.
- **Closed-loop tracking.** Driving runs are logged to a dedicated experiment
  tracker (destinations reached, emergency-braking events, cost profile,
  throughput) separate from the perception-training metrics.

---

## Selected result — perception generalization (Phase 3.1)

A data-expansion experiment added procedurally novel maps to the training corpus
and retrained the perception model, validated against a reserved, never-trained map:

| Metric (novel-map holdout) | Prior model | Retrained model | Change |
|---|---|---|---|
| Validation loss (total)    | 0.0849      | **0.0684**      | **-19.5 %** |
| Detection loss             | 0.1197      | **0.0847**      | **-29.2 %** |
| Occupancy loss             | 0.0251      | 0.0260          | +3.7 % (negligible) |

The improvement is concentrated in **agent detection on unseen maps** — exactly the
capability that drives trustworthy detection-aware planning — with no meaningful
cost to occupancy. The comparison is leakage-free: both models scored on the same
reserved novel map, same conditions.

*(Full methodology and reproducibility notes in [docs/](docs/).)*

---

## Tech stack

- **Core:** Python, PyTorch (perception), NumPy/SciPy
- **Performance:** C++ via pybind11 for the trajectory-scoring hot path
- **Tooling:** Docker (reproducible training/eval), MLflow (experiment tracking)
- **Sim/visual:** custom procedural world, pygame cockpit renderer, live telemetry HUD

---

## Documentation

- [docs/architecture.md](docs/architecture.md) — system architecture in depth
- [docs/methodology.md](docs/methodology.md) — validation methodology and rigor
- [docs/results.md](docs/results.md) — results, metrics, and reproducibility notes

---

## About

Built by **Emmanuel Fletcher** — paramedic and self-taught engineer (12+ years IT) —
as the founding work of **Innovation Drive Line Corp (IDL.Co)**, an autonomous-vehicle
intelligence company (in formation), Newfoundland & Labrador, Canada.

This repository showcases the engineering and methodology. The proprietary system
itself is not published. For collaboration or employment inquiries, see the contact
in my GitHub profile.
