# Architecture

> In-depth view of the IDL.Co FSD simulator's closed-loop architecture.
> Proprietary source is not included; this describes the design.

## Closed loop

Each simulation step runs the full pipeline:

1. **World step** — the procedural world advances ego + traffic agents.
2. **BEV sensor model** — a bird's-eye-view grid is constructed around the ego,
   with spatial range-culling (KDTree index) so per-step cost stays bounded even
   with 1,000+ lanes in range.
3. **Perception** — a compact CNN (~1.05M parameters) infers:
   - a **road-occupancy map** (where it's drivable), and
   - an **agent-detection head** (where other road users are).
   Under the sensor-realistic regime, the model's *input* is a degraded
   observation (occlusion + noise) while training *targets* remain clean
   ground truth — the model must infer, not copy.
4. **Route planning** — an A* route over a graph of analytic grid crossings and
   lane segments, with type-aware costs and forward-progress re-sync.
5. **Motion planning** — candidate trajectories are sampled and scored by a
   multi-term cost function; the lowest-cost trajectory is selected.
6. **Control** — actuation follows the chosen trajectory.

## Procedural world

- ~441 km^2 road network: outer highway ring, inner ring, radial spokes,
  N-S highway, city grid (169x170), interchange ramps, industrial zone.
- ~1,786 lanes, 17 named destinations spanning near-interior to far-ring.
- Fully deterministic by seed — the same seed reproduces the same world and
  traffic, which is what makes controlled A/B testing possible.

## Perception

- Encoder-decoder CNN with two heads (occupancy, detection).
- Trained by imitation on recorded driving sessions.
- Evaluated under the same sensor-observation model used in deployment.

## Route graph

- Analytic computation of grid crossings to graph nodes.
- KDTree proximity stitching links nearby nodes across the network.
- A* with type-aware lane costs (highway vs. surface vs. ramp).
- Forward-progress re-sync prevents the follower from targeting a waypoint
  geometrically behind the ego near curves/intersections (a common stall mode
  on large graphs).

## Trajectory scoring (C++ hot path)

The cost function combines collision, occupancy, detection, speed-error,
lateral-deviation, and progress terms. The scoring loop — the dominant per-step
cost — is implemented in C++ via pybind11, with a pure-Python fallback for
portability.

## Velocity prediction (in progress)

A perception-only multi-object tracker:
- clusters dense per-cell detections into discrete objects,
- transforms detections from ego frame to a stable world frame using the ego
  pose (so a parked car reads ~0 velocity, not ego-relative motion),
- estimates per-track velocity by differencing world positions over time, and
- projects each track forward with a constant-velocity model for predictive
  avoidance.

The tracker is honest by design: it sees only what perception emits, never
ground-truth agent state.
