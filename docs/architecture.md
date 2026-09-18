# Architecture

This is the structural reference for the pipeline summarized in the
README's System Architecture section. For *why* the system is split this
way and what each split cost, see the README's Design section; this
document only covers *how* the pieces fit together.

## Pipeline

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

## Component Responsibilities

| Component | Tech | Responsibility |
|---|---|---|
| Perception server (`csharp_server/perception_server.py`) | Python 3.10+, Flask, OpenCV, YOLO11n, RealSense SDK | Streams RealSense frames continuously; detects HSV cubes/dominoes, ArUco QR anchors, and COCO objects; serves scene snapshots and a live debug view over HTTP |
| C# orchestrator (`csharp_server/*.cs`) | .NET 8 | Routes each command, runs the five-layer plan/execute/verify loop, calls OpenAI/Gemini, and validates every plan against the safety envelope before handing it to Unity |
| Unity executor (`unity_project/Assets/Scripts/*.cs`) | Unity 2022.3.62f3, C# | Polls the plan file, converts whitelisted robot functions into URScript, drives the UR3e over TCP, and mirrors the detected scene visually |

## Five-Layer Plan/Execute/Verify Loop

The orchestrator runs this loop once per command, and re-enters it once
per step within a multi-step command (`Program.cs:12-19`):

| Layer | Component | Produces |
|---|---|---|
| 1 | PatternDesigner | `CanonicalPattern` |
| 2 | LayoutRealizer | `List<TargetCell>` |
| 3 | TaskAssigner | one `Assignment` per step |
| 4A | MotionPlanner | LLM-composed robot functions |
| 4B | MotionPlanValidator / Unity | safety-checked execution |
| 5 | Verifier | retry / replan / abort decision |

Layer 4A is documented in full in `docs/llm_motion_planner.md` (the
robot-function whitelist, the safety limits `MotionPlanValidator`
enforces, and the retry/replan contract) — it is not repeated here.

## Coordinate Frames

The perception system and Unity use different handedness, and the
mapping between them is fixed in `SceneSyncer.cs`:

| | Perception (QR frame) | Unity |
|---|---|---|
| Handedness | right-handed | left-handed |
| Horizontal axis 1 | X (QR1 → QR2) | X |
| Horizontal axis 2 / depth | Y (QR1 → QR3) | Z |
| Height | Z | Y |

A perception-frame point `(x, y, z)` becomes Unity local `(x, z, y)`.
Separately, `unity_project/Assets/Scripts/JsonExecutor.cs` converts QR-frame
coordinates into the UR3e's own base frame using per-site calibration
constants (`QR1_X`, `QR1_Y`, `QR1_Z`) that must be re-measured with the
Teach Pendant whenever the table or the ArUco markers move — see the
README's Running Locally section.

## Inter-Process Communication

| Channel | Direction | Purpose |
|---|---|---|
| `StreamingAssets/user_input.txt` | Unity → csharp_server | the typed command |
| `GET /scene` | csharp_server → perception_server | full detection snapshot |
| `GET/POST /scene/mode` | csharp_server ↔ perception_server | idle/executing handshake so Unity can freeze the display mid-motion |
| `GET /health` | operator → perception_server | `frames_processed`, `detect_ms`, staleness — first place to check if detection looks stuck |
| `GET /debug/live`, `GET /debug/frame` | operator → perception_server | live annotated camera view in a browser |
| `StreamingAssets/current_step.json` | csharp_server → Unity | one validated step (robot function sequence) |
| `step_done.json` | Unity → csharp_server | execution result of that step |

## Why this split, and its cost

Perception, planning, and execution run as three separate processes — a
Python Flask service, a .NET orchestrator, and a Unity executor — rather
than one monolith. That lets the RealSense stream run continuously
regardless of whether a plan is mid-execution (the perception server
keeps sampling even while Unity is moving the arm), and lets the
LLM-facing planning code be modified or tested without touching the UR3e
TCP client at all.

The accepted cost is real: all three runtimes must be started, and
started in a working order, for the system to do anything (see the
README's Running Locally section for the exact sequence); and
coordination happens through an HTTP + file-based IPC layer instead of
in-process calls, which is easy to leave partially running by accident —
`docs/known-issues.md` covers what that looks like in practice.

## See also

- `docs/llm_motion_planner.md` — Layer 4A whitelist and safety-limit deep dive
- `docs/known-issues.md` — C-2 covers stray duplicate Unity project files at the repository root that can make this architecture look different from how it's actually laid out
- `README.md` — Design section, for why each of these structural choices was made and what it cost
