# Metrics

Every figure below is either a static measurement of the codebase itself
(reproducible by anyone who clones the repository) or a design-time
constant pulled directly from source. The point of this document is to
let a reader verify each number, not to accept it on faith — every row
says exactly what command or file produced it.

**Snapshot date** 2026-09-18

**Observation window** none — no logged runs of the system exist. Every
figure here is either a codebase-size measurement taken at the snapshot
date, or a design-time constant that has never been checked against
observed behaviour.

**Raw export** not applicable — no export exists, because no run has ever
been logged to a file. See Open TODO below.

## Sources

Design-time constants cited throughout the README and `docs/known-issues.md`
live in these files:

| Source file | What it defines |
|---|---|
| `csharp_server/MotionPlanValidator.cs` | Safe-height envelope, max function calls per plan |
| `csharp_server/Program.cs` | Per-target retry limit, Unity step timeout |
| `csharp_server/MotionPlanner.cs` | Motion-planner LLM request timeout |
| `csharp_server/SpatialPatternDesigner.cs` | 3D feasibility LLM request timeout |
| `csharp_server/SingleObjectTaskBuilder.cs` | Accepted measured-height range for stacking |
| `csharp_server/PatternDesigner.cs` | OpenAI/Gemini bitmap vote weighting, revision-round cap |
| `unity_project/Assets/Scripts/JsonExecutor.cs` | TCP position tolerance, motion confirmation timeout, base-exclusion radius, manual home-retry limit |
| `csharp_server/perception_server.py` | Scene refresh interval, HSV cube/domino size thresholds |

These are engineering assumptions the team chose before running the
system — not measured outcomes. The full table with values is in the
README's Measurement Basis section; it is not duplicated here to avoid
the two drifting out of sync.

## Scale

| Figure | Value | Derivation |
|---|---|---|
| `csharp_server/*.cs` files | 13 | `find csharp_server -maxdepth 1 -name '*.cs' \| wc -l` |
| `csharp_server/*.cs` total lines | 3,558 | `wc -l csharp_server/*.cs` |
| `perception_server.py` lines | 1,188 | `wc -l csharp_server/perception_server.py` |
| Flask HTTP routes | 6 (5 distinct paths) | `grep -c '@app.route' csharp_server/perception_server.py` |
| `unity_project/Assets/Scripts/*.cs` files | 8 | `find unity_project/Assets/Scripts -name '*.cs' \| wc -l` |
| `unity_project/Assets/Scripts/*.cs` total lines | 2,271 | `wc -l unity_project/Assets/Scripts/*.cs` |
| Development period | 2026-06-30 – 2026-09-18 | `git log --reverse --format=%ad --date=short \| head -1` and `git log -1 --format=%ad --date=short` |
| Commit count | 81 | `git rev-list --count HEAD` |

**What this figure does not claim.** File and line counts describe how
much code exists, not how well it works, how fast it runs, or how often a
command succeeds against it — that would need logged runtime data, which
doesn't exist yet (see Open TODO).

**Current versus cumulative.** These are current-state counts as of the
snapshot date. They will change as the project continues and should be
re-derived with the commands above rather than assumed stable.

## Interpretation

### "13 files" is not "13 things done well"

Line count and file count are cheap to produce and easy to misread as a
proxy for completeness or quality. They aren't — a file existing says
nothing about whether the behaviour it implements has ever been checked
against a real outcome. See the README's Threats to Validity section for
why even a passing `Verifier` check on one run doesn't establish a
success rate.

### No duration or latency number in this document is a measured outcome

Every timeout and interval constant cited under Sources is something the
team chose, not something the team observed happening. Reading, for
example, "180 s motion timeout" as "motions take about 180 s" would be a
misread — it is an upper bound the system enforces before giving up, not
an average or typical duration.

## Open TODO — what would turn this into a real metrics document

- A batch of repeated identical commands, logged with success/failure and
  elapsed time per step, to produce an actual success-rate and latency
  distribution.
- Detection accuracy against a labelled object set — the current
  YOLO/HSV detectors have never been scored against ground truth.
- A recorded count of protective/emergency-stop events during testing, if
  any occurred, since none is currently logged anywhere.

None of the above exists yet. This section exists so that gap stays
visible rather than silently absent from the document.
