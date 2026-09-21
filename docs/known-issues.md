# Known Issues

Issues here are not deleted once resolved — the classification itself is
the point: it records what was checked and found not to be a problem,
what's still unverified, what's an actual defect, and what's just a
documentation gap, so the same question doesn't get re-investigated from
scratch later.

Snapshot: 2026-09-18, 81 commits, repository at
`stephanieyenyu/LLM-RobotArm`. Updated 2026-09-21 (C-1, C-2, D-1, D-2, D-4).

| ID | Issue | Class | Disposition |
|---|---|---|---|
| A-1 | Fine-angle grasp correction disabled | Not a defect | No action |
| A-2 | No automatic motion resume after a protective/emergency stop | Not a defect | No action |
| A-3 | Duplicate `JsonExecutor` instances are blocked | Not a defect | No action |
| B-1 | Demo video not yet linked in README | Unverified | Open |
| B-2 | Greedy domino packing / source assignment optimality gap | Unverified | Open |
| B-3 | 3D voxel prototype reachability/support not evaluated | Unverified | Open |
| C-1 | Model `.pt` weight files not distributed with the repository | Defect | **Fixed** — documented, less severe than thought |
| C-2 | Stray duplicate Unity project files at repository root | Design limitation | Accepted, not fixed |
| D-1 | `perception_server.py` docstring claims a pliers model is loaded | Documentation | **Fixed** |
| D-2 | Written report and code disagree on the per-target retry limit | Documentation | No action (report removed from repo) |
| D-3 | Old README claimed a single 5 cm cube shape; code defines two shapes | Documentation | No action (fixed in this rewrite) |
| D-4 | No `requirements.txt` for the Python perception server | Documentation | **Fixed** |
| D-5 | Legacy "Part A" single-image detection prototype | Documentation | No action (superseded, recorded here) |
| D-6 | Old README's file inventory omitted 9 of 21 source files | Documentation | No action (fixed in this rewrite) |
| D-7 | Old README cited a 120 s timeout that doesn't exist in the current code | Documentation | No action (fixed in this rewrite) |

---

### A-1 · Fine-angle grasp correction disabled

**Symptom.** The gripper only ever rotates to one of two fixed
orientations (0° or 90°), never a fine angle matched to a detected
object's skew.

**Location.** `unity_project/Assets/Scripts/JsonExecutor.cs:90-92,
711-716, 744-749` (`SKEW_SIGN` constant commented out; code comment:
"Disable camera-estimated fine skew. It was causing noisy wrist
rotation.").

**Assessment.** Intentional. Fine-angle correction was tried and produced
noisier wrist motion than the two-orientation fallback, so it was
deliberately disabled rather than left half-working.

**Verified.** Yes, by reading the commented-out code and its adjacent
comment directly.

---

### A-2 · No automatic motion resume after a protective/emergency stop

**Symptom.** If the UR3e hits a protective or emergency stop mid-motion,
the interrupted motion is not automatically resent.

**Location.** `unity_project/Assets/Scripts/JsonExecutor.cs:106`
(`MAX_MANUAL_HOME_RETRIES = 1`), with the surrounding logic at lines
485, 580, 649.

**Assessment.** Intentional, safety-motivated — see the README's Design
section ("A fixed delay could not tell 'still moving' from 'something
went wrong'"). Only a single manual return-to-home retry is permitted;
the original motion itself is never resent blind.

**Verified.** Yes.

---

### A-3 · Duplicate `JsonExecutor` instances are blocked

**Symptom.** Running a second `JsonExecutor` in the same Unity scene logs
`"[Executor] Duplicate JsonExecutor disabled; only one UR command owner
is allowed."` instead of running both.

**Location.** `unity_project/Assets/Scripts/JsonExecutor.cs:130-134`.

**Assessment.** Intentional. A second executor would open a second UR
connection and could send conflicting commands concurrently.

**Verified.** Yes.

---

### B-1 · Demo video not yet linked in README

**Symptom.** The README's Demo section has no embedded video.

**Observation.** A recorded walkthrough exists (per the project team) but
has not yet been added to the repository or linked.

**Check.** Once provided, embed it in the README Demo section with a
one-line italic caption describing the pipeline shown and the clip's
length, then close this issue.

---

### B-2 · Greedy domino packing / source assignment optimality gap

**Symptom.** None observed directly — flagged from reading the algorithm,
not from a failed run.

**Location.** `csharp_server/LayoutRealizer.cs` (domino packing),
`csharp_server/TaskAssigner.cs` (supply-to-target assignment).

**Hypothesis.** Both use greedy heuristics with no optimality guarantee.
At small cell counts the gap to optimal may be negligible; at larger
patterns it may not be.

**Impact.** Unknown — see the README's Open Problems section.

**Check.** Would need a batch of runs across a range of pattern sizes,
comparing greedy output against an exact or near-optimal solver, which
hasn't been done.

---

### B-3 · 3D voxel prototype reachability/support not evaluated

**Symptom.** None observed directly.

**Location.** `csharp_server/SpatialPatternDesigner.cs`.

**Observation.** The feasibility check here validates glyph geometry
(size, support, inventory) but not whether the arm can physically reach
every resulting cell, or whether upper layers are adequately supported
across columns.

**Impact.** Unknown — the prototype is currently limited to a single
Y/depth row, which sidesteps the harder multi-depth case rather than
solving it.

**Check.** Needs either a reachability solver integrated into the
feasibility check, or empirical testing across a range of generated
glyphs, neither of which exists yet.

---

### C-1 · Model `.pt` weight files not distributed with the repository

**Symptom.** A fresh clone of this repository has no `yolo11n.pt` on disk.

**Cause.** `.gitignore` excludes `*.pt`, and the file was never committed
or otherwise made available alongside the repository.

**Turned out less severe than first assessed.** `perception_server.py`
loads the model via `YOLO(str(BASE_DIR / "yolo11n.pt"))`. Tested directly:
running that exact call against a path where the file doesn't exist
causes `ultralytics` to download the standard COCO checkpoint from the
Ultralytics GitHub releases into that path automatically — confirmed by
running it against an empty directory and observing the download and a
successful load. This is not a documentation-only fix; it changes what
the defect actually is.

**What's still true.** `models/pliers.pt` has no equivalent path — it was
a custom-trained model never published to any model zoo, so there is
nothing for any tool to auto-fetch. This is moot in practice because
pliers detection is disabled in code regardless (see D-1).

**Fixed.** README's Running Locally section now states the auto-download
behaviour directly instead of listing the weight file as something to
manually source, and drops the now-inaccurate "fails silently" framing.

---

### C-2 · Stray duplicate Unity project files at repository root

**Symptom.** `Packages/` and `ProjectSettings/` exist both at the
repository root and inside `unity_project/`.

**Location.** Repository root (`./Packages/manifest.json`,
`./ProjectSettings/*`), tracked in git since commit `d6ad3f5`.

**Observation.** The root-level `ProjectVersion.txt` is identical to
`unity_project/ProjectSettings/ProjectVersion.txt`, but the root-level
`Packages/manifest.json` differs from — and is missing packages present
in — `unity_project/Packages/manifest.json`, including
`com.unity.nuget.newtonsoft-json`.

**Cause.** Most likely Unity Hub was pointed at the repository root
instead of `unity_project/` at some point, which silently generated a
second, incomplete project skeleton alongside the real one.

**Consequence.** Anyone who opens the repository root itself as a Unity
project (rather than `unity_project/`) gets a project that looks valid
but is missing at least one dependency the real project needs.

**Fix direction.** Confirm the root-level `Packages/`/`ProjectSettings/`
are unused, then remove them; keep `unity_project/` as the only real
Unity project root.

**Severity.** Low day-to-day (nobody currently opens the repo root as a
Unity project), but confusing to a new contributor and worth cleaning up.

**Accepted, not fixed — 2026-09-21.** Confirmed with the team: the
severity assessment above (low day-to-day impact) is accepted as-is, and
the folders are being left in place rather than removed.

---

### D-1 · `perception_server.py` docstring claims a pliers model is loaded

**Symptom.** The module's top-of-file docstring
(`perception_server.py:6`) still describes loading "YOLO11n (COCO 常見物件)
+ 自訓 pliers 模型" (a self-trained pliers model).

**Location.** Docstring at `perception_server.py:6`; actual behaviour at
`perception_server.py:184-186` (`PLIERS_MODEL = None`).

**Cause.** The docstring wasn't updated when pliers detection was
disabled (see the README's Design section, "Pliers detection could not
clear the confidence bar on the real workspace").

**Fix direction.** Update the docstring to match current behaviour, or
remove the pliers reference until the feature is re-enabled.

**Fixed.** The docstring now reads "自訓 pliers 模型因 domain gap 停用" (self-
trained pliers model disabled due to domain gap) instead of describing it
as loaded.

---

### D-2 · Written report and code disagree on the per-target retry limit

**Symptom.** `專題_整合最新進度.docx` §5 states the per-target retry limit
as 2 attempts; `csharp_server/Program.cs:226` sets `const int MAX_RETRY =
1`.

**Assessment.** The code is the source of truth; the report is stale
(or described an earlier version of the constant). This README follows
the code (see Measurement Basis).

**Fix direction.** Either update the report to say 1, or change
`MAX_RETRY` to 2 if 2 was actually intended — whichever reflects the
team's current intent.

**No action — 2026-09-21.** `專題_整合最新進度.docx` is removed from the
repository as of this commit (the README now stands alone as the
complete documentation). The comparison this entry describes is no
longer checkable by a reader of this repository; kept here as a
historical record of a discrepancy that existed while the report was
present.

---

### D-3 · Old README claimed a single 5 cm cube shape; code defines two shapes

**Symptom.** The previous README's "支援的物件" section described both HSV
shapes as "5cm 黃色立方體、5cm 黑色立方體" (5 cm cubes).

**Location.** `perception_server.py:91-92` actually defines a 2.5 × 2.5 ×
2.5 cm cube and a separate 5 × 2.5 × 2.5 cm domino shape.

**Disposition.** Fixed in this documentation rewrite — the current
README's "What it does" and Design sections describe both shapes
correctly. Recorded here for traceability.

---

### D-4 · No `requirements.txt` for the Python perception server

**Symptom.** `csharp_server/perception_server.py`'s dependencies (`flask`,
`opencv-python`, `numpy`, `pyrealsense2`, `ultralytics`) are only
discoverable by reading its imports; there is no committed dependency
manifest and no `.env.example` for its configuration.

**Fix direction.** Add a `requirements.txt` (or equivalent) pinning the
versions actually used in `csharp_server/yolo11_env`.

**Fixed, partially.** `csharp_server/requirements.txt` now lists the five
third-party packages verified against the actual imports (`opencv-python`,
`numpy`, `pyrealsense2`, `flask`, `ultralytics`). Unpinned — no lockfile
or `pip freeze` output from `yolo11_env` was available to pin honest
version numbers against, so pinning was left undone rather than guessed.

---

### D-5 · Legacy "Part A" single-image detection prototype

> Historical record. This describes an earlier prototype, not the
> current system. No action needed beyond keeping this entry.

**Symptom.** The previous README devoted roughly 250 of its 362 lines to
a separate "Part A" document describing an early single-shot detection
module that read one static image
(`csharp_server/images/test_scene.jpg`), detected 3 ArUco QR anchors plus
COCO objects via an ONNX YOLO model, and wrote
`outputs/detection_result.json` / `outputs/visual_result.jpg`.

**Assessment.** This has been superseded by the always-on
`perception_server.py` Flask service described in the current README,
which streams continuously from a live RealSense feed instead of reading
a single static image. The old text is not carried forward into the
active README.

**Why it's kept here rather than discarded.** One limitation Part A
identified remains true of the current system: YOLO cannot recognize
arbitrary custom objects (e.g. "red cube," "custom metal part") without
either training a custom model or adding open-vocabulary detection (e.g.
OWL-ViT, Grounding DINO) — simply editing a class-name lookup table
doesn't teach the model anything new, since that table only renames
already-trained class IDs. The current system still only recognizes the
YOLO COCO whitelist plus HSV-defined cubes/dominoes; general custom-object
recognition is still future work, exactly as Part A originally described.

---

### D-6 · Old README's file inventory omitted 9 of 21 source files

**Symptom.** The previous "檔案總覽" section listed 7 of 13
`csharp_server/*.cs` files and 5 of 8 Unity `Assets/Scripts/*.cs` files —
omitting `BitmapParser.cs`, `LayeredTypes.cs`, `LayoutRealizer.cs`,
`SpatialPatternDesigner.cs`, `TaskAssigner.cs`, `Verifier.cs`,
`RobotArm.cs`, `SceneSyncer.cs`, and `SyncGripper.cs`.

**Disposition.** Fixed in this rewrite — the current README's Repository
Layout section lists all 21 files. Recorded here for traceability.

---

### D-7 · Old README cited a 120 s timeout that doesn't exist in the current code

**Symptom.** The previous README's troubleshooting section said waiting
for `robot_plan.json` times out at 120 seconds.

**Assessment.** No 120 s constant exists anywhere in the current code.
The actual timeouts are `UNITY_STEP_TIMEOUT_SEC = 600`
(`Program.cs:79`), a 180 s LLM request timeout (`MotionPlanner.cs:10`),
and a 300 s LLM request timeout for 3D feasibility
(`SpatialPatternDesigner.cs:11`). Likely a stale reference to a value from
an earlier version of the code that has since changed.

**Disposition.** Fixed in this rewrite — the current README's Measurement
Basis table cites the actual constants. Recorded here for traceability.
