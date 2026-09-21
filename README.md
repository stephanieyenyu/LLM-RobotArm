# LLM_RobotArm — Natural Language Control for a UR3e Robot Arm via LLM and Vision Perception

LLM_RobotArm lets a UR3e robot arm be operated with typed natural-language
commands. It was built as an undergraduate capstone project (專題) at
National Tsing Hua University, developed 2026-06-30 – 2026-09-18, by a team
of NTHU students working collectively without a fixed division of labour.
An Intel RealSense D435i camera watches the workspace in real time, an LLM
(OpenAI GPT-5, with Gemini cross-reviewing pattern layouts) turns a typed
Chinese command into a task, and a deterministic C# planner and safety
validator turn that task into UR3e motion executed through Unity.

Sending a single URScript command to a UR3e is not the hard part — the TCP
interface handles that directly. Three things took the work:

**Reconciling an LLM's non-deterministic output with a physical safety
envelope.** An LLM asked for joint angles or raw URScript can propose
something wrong or simply different on every call. The motion planner
therefore never lets the LLM emit a coordinate, a speed, or raw URScript —
it can only compose from a seven-function whitelist, and every resulting
plan is checked by a deterministic safety validator before a motor moves.

**Recovering from a step that half-fails without losing track of the
world.** A batch plan computed once and executed blindly compounds error:
if step one drifts, every later step still executes against the original,
now-stale scene. The system instead re-senses and re-verifies before and
after every single step, so a partial failure can be retried or replanned
against the real current state rather than an assumption.

**Getting two independent LLMs to agree on a bitmap layout without a human
arbitrating.** A single model's proposed pattern had no check on it. Every
`arrange_pattern` command now has OpenAI and Gemini independently propose a
candidate bitmap and cross-review each other's, with a weighted vote
deciding the result.

It contains one learned component whose behaviour is bounded: the LLM
never outputs raw URScript, joint angles, or arbitrary coordinates — every
plan it proposes is filtered through a whitelist and a deterministic
safety validator before anything physical happens.

## Demo

*Demo video: pending. A recorded walkthrough exists and is being added —
see `docs/known-issues.md` (B-1) for tracking.*

## What it does

**Turns a typed Chinese sentence into one of three concrete task types
before any coordinate math happens.** `CommandRouter` classifies each
command as `arrange_pattern`, `move_relative`, or `stack`; every
downstream layer works from that classification rather than re-parsing
free text.

**Builds every letter or shape pattern by having two LLMs argue over
it.** `PatternDesigner` asks OpenAI and Gemini to each independently
generate a candidate bitmap, has them cross-review one another's
candidate, and picks a result by a weighted vote (OpenAI 0.80 / Gemini
0.20) over up to two revision rounds.

**Never lets an LLM see a raw coordinate or emit raw URScript.**
`MotionPlanner` composes motion only from seven whitelisted functions
(`move_above`, `descend`, `grasp`, `release`, `lift`, `wait`, `go_home`);
`MotionPlanValidator` rejects any plan outside the whitelist, the 20-call
limit, or the 0.05–0.15 m safe-height envelope before Unity ever sees it.

**Re-senses the workspace before every single step, not once per
command.** Each step in a multi-step task triggers a fresh RealSense
scene snapshot and a fresh plan for just that step, so drift or a missed
grasp on step one doesn't propagate unnoticed into step five.

**Refuses to stack on a measured height it doesn't trust.** Stacking
targets use the source block's actual measured top-surface height
(`targetZ = reference.Z + source.Z`) rather than an assumed nominal
height, and rejects the step outright if the measurement falls outside
0.005–0.100 m rather than proceeding on a bad depth read.

**Keeps a virtual mirror of the workspace in Unity for visual
confirmation.** `SceneSyncer` refreshes a Unity-side copy of every
detected object whenever the system returns to idle, remapping the
perception system's right-handed QR frame into Unity's left-handed
coordinate space.

## Scope

| Component | Tech | Responsibility |
|---|---|---|
| Perception server | Python 3.10+, Flask, OpenCV, YOLO11n, RealSense SDK | Streams RealSense frames; detects HSV cubes/dominoes, ArUco QR anchors, and COCO objects; serves scene snapshots over HTTP |
| C# orchestrator | .NET 8 | Routes commands, runs the five-layer plan/execute/verify loop, calls OpenAI/Gemini, validates safety |
| Unity executor | Unity 2022.3.62f3, C# | Polls the plan file, converts whitelisted robot functions to URScript, drives the UR3e over TCP, mirrors the scene visually |

13 C# orchestrator source files (3,558 lines) · 1 perception server file
(1,188 lines) · 6 Flask HTTP routes across 5 paths · 8 Unity C# scripts
(2,271 lines)

These counts are reproducible directly from the repository: `wc -l
csharp_server/*.cs`, `wc -l csharp_server/perception_server.py`, `grep -c
'@app.route' csharp_server/perception_server.py`, and `wc -l
unity_project/Assets/Scripts/*.cs`.

Development period: 2026-06-30 – 2026-09-18 (81 commits).

This was a collective NTHU capstone effort. Commit authorship in this
repository does not reliably map to who wrote which part — contributors
shared machines during development — so responsibilities above are not
broken out by person.

## Test Setup and Success Criterion

All testing was performed on a single physical rig: one table with four
printed ArUco markers (QR1–QR4) defining the work plane, an Intel
RealSense D435i mounted overhead, and a UR3e arm (or URSim in simulation)
connected through the Teach Pendant's Remote Control interface. Testing
was carried out by the project team during development, not by an
independent evaluator, using whatever objects were on hand — the YOLO
COCO whitelist items, plus 2.5 cm HSV cubes and 5×2.5×2.5 cm dominoes —
rather than a fixed labelled dataset.

A run is considered successful when a typed command results in the UR3e
completing the corresponding motion end-to-end and `Verifier` confirms
the expected object arrangement, without manual intervention beyond what
the system does automatically (its own retry/replan loop).

**What this setup cannot validate**: success rate, accuracy, or timing
under varied lighting, a larger or different object set, repeated trials
of the same command, or any load beyond one operator issuing commands one
at a time. No batch of trials has been logged — see `docs/metrics.md`.

## Measurement Basis

| Measurement | Value | Nature |
|---|---|---|
| Safe height envelope | 0.05 – 0.15 m | design constant, not measured (`MotionPlanValidator.cs:78-104`) |
| Max function calls per plan | 20 | design constant, not measured (`MotionPlanValidator.cs:86`) |
| Per-target retry limit | 1 | design constant, not measured (`Program.cs:226`) |
| Motion planner LLM timeout | 180 s | design constant, not measured (`MotionPlanner.cs:10`) |
| 3D feasibility LLM timeout | 300 s | design constant, not measured (`SpatialPatternDesigner.cs:11`) |
| Unity step timeout | 600 s | design constant, not measured (`Program.cs:79`) |
| Accepted measured-height range for stacking | 0.005 – 0.100 m | design constant, not measured (`SingleObjectTaskBuilder.cs:11-12`) |
| TCP position tolerance | 0.012 m | design constant, not measured (`JsonExecutor.cs:100`) |
| Motion confirmation timeout | 180 s | design constant, not measured (`JsonExecutor.cs:99`) |
| Base-exclusion radius | 0.16 m | design constant, not measured (`JsonExecutor.cs:95`) |
| Manual home-retry limit after a protective stop | 1 | design constant, not measured (`JsonExecutor.cs:106`) |
| OpenAI/Gemini bitmap vote weighting | 0.80 / 0.20 | design constant, not measured (`PatternDesigner.cs:11-12`) |
| Bitmap generation revision rounds | 2 | design constant, not measured (`PatternDesigner.cs:10`) |
| Scene refresh interval | 200 ms | design constant, not measured (`perception_server.py`) |

**None of these are empirical outcomes.** Every row above is a value the
team chose before running the system, not a value derived from measured
behaviour. See `docs/metrics.md` for a fuller accounting of what has and
hasn't actually been measured.

**One documented internal inconsistency.** The project's written report
states the per-target retry limit as 2; the shipped code
(`Program.cs:226`) sets `MAX_RETRY = 1`. This README follows the code.
See `docs/known-issues.md` D-2.

## Problem Statement

An operator wants to describe a task in natural Chinese — 「排 H」,
「把黃色方塊往前移 5 公分」 — and have a UR3e arm carry it out. Asking an LLM
directly for joint angles or URScript fails for a specific reason: an
LLM's output is not guaranteed correct or repeatable, and a wrong joint
command executed on real hardware can damage the arm, the workspace, or
whoever is standing nearby. Restricting the LLM to a small set of
pre-validated actions removes that danger, but introduces a different
problem: the LLM must still turn an open-ended sentence into a sequence
of those actions, coordinated with a perception system that has its own
real error (camera calibration drift, depth noise, occlusion), across a
task that must survive a step going wrong partway through without
leaving the arm, the object, or the system's internal state in an
inconsistent place.

The problem this project addresses is: how to let an LLM plan robot
motion from natural language while keeping every physical consequence of
that plan bounded, checked, and recoverable — without falling back to
either a fixed hard-coded task list (which isn't natural-language control
at all) or an unconstrained LLM-to-hardware pipeline (which isn't safe).

## System Architecture

```
Unity UI (command input)
   ↓  StreamingAssets/user_input.txt
csharp_server (dotnet)
   ↓  HTTP GET localhost:5000/scene
perception_server (Python + Flask)
   ├─ persistent RealSense stream
   ├─ YOLO11n (COCO objects) + HSV cubes/dominoes + ArUco QR anchors
   └─ scene refreshed every 200 ms, returns 3D world coordinates
   ↓
LLM CommandRouter (arrange_pattern / move_relative / stack)
   ├─ PatternDesigner: OpenAI + Gemini independently generate a bitmap, cross-review each other
   └─ SingleObjectTaskBuilder: direction/distance or stack target → actual coordinates
   ↓
LLM MotionPlanner → MotionPlanValidator
   ↓  StreamingAssets/current_step.json (robot function sequence)
Unity JsonExecutor (high-level function → URScript)
   ↓  TCP 30002 URScript
UR3e
```

The orchestrator runs a five-layer closed loop per command
(`Program.cs:12-19`):

| Layer | Component | Produces |
|---|---|---|
| 1 | PatternDesigner | CanonicalPattern |
| 2 | LayoutRealizer | List\<TargetCell\> |
| 3 | TaskAssigner | one Assignment per step |
| 4A | MotionPlanner | LLM-composed robot functions |
| 4B | MotionPlanValidator / Unity | safety-checked execution |
| 5 | Verifier | retry / replan / abort decision |

Splitting perception, planning, and execution into three separate
processes — a Python Flask service, a .NET orchestrator, and a Unity
executor — rather than one monolith lets the RealSense stream run
continuously regardless of whether a plan is mid-execution, and lets the
LLM-facing planning code be replaced or tested without touching the UR3e
TCP client. The accepted cost is three separate runtimes that must be
started in the right order for the system to work at all, plus an HTTP-
and file-based IPC layer (`/scene`, `current_step.json`, `step_done.json`)
in place of in-process calls.

See `docs/architecture.md` for the full component/IPC breakdown and
`docs/llm_motion_planner.md` for the Layer 4A whitelist and safety-limit
deep dive.

## Design

### Fixed pick-and-place sequences could not use what the LLM could actually plan

The early system executed the same fixed pick-and-place motion sequence
for every task, regardless of what the LLM could contribute — easy to
implement, but it never used the LLM's planning ability at all. The
current `MotionPlanner` instead asks the LLM to compose a motion from a
small robot-function API for every step. The accepted cost is that every
LLM-composed plan must pass `MotionPlanValidator` before a single motor
command is sent, and the LLM itself is restricted to seven whitelisted
functions — it can never request its own coordinate, speed, or
acceleration.

### Sensing once per command could not survive an error at step one

The traditional open-loop approach is Sense → Plan All → Execute All: if
the first action drifts, every later action still executes against the
original, now-stale scene, compounding the error. This system instead
runs Sense → Plan One Step → Execute → Sense Again → Verify → Plan Next
Step. The accepted cost is that every step re-senses and re-plans, which
is slower per command than executing a pre-computed batch, in exchange
for tolerating drift and partial failure mid-task.

### A single model's bitmap was not trustworthy enough to place blocks against

Generating a pattern with one LLM call had no check on what it proposed —
whatever that one model returned was accepted. `PatternDesigner` now has
OpenAI and Gemini each independently generate a candidate bitmap, cross-
review one another's candidate, and resolves the result by a weighted
vote (OpenAI 0.80 / Gemini 0.20) over up to two revision rounds. The
accepted cost is two LLM providers and API keys required, more API calls
per pattern command, and a command that fails outright if no bitmap is
accepted within those two rounds.

### A fixed block height could not survive being measured on a real table

Assuming a nominal fixed block height for stacking targets doesn't
reflect the block actually sitting on the table. Stacking now uses the
source block's measured top-surface height from RealSense depth
(`targetZ = reference.Z + source.Z`), accepted only within 0.005–0.100 m.
The accepted cost is that a reading outside that band is rejected outright
and forces a fresh scene scan rather than proceeding on an untrustworthy
depth value.

### A fixed delay could not tell "still moving" from "something went wrong"

Advancing to the next motion after a fixed delay elapsed could not
distinguish a slow-but-fine motion from a stalled or faulted one — as the
code itself notes, "do not advance merely because a fixed delay
elapsed." Every motion is now confirmed against the UR secondary-
interface TCP feedback, within a 0.012 m tolerance, before advancing, up
to a 180 s timeout. The accepted cost is that after a protective or
emergency stop, the interrupted motion is never automatically resent —
only a single manual return-to-home retry is permitted — trading
throughput for never repeating a motion blind after a safety stop.

### Pliers detection could not clear the confidence bar on the real workspace

The perception module was written to load a custom-trained pliers model
alongside YOLO11n. In practice, the domain gap between the training
dataset (Roboflow) and the real overhead workspace scene kept confidence
under 0.06 — unusable. Pliers detection is currently disabled
(`PLIERS_MODEL = None`), deferred until real photos of the workspace are
available to fine-tune on. The accepted cost is that pliers are not
detectable at all right now, and the module's own docstring is currently
stale about this (tracked in `docs/known-issues.md` D-1).

## Evaluation

No controlled batch of trials has been run and logged (see
`docs/metrics.md`), so this section reports what has and hasn't been
observed rather than a success-rate table.

**Observed working end-to-end**: `arrange_pattern`, `move_relative`, and
`stack` commands have each been run against the physical rig and
completed, including the closed-loop retry path (falling back to an
untried same-color/same-shape block on repeated failure) and the
dual-model bitmap cross-review.

**Never observed to complete**: pliers detection — the model is disabled,
so no pliers-related plan has ever been attempted, successfully or
otherwise. Multi-layer 3D voxel stacking beyond the current single-
depth-plane implementation has not been evaluated for arm reachability or
inter-column support; `SpatialPatternDesigner`'s feasibility check covers
glyph geometry, not whether the arm can physically reach every resulting
cell.

**What the demo video does and doesn't substitute for**: a recorded
walkthrough exists (pending — see Demo) and shows the pipeline working on
one run. It is evidence the system can work, not evidence of how often it
does.

## Threats to Validity

**External validity.** All testing used one table, one lighting
condition, and one physical rig. Nothing here shows the behaviour
generalizes to a different workspace, camera mount height, or lighting.

**Construct validity.** "Task completed" is judged by `Verifier`'s own
position/shape check against the same camera that placed the object — a
systematic camera bias would still pass `Verifier` while being wrong in
the real-world frame.

**Internal validity.** Every result blends perception, LLM planning, and
physical execution in one closed loop. A failure can't currently be
attributed to a single layer without additional per-layer logging, which
doesn't exist.

**Instrumentation.** There is no logging pipeline. `Console.WriteLine`
output (e.g. `Program.cs:490`'s final match-count print) is the only
record of a run, and it isn't persisted — there is nothing today that
would let anyone go verify a past run's numbers.

**Configuration.** Two failure modes fail silently rather than loudly:
YOLO/pliers `.pt` model weight files are gitignored and not distributed
with this repository, so a fresh clone with no weights added will simply
detect nothing, without an obvious runtime error pointing at the cause
(`docs/known-issues.md` C-1); and two root-level Unity `Packages/`/
`ProjectSettings/` folders duplicate — and diverge from —
`unity_project/`'s (missing `com.unity.nuget.newtonsoft-json`), so
opening Unity at the repository root instead of `unity_project/` produces
a project that looks valid but isn't (`docs/known-issues.md` C-2).

## Open Problems

**Does two-model cross-review actually reduce bitmap non-determinism, or
just relocate it?** `arrange_pattern` for the same command can produce a
different bitmap across runs, because the layout is LLM-generated rather
than deterministically retrieved. Whether the 0.80/0.20 vote weighting is
meaningfully narrowing that variance, versus the two-round revision cap
simply capping how much variance is allowed through, hasn't been
measured — answering it needs a batch of repeated identical commands with
logged bitmap output, which doesn't currently exist.

**How far from optimal are the greedy packing and assignment
heuristics?** Domino packing (`LayoutRealizer`) and source-to-target
assignment (`TaskAssigner`) both use greedy heuristics with no
optimality guarantee. Whether the gap to an optimal assignment matters in
practice at this cell count and object scale, or is negligible, is open.

**Can arm reachability and inter-column support be checked with the same
deterministic-validator approach already used for 2D safety?** The 3D
voxel prototype (`SpatialPatternDesigner`) is currently constrained to a
single Y/depth row. Extending it to true multi-depth voxel structures
raises a question this project hasn't answered: whether reachability and
support can be validated the same deterministic way as the existing
height/whitelist checks, or need a different kind of check entirely.

All three questions above are also bounded by the same hardware: one
UR3e (or URSim), one RealSense D435i, one table. Any answer this project
could produce would need to be re-checked on a different rig before it
generalizes — this repository's own findings are a starting point for the
next iteration, not a final measurement (see Threats to Validity). The
next concrete step the team wants to take is collecting the logged,
repeated-trial data described in `docs/metrics.md`'s open TODO, so these
three questions can be asked with real numbers instead of design
intuition.

## Repository Layout

```
LLM_RobotArm/
├── csharp_server/                   .NET 8 orchestrator + Python perception server
│   ├── perception_server.py         1,188 lines — RealSense stream, YOLO11n, HSV cube/domino & ArUco detection, 6 Flask routes
│   ├── Program.cs                   995 lines — five-layer plan/execute/verify orchestrator loop
│   ├── PatternDesigner.cs           426 lines — Layer 1, dual-model (OpenAI/Gemini) bitmap generation + cross-review
│   ├── SpatialPatternDesigner.cs    337 lines — 3D voxel glyph feasibility (single depth plane)
│   ├── Verifier.cs                  302 lines — Layer 5, post-step scene verification
│   ├── SingleObjectTaskBuilder.cs   224 lines — move_relative / stack coordinate math
│   ├── MotionPlanValidator.cs       180 lines — Layer 4B, deterministic safety gate
│   ├── LayeredTypes.cs              172 lines — shared record/class types between layers
│   ├── BitmapParser.cs              169 lines — LLM string-array bitmap → int[,]
│   ├── CommandRouter.cs             165 lines — classifies a command into arrange_pattern/move_relative/stack
│   ├── TaskAssigner.cs              164 lines — Layer 3, greedy supply-to-target assignment
│   ├── LayoutRealizer.cs            157 lines — Layer 2, bitmap → world-coordinate targets + domino packing
│   ├── MotionPlanner.cs             155 lines — Layer 4A, LLM robot-function composition
│   ├── RobotPlan.cs                 112 lines — plan / SceneObject data classes
│   ├── QRcode/                      4 printable ArUco markers (aruco_1–4.png)
│   ├── csharp_server.csproj         .NET 8, OnnxRuntime, OpenAI SDK, OpenCvSharp4, ZXing.Net
│   └── requirements.txt             Python dependencies for perception_server.py
├── unity_project/Assets/Scripts/    Unity 2022.3.62f3 executor
│   ├── JsonExecutor.cs              796 lines — Layer 4 executor, function sequence → URScript
│   ├── SceneSyncer.cs               342 lines — snapshot sync + QR-frame↔Unity coordinate remap
│   ├── URPackageListener.cs         287 lines — UR3e TCP client (port 30002)
│   ├── SyncGripper.cs               207 lines — parents nearest cube to gripper on grasp/release
│   ├── RobotArm.cs                  203 lines — joint transforms for the UR3 model
│   ├── UIManager.cs                 183 lines — command input UI, plan-update watcher
│   ├── Util.cs                      129 lines — mesh/geometry helpers
│   └── URUtil.cs                    124 lines — byte-array↔struct marshalling
├── docs/
│   ├── architecture.md              full component/IPC/coordinate-frame breakdown
│   ├── known-issues.md              categorized known issues
│   ├── metrics.md                   codebase scale figures + open measurement TODOs
│   └── llm_motion_planner.md        Layer 4A whitelist + safety-limit deep dive
└── README.md
```

## Tech Stack

**Back end**  C# · .NET 8 · Microsoft.ML.OnnxRuntime 1.27.0 · OpenAI SDK 2.11.0 · OpenCvSharp4 4.13.0 · ZXing.Net 0.16.11

**Perception**  Python 3.10+ · Flask · OpenCV · Ultralytics YOLO11n · pyrealsense2

**Simulation / execution**  Unity 2022.3.62f3 · C#

**LLMs**  OpenAI GPT-5 · Google Gemini (`gemini-3.1-flash-lite` default)

**Hardware**  Intel RealSense D435i · UR3e or URSim · 4× printable ArUco markers (Dict4X4_50)

## Running Locally

**Terminal 1** (perception):
```powershell
cd csharp_server
yolo11_env\Scripts\python.exe perception_server.py
```

**Terminal 2** (orchestrator):
```powershell
cd csharp_server
dotnet run
```

**Unity**: open `unity_project` in Unity Hub → Play → set the Executor's
`Ur IP` field to the UR3e's IP address.

**Debug**: `http://localhost:5000/debug/live` in a browser shows the live
detection feed.

Prerequisites:
- .NET SDK 8+
- Python 3.10+, using the `csharp_server/yolo11_env` venv, with
  dependencies from `csharp_server/requirements.txt`
- Unity 2022.3.62f3
- Intel RealSense D435i, USB3, connected directly (not through a hub)
- `OPENAI_API_KEY` and `GEMINI_API_KEY` set via `setx` in PowerShell, then
  PowerShell restarted
- Optional `GEMINI_MODEL` (defaults to `gemini-3.1-flash-lite`, chosen for
  its free tier)
- A UR3e or URSim, Teach Pendant switched to Remote Control, TCP Z offset
  set to 0.170, speed slider at 100%
- Four printed ArUco markers on the table (QR1 bottom-left, QR2
  bottom-right, QR3 top-left, QR4 top-right)

**`yolo11n.pt` does not need to be sourced manually.** `perception_server.py`
loads it via `YOLO(str(BASE_DIR / "yolo11n.pt"))`; when that file is
missing, `ultralytics` downloads the standard COCO checkpoint from the
Ultralytics GitHub releases straight into that path on first run —
confirmed by testing the same load pattern against an empty directory.
An internet connection is required the first time only. Pliers detection
has no equivalent path: that was a custom-trained model, is not part of
the public model zoo, and is disabled in code regardless
(`docs/known-issues.md` C-1).

**A second running instance will not fail loudly.** `JsonExecutor` guards
against a duplicate command owner, but a stale Unity Play session left
running from a previous test, or a second `dotnet run` pointed at the
same `StreamingAssets` files, can still leave two processes racing to
read the same step file. Restart both terminals together rather than
assuming a fresh `dotnet run` alone is enough.

## Author

NTHU (National Tsing Hua University) undergraduate capstone project
(專題), developed 2026-06-30 – 2026-09-18.

Team: 林彥妤, 張以昕, 于庭黃, and one additional contributor identified in
this repository's history only by student ID (`112034053`).

The work was genuinely collaborative; commit authorship in this
repository does not reliably indicate who wrote which part, since
contributors shared machines during development. No course name or
advisor is recorded in this repository.
