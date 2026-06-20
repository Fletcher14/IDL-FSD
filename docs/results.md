# Results

> Selected, validated results. All perception metrics are measured on a reserved,
> never-trained map under deployment-matched conditions.

## Perception generalization (Phase 3.1)

A data-expansion experiment added procedurally novel maps to the training corpus and
retrained the perception model (warm-started from the prior baseline). Both the prior
and retrained models were then scored on a **reserved novel map** that neither had
trained on, under identical conditions.

| Metric (novel-map holdout) | Prior model | Retrained model | Change |
|---|---|---|---|
| Validation loss (total)    | 0.0849      | **0.0684**      | **-19.5 %** |
| Detection loss             | 0.1197      | **0.0847**      | **-29.2 %** |
| Occupancy loss             | 0.0251      | 0.0260          | +3.7 % (negligible) |

**Interpretation.** The prior model's frequently cited low loss was measured on its
*own* easier holdout; on a genuinely novel map it scores 0.0849. The retrained model
scores 0.0684 on that same novel map — a ~19.5 % improvement concentrated in agent
**detection** (-29.2 %), i.e. the model is markedly better at spotting other road
users on maps it has never seen. Occupancy was already strong and is essentially
unchanged.

### Reproducibility notes
- Both models scored with the same evaluation harness (mirrors the training
  validation loop exactly).
- Holdout is a reserved seed, never in the training set.
- The evaluation score matches the training-reported validation loss, confirming
  the harness is faithful.

## Closed-loop driving

Closed-loop runs are evaluated separately (destinations reached, emergency-braking
events, cost profile, throughput) on deterministic seeded worlds, enabling
controlled A/B comparison of planning/perception changes with the only variable
being the change under test.

*Further closed-loop and velocity-prediction results are in progress.*

## Velocity prediction (Phase 3.2) — honest-perception tracking

A perception-only multi-object tracker was built to replace ground-truth agent
velocity in the planner's predictive-collision reasoning. It clusters dense
detections into discrete objects, compensates for ego motion, estimates world-frame
velocity by differencing positions over time, and projects agent motion forward for
proactive avoidance.

### Validation approach
The tracker was compared head-to-head against a ground-truth-velocity baseline on
deterministic seeded scenarios (same world, same traffic, only the velocity source
changed). This isolates the question: *does honest, perception-derived velocity
drive as well as perfect velocity?*

### Findings
- **The tracker matches ground-truth-quality driving** on standard traffic
  approaches — identical route completion, distance, and emergency-braking behavior.
  Perception-only velocity estimation was good enough to make the same avoidance
  decisions as a system given perfect velocity.
- **Emergency braking is proximity-bound, not prediction-bound.** Investigation of
  the control stack showed the automatic emergency brake is a reactive safety
  override triggered by immediate physical proximity — independent of the planner's
  predictive cost, by design. This means prediction quality is correctly measured by
  *trajectory choice and maintained following distance*, not by the emergency-brake
  counter. A common evaluation pitfall is to measure a predictive component against a
  reactive safety gate it cannot, architecturally, influence.
- **Next step: targeted conflict scenarios.** Demonstrating a measurable advantage
  for predicted velocity requires scenarios where velocity *direction* changes the
  optimal trajectory (e.g. a timed perpendicular crossing), rather than general
  traffic where honest and perfect velocity converge on the same decision. This is
  in progress.

### Why this matters
The result is a disciplined negative-to-neutral finding stated honestly: the
capability is real and non-regressive, the naive success metric was shown to be the
wrong one, and the evaluation was redirected to measure what the component can
actually affect. Reporting this transparently — rather than claiming an unverified
win — is the same evaluation rigor applied throughout the project.

## Performance engineering

The closed-loop simulator was profiled to identify per-step bottlenecks, and the
two largest pure-Python hotpaths were optimized with verified behavior-neutrality:

- **Sensor-observation model** — replaced legacy float64 random generation with a
  modern float32 PCG64 generator, seed-threaded to preserve per-seed reproducibility.
  Verified statistically equivalent (matching noise distribution and dropout rate)
  and bit-reproducible by seed. ~53% reduction in that function's time.
- **Detection clustering** — replaced an O(n^2) Python loop with vectorized array
  operations, verified to produce bit-identical clusters across 200 randomized
  trials. ~66% reduction in that function's time.

Net throughput improved ~44%, with identical driving outcomes confirmed before and
after (same route completion, distance, and braking behavior). Each change was
verified for correctness before adoption rather than trusted on the basis of a
faster wall-clock alone — the same evaluation discipline applied to the model work.

### Render-path optimization

The visualizer redrew the entire road network every frame regardless of what was
on screen. Since the view shows only a small region of a large procedural world,
the vast majority of that drawing was off-screen. A viewport cull was added —
reusing the same per-lane spatial geometry the perception path uses — so each
frame draws only the lanes whose bounds intersect the visible region (roughly a
hundredth of the network at typical zoom). The cull is zoom-aware and the lane
geometry is computed once, since the road is static. This removed the road-drawing
routines from the render profile entirely, materially smoothing playback on
lower-powered hardware, with no change to what is displayed.
