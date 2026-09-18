# LLM Motion Planner (Layer 4A)

Each `TaskAssigner` assignment is no longer expanded by Unity into a fixed
12-step sequence. It is instead handed to `MotionPlanner`, which asks the
LLM to compose motion from the following high-level robot functions:

- `move_above(location, height_m)`
- `descend(location)`
- `grasp()`
- `release()`
- `lift(location, height_m)`
- `wait(seconds)`
- `go_home()`

The LLM may not directly output raw URScript, arbitrary world coordinates,
speed, acceleration, or I/O commands. Once an `action_sequence` is
produced, `MotionPlanValidator` checks it against the whitelist and
parameter ranges before it is written to `current_step.json`. Unity's
`JsonExecutor` only interprets the functions above, using its own existing
coordinate conversion and URScript safety implementation.

If a planning call fails, returns non-JSON, or fails validation, the
system asks the LLM to correct itself up to three times before abandoning
the task. When `Verifier` returns `retry` or `replan`, its error
explanation is passed to the Motion Planner in the next round.

## Data Flow

`TaskAssigner → MotionPlanner → MotionPlanValidator → current_step.json → Unity Executor → UR3e → Verifier`

## Safety Limits

- Each plan may contain at most 20 function calls.
- Safe height is limited to 0.05–0.15 m.
- Motion order must follow: approach source, descend, grasp, lift,
  approach target, descend, release, lift, return home.
- All position parameters may only be `source` or `target`.
- Unity does not accept arbitrary URScript or world coordinates in the
  JSON.
