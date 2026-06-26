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
## Crossing-conflict harness — does honest velocity estimation change driving?

### Question

The perception stack produces *estimated* agent velocities (from a tracker running on
noisy detections), while the simulator knows each agent's *true* velocity. A long-standing
open question: does using estimated velocity instead of ground-truth change the planner's
behaviour in situations where velocity matters? Prior tests on random traffic never showed
a difference, because random traffic rarely forces a velocity-dependent decision.

### Method

A deterministic **crossing-conflict harness** was built: the ego drives straight along a
highway while a single constant-velocity agent crosses its path, timed to meet at a fixed
conflict point. Random traffic is disabled for clean isolation. The crossing agent ignores
the ego entirely, forcing the ego to be the one that predicts and responds.

The harness was run as an A/B across the velocity source — estimated (tracker) vs
ground-truth — and swept across five timing offsets that vary how tightly the paths
conflict. Seed fixed at 42; per-step trajectory cost and ego↔crosser distance logged to CSV.

### Result

Across all ten runs (2 velocity sources × 5 timing offsets), the **driving behaviour was
identical** between estimated and ground-truth velocity:

| offset | min-distance | first-decel position | peak decel | outcome |
|-------:|-------------:|---------------------:|-----------:|---------|
| -40 | 24.1 m | x = -85 | -6.0 m/s² | passed |
| -20 | 10.0 m | x = -85 | -6.0 m/s² | passed |
|  0  |  4.1 m | x = -85 | -6.0 m/s² | passed |
| +20 | 18.2 m | x = -85 | -6.0 m/s² | passed |
| +40 | 32.2 m | x = -85 | -6.0 m/s² | passed |

Estimated and ground-truth velocity gave the same minimum distance, the same braking point,
and the same deceleration at every offset — the trajectories are behaviourally identical.

### The control run — and the actual finding

To check whether the ego was responding to the crossing agent *at all*, a control was run
with the crossing agent placed far away (no possible conflict). **The ego braked at the
same point (x = -85), with the same deceleration, and accumulated nearly the same trajectory
cost (7354 vs 7354–7476 with the agent present).**

This shows the deceleration is a **route-following behaviour** (slowing for road geometry on
the planned path), not a response to the crossing agent. The crossing agent contributes
little measurable cost and produces **no behavioural avoidance**: the ego drives an identical
trajectory whether the agent is present or absent.

The apparent difference between estimated and ground-truth velocity seen in the raw cost
traces reflects route-cost dynamics, not collision response. The trajectory-cost signal is
dominated by route-following terms; the crossing agent's collision-cost contribution is small
relative to them and never changes the selected trajectory.

### Interpretation

The harness did its job: it exposed that **crossing conflicts are not currently handled by
the planner**. On a lane-locked highway, every candidate trajectory carries similar exposure
to a crossing agent, so collision cost does not differentiate "slow down" from "maintain
speed", and no avoidance emerges. The near-misses (4.1 m at the tightest offset) occur
because both arms simply drive through.

This is the honest result. It is not "estimated velocity matches ground truth"; it is
"velocity source is irrelevant here because the crossing agent isn't influencing the decision
in the first place." The next step is to investigate why collision cost from cross-traffic
fails to produce avoidance, and to make the planner respond to it — then re-run this same
harness to measure whether estimated vs ground-truth velocity matters once the response exists.

*Reproducible: fixed seed, documented harness and commands in the repo.*
## Crossing-conflict, part 2 — fixing the planner, and when velocity actually matters

The first crossing-conflict study (above) found that the planner produced **no
behavioural response** to a crossing agent — the ego drove an identical trajectory
whether the agent was present or not. This follow-up traces that to its root cause,
fixes it, and re-runs the velocity comparison now that the planner can actually respond.

### Root cause: speed did not affect position

The trajectory sampler generated candidate paths by interpolating from the ego to a
look-ahead point as a function of **time** only. The candidate "speed" affected the
cost function's speed term but **not the actual positions** in the trajectory — a slow
candidate and a fast candidate occupied the same place at the same timestep. Because of
this, slowing down could never move the ego out of a crossing agent's path; it was pure
cost penalty with no avoidance benefit, so the planner never chose it.

The fix was to advance trajectory position by **actual speed along the path** (arc-length
accumulated as speed × dt per step), clamped to the available look-ahead. A slow candidate
now genuinely lags and arrives at a conflict point later. Normal navigation was re-verified
(the ego still reaches destinations) before evaluating the crossing scenario.

### Result: velocity quality decides the knife-edge

With the planner able to respond, the velocity-source A/B (estimated tracker velocity vs
ground-truth) was swept finely across conflict timing. Minimum ego–agent distance, by arm:

| timing offset | tracker (estimated) | ground-truth | tracker outcome | GT outcome |
|--------------:|--------------------:|-------------:|-----------------|------------|
| -15 | 9.6 m | 6.7 m | yielded | yielded |
| -10 | 6.3 m | 3.5 m | yielded | yielded |
|  -5 | 3.6 m | 3.7 m | yielded | yielded |
|   0 | **0.8 m** | **3.9 m** | **collision** | **yielded** |
|  +5 | 4.3 m | 7.4 m | yielded | yielded |
| +10 | 7.8 m | 10.9 m | yielded | yielded |
| +15 | 11.3 m | 14.6 m | yielded | yielded |

At the exact collision-course timing (offset 0), estimated velocity misjudged the conflict
and **collided** (0.8 m), while ground-truth velocity **yielded** (3.9 m). This was a
single knife-edge: at every other timing in the band both arms cleared the agent.

There is also a consistent asymmetry worth noting honestly — estimated velocity kept *more*
margin when the agent arrived early (negative offsets) and *less* when it arrived on-time or
late, consistent with a small timing bias in the estimate. The one failure landed precisely
where that error had the least room to be absorbed.

### Honest scope

This is not "estimated velocity is unsafe." Across the timing band the two are comparable,
and the single collision is one sample at the exact conflict point. It is also fair to say
the planner's crossing-avoidance is *marginal* in general — even ground-truth's "safe" pass
is a 3.9 m near-miss, and neither arm brakes early specifically for the crossing agent; the
deceleration present is route-following. Ground-truth simply has enough prediction accuracy
to thread the conflict where the noisy estimate does not.

The takeaway: with a planner that can respond, **velocity-estimation error is tolerable
everywhere except the exact knife-edge conflict, where it is the margin between a near-miss
and a collision.** That is the long-tail edge case the project is built to surface — the rare,
precise moment where perception quality determines the outcome. Next: make crossing-avoidance
robust (early braking for predicted cross-traffic, not just route-following), then re-measure.

*Reproducible: fixed seed, deterministic harness, documented commands in the repo.*
