# Meridian Live Stitch Agent Task List

Source: `docs/MLS_MVP_PRD.md`.

Purpose: issue-ready work list for implementation agents. Each task is a vertical slice: it should produce visible behavior plus durable state/data/tests where relevant. Prefer assigning one task at a time unless dependencies clearly allow parallel work.

Working kanban: `docs/MLS_BUILD_KANBAN.md`.

Status: draft. Ready to publish to an issue tracker once the team confirms granularity and dependencies.

## Review Questions

- Is this too coarse or too fine for agent work?
- Are the dependency relationships correct?
- Are the HITL tasks the right ones to require operator/design/hardware review?
- Should Phase 1 tasks be implemented before publishing Phase 2/3 tasks?

## Dependency Map

1. Startup, Resume, and Recovery Shell - AFK - Blocked by: None
2. Setup Mode and Working Mode Gate - AFK - Blocked by: 1
3. Operator Console Readiness Dashboard - HITL - Blocked by: 2
4. Default PTGui Template and Active Calibration Lifecycle - AFK - Blocked by: 1
5. 9-Grid and PTGui Mapping Verification - HITL - Blocked by: 2, 4
6. ROI Profile Store and Overlay Verification Roll - HITL - Blocked by: 2, 5
7. 500 ms REC Consensus and Recording Classification - AFK - Blocked by: 6
8. Authoritative Roll/Clip OCR Take Identity - AFK - Blocked by: 6, 7
9. Take Lifecycle Sorting, Sidecars, and Manifest Rollup - AFK - Blocked by: 7, 8
10. Alarm Controller and Acknowledgement Model - AFK - Blocked by: 7
11. Live 9-Grid Recording/Alarm Overlays - HITL - Blocked by: 10
12. Independent Recorder Health and Protected 9-Grid Fallback - AFK - Blocked by: 9, 10
13. Late Stop and False Roll Flows - AFK - Blocked by: 9, 10
14. Setup Test Recording Sandbox - AFK - Blocked by: 2, 9
15. Minimal TC Source Status and Degraded-TC Marking - AFK - Blocked by: 9
16. Open Record Folder Action - AFK - Blocked by: 9
17. DeckLink/RP188 Capture Provider - HITL - Blocked by: 15
18. Primary TC Source Failover and Drift Evidence - AFK - Blocked by: 17
19. Embedded QuickTime `tmcd` and Proxy TC Self-Test - AFK - Blocked by: 17, 18
20. Live Stitch Program Overlay System - HITL - Blocked by: 15
21. Slate Service and Operator Console Slate Controls - HITL - Blocked by: 9, 20
22. GPS/IMU Motion Data Source and Per-Take Slicing - HITL - Blocked by: 15, 17
23. Stream Comment TC Stamping and Standby Blur - AFK - Blocked by: 15, 20
24. End-of-Day Camera Report Package - HITL - Blocked by: 9, 15, 21, 22, 23
25. Analyzer Corpus and Simulator Harness - AFK - Blocked by: 6, 7, 8
26. PTGui Still Grab Support Tool - HITL - Blocked by: 4, 5
27. System Calibration Mode - HITL - Blocked by: 25
28. 9-Grid Camera Punch-Up - AFK - Blocked by: 11
29. Flow Warp Validation Harness - AFK - Blocked by: 4, 9
30. Radial Distortion Parity Experiment - AFK - Blocked by: 4, 29

## Phase 1 - Core Loop

### 1. Startup, Resume, and Recovery Shell

Type: AFK

Blocked by: None - can start immediately.

What to build:

Add first-run startup routing that separates startup decisions from source/calibration setup. Normal startup shows Resume Existing Job first and Start New Job second. Interrupted sessions show Recovery Startup with job details, last mode, last closed take, in-progress take, and actions for Resume Session, Open Setup Mode, Start New Job, and archive/ignore.

Acceptance criteria:

- [ ] Startup shows Resume Existing Job as the primary action when previous job state exists.
- [ ] Startup shows Start New Job as the secondary action.
- [ ] Interrupted/unclosed session state routes to Recovery Startup instead of normal startup.
- [ ] Recovery Startup shows job identity, last mode, last closed take, in-progress take if known, and project folder if known.
- [ ] Resume Existing Job and Recovery Startup route into Setup Mode.
- [ ] Starting a new job clears current job setup state and enters the normal job setup flow.
- [ ] Session state persists enough information to detect abnormal termination on next launch.
- [ ] Tests or manual verification cover cold start, resume, archive/ignore recovery, and interrupted session detection.

### 2. Setup Mode and Working Mode Gate

Type: AFK

Blocked by: 1.

What to build:

Add explicit Setup Mode and Working Mode state. Setup Mode runs verification and optional setup recording, uses orange visual treatment across MLS windows, and has a large green action to enter Working Mode. Non-green readiness requires acknowledgement but never blocks entry.

Acceptance criteria:

- [ ] Current mode persists in app/project session state.
- [ ] New/resumed jobs enter Setup Mode.
- [ ] Setup Mode orange treatment appears on Operator Console, 9-grid, and Live Stitch windows.
- [ ] Working Mode requires deliberate operator action.
- [ ] Non-green readiness prompts acknowledgement before Working Mode but does not block.
- [ ] Mode transitions are logged.
- [ ] Existing jobs can reopen in Setup Mode with still-valid readiness checks green.

### 3. Operator Console Readiness Dashboard

Type: HITL

Blocked by: 2.

What to build:

Convert the third window into the Operator Console with an OBS-style Spheris-branded dashboard and modules for Dashboard, Setup, Cameras/Mapping, Overlay/QC, Calibration, Slate, Stream, GPS/IMU, Reports, and Diagnostics. Phase 1 must expose readiness status and key drill-ins even if later modules are placeholders.

Acceptance criteria:

- [ ] UI uses Operator Console / OC terminology.
- [ ] Dashboard summarizes job, mode, readiness, active alarms, recording state, disk/recorder health, and key actions.
- [ ] Core modules are reachable as tabs or equivalent navigation.
- [ ] Readiness categories show green/yellow/orange/red state.
- [ ] Setup Mode vs Working Mode is obvious.
- [ ] Layout remains usable at the existing minimum OC window size.
- [ ] Operator/design review signs off that the layout is clear enough for on-set use.

### 4. Default PTGui Template and Active Calibration Lifecycle

Type: AFK

Blocked by: 1.

What to build:

Make the global Spheris XL default `.pts` template a first-class active calibration. New jobs load it immediately. Operators can import/replace the active PTGui template in Setup Mode or between takes. MLS copies active templates into the project calibration folder and records active template identity in manifest and per-take sidecars.

Acceptance criteria:

- [ ] New jobs start with bundled/global default PTGui template loaded.
- [ ] Resumed jobs restore the last active job template when available.
- [ ] Import/replace works in Setup Mode and between takes.
- [ ] Import/replace is disabled during active recording.
- [ ] Active template is copied into the project calibration folder.
- [ ] Template switches are logged.
- [ ] Project manifest and each take sidecar record the active template.
- [ ] Live Stitch remap updates after template switch.

### 5. 9-Grid and PTGui Mapping Verification

Type: HITL

Blocked by: 2, 4.

What to build:

Add a Cameras/Mapping module that verifies 9-grid/SDI mapping separately from Live Stitch/PTGui template mapping. The default behavior derives PTGui assignment from verified 9-grid positions. Add a clearly marked advanced Template Mapping Override for nonstandard templates.

Acceptance criteria:

- [ ] Mapping view shows SDI input/channel, 9-grid cell, OCR camera ID, Pinwheel position, PTGui slot, Live Stitch expected camera ID, and match status per slot.
- [ ] Standard mapping derives Live Stitch assignment from verified 9-grid position.
- [ ] Template Mapping Override can reorder PTGui assignment independently.
- [ ] Override state is visually obvious.
- [ ] Mapping state persists to job setup and manifest.
- [ ] Human review verifies the flow matches the physical array workflow.

### 6. ROI Profile Store and Overlay Verification Roll

Type: HITL

Blocked by: 2, 5.

What to build:

Create saved ROI profiles keyed by SDI/output profile and a required setup verification roll. The roll/cut cycle verifies REC bug detection and roll/clip advancement. Stable overlay fields are passively checked against expected baseline.

Acceptance criteria:

- [ ] ROI profiles load/save by SDI/output profile.
- [ ] Manual ROI drawing/editing exists as an advanced repair path.
- [ ] Setup verification forces a roll/cut cycle.
- [ ] Verification records REC bug detection evidence per camera.
- [ ] Verification records roll/clip advancement evidence per camera.
- [ ] Stable fields are passively checked against baseline.
- [ ] Verification becomes stale when mapping, ROI profile, SDI profile, calibration/job setup, capture device mapping, expected settings, or timestamp context changes.
- [ ] Operator review confirms the required verification is fast enough.

### 7. 500 ms REC Consensus and Recording Classification

Type: AFK

Blocked by: 6.

What to build:

Implement Working Mode red-REC consensus using a 500 ms window. Any red-REC-triggered event writes proxy files. The consensus engine classifies normal 9/9, 8/9 `CAMERA_FAILED_TO_RECORD`, 1/9 `CAMERA_RECORDED_ALONE`, and middle partial `RECORD_CONSENSUS_FAILED`.

Acceptance criteria:

- [ ] Start consensus uses 500 ms.
- [ ] 9/9 REC starts a normal take.
- [ ] 8/9 REC emits `CAMERA_FAILED_TO_RECORD` with missing camera evidence.
- [ ] 1/9 REC emits `CAMERA_RECORDED_ALONE`.
- [ ] Middle partial patterns emit `RECORD_CONSENSUS_FAILED`.
- [ ] Any red-REC-triggered event begins writing configured proxy outputs.
- [ ] Consensus evidence is emitted through the shared event stream.
- [ ] Tests cover 9/9, 8/9, 1/9, 5/9, delayed REC, and no REC timelines.

### 8. Authoritative Roll/Clip OCR Take Identity

Type: AFK

Blocked by: 6, 7.

What to build:

Make OCR roll/clip consensus the production identity for Working Mode take folders/files. Support arbitrary roll and clip values. Treat unreadable or low-confidence roll/clip as a mission-critical analyzer/readiness failure rather than silently inventing counters.

Acceptance criteria:

- [ ] Working Mode take identity comes from OCR roll/clip consensus.
- [ ] Arbitrary roll starts are supported.
- [ ] Unreadable OCR creates visible analyzer/readiness failure.
- [ ] Low-confidence OCR does not silently produce invented production naming.
- [ ] OCR evidence per camera is written to event/sidecar data.
- [ ] Tests cover high rolls, unreadable OCR, split consensus, clip advancement, and resumed arbitrary roll/clip.

### 9. Take Lifecycle Sorting, Sidecars, and Manifest Rollup

Type: AFK

Blocked by: 7, 8.

What to build:

Finalize the Working Mode take lifecycle. At stop, close all outputs, classify the take, move it into `takes/`, `qc_fail/`, or `false_roll/`, write the take sidecar, and update project manifest. Manifest should be current after every closed take.

Acceptance criteria:

- [ ] Closed takes write per-take sidecar before UI reports completion.
- [ ] Project manifest updates after every closed take.
- [ ] Normal 9/9 takes land in `takes/`.
- [ ] Start-consensus failures land in `qc_fail/`.
- [ ] Sidecar records sort folder, anomaly class, severity, per-camera evidence, output success/failure, and active calibration.
- [ ] Folder/file moves handle collisions and interruption safely.
- [ ] Tests cover folder moves, sidecar content, manifest rollup, and collision handling.

### 10. Alarm Controller and Acknowledgement Model

Type: AFK

Blocked by: 7.

What to build:

Build a shared alarm controller for severity, visual priority, acknowledgement, active/resolved state, and audio/TTS hooks. Red is recording. Orange is critical. Yellow/amber is warning/status. Acknowledgement silences repeat audio/TTS but keeps visual state until resolution.

Acceptance criteria:

- [ ] Alarm categories include critical recording failure, critical sync failure, warning, and info/no-sound.
- [ ] Red/orange/yellow visual priority is enforced.
- [ ] Critical alarms require acknowledgement.
- [ ] Acknowledgement suppresses repeat audio/TTS hooks while condition remains visible.
- [ ] Warnings can auto-clear visually after resolution while remaining logged.
- [ ] Event log and sidecar preserve alarm detail and acknowledgement state.
- [ ] Tests cover priority, acknowledgement, clear, supersede, and resolved-still-active states.

### 11. Live 9-Grid Recording/Alarm Overlays

Type: HITL

Blocked by: 10.

What to build:

Show per-camera recording and alarm state on the live 9-grid without burning MLS status/QC overlays into the recorded 9-grid by default. Recording uses red. Critical alarms use orange. Warnings use yellow/amber.

Acceptance criteria:

- [ ] Live 9-grid cells show per-camera REC state.
- [ ] Live 9-grid cells show camera-specific critical/warning alarms.
- [ ] Recording state is not confused with orange/yellow alarms.
- [ ] Acknowledged alarms change visual state without disappearing until resolved.
- [ ] Recorded 9-grid proxy excludes MLS status/QC overlays by default.
- [ ] Debug/simulator triggers representative overlays.
- [ ] Operator review confirms readability and visual priority.

### 12. Independent Recorder Health and Protected 9-Grid Fallback

Type: AFK

Blocked by: 9, 10.

What to build:

Separate Live Stitch and 9-grid writer lifecycle/error states and surface health in the Operator Console. 9-grid is the protected fallback. Live Stitch writer failure should not stop 9-grid. 9-grid writer failure raises critical alarm.

Acceptance criteria:

- [ ] Live Stitch and 9-grid have independent active/error states.
- [ ] OC shows active state, frames written, dropped frames, disk free, take duration, and last write time.
- [ ] Writer stalls and severe frame drops alarm during the take.
- [ ] Live Stitch failure does not stop 9-grid.
- [ ] 9-grid failure raises critical alarm.
- [ ] Sidecar records per-output success/failure and health anomalies.
- [ ] Fault-injection tests cover Live Stitch failure, 9-grid failure, disk low, stall, and drops.

### 13. Late Stop and False Roll Flows

Type: AFK

Blocked by: 9, 10.

What to build:

Implement late-stop warning and false-roll routing. Cameras still recording after array stop warn loudly. Once stopped, the take can remain good with `CAMERA_STOPPED_LATE` metadata. False-roll action moves the relevant take to `false_roll/`.

Acceptance criteria:

- [ ] Late stop is separate from start consensus.
- [ ] Active late stop warns while one or more cameras are still recording.
- [ ] Resolved late stop can remain in `takes/`.
- [ ] Sidecar records late camera(s) and offsets.
- [ ] New REC toggle while a camera remains rolling is classified through next take start consensus.
- [ ] Operator can mark last take false roll.
- [ ] False-roll move updates sidecar, manifest, and event log.
- [ ] Tests cover late stop, resolved late stop, false roll, and next take after unresolved roll.

### 14. Setup Test Recording Sandbox

Type: AFK

Blocked by: 2, 9.

What to build:

Add visible `Record Setup Tests` toggle in Setup Mode. Enabled setup recordings write under `_SETUP_TESTS/` with `jobname_setup_test_0001` naming and do not enter production manifest or reports.

Acceptance criteria:

- [ ] Setup Mode exposes Record Setup Tests toggle.
- [ ] Setup tests write under `_SETUP_TESTS/`.
- [ ] Setup test naming follows `jobname_setup_test_0001`.
- [ ] Setup tests do not increment production take/slate counts.
- [ ] Setup tests do not appear in production manifest `takes[]`.
- [ ] Enter Working Mode prompts clear-by-default with keep option.
- [ ] Tests cover setup recording and manifest exclusion.

### 15. Minimal TC Source Status and Degraded-TC Marking

Type: AFK

Blocked by: 9.

What to build:

Use existing captured-frame metadata quality to expose TC source confidence before full RP188 implementation. Mark host-clock/frame-counter/no-RP188 recordings as degraded and non-certified in UI, sidecars, and manifest-compatible metadata.

Acceptance criteria:

- [ ] OC shows TC source quality/status.
- [ ] Takes without RP188 are marked degraded/non-certified.
- [ ] Sidecar/manifest metadata distinguishes certified RP188 from fallback timing.
- [ ] Missing RP188 makes TC drift unavailable/degraded, not failed.
- [ ] Event log records no-RP188/degraded-TC events.
- [ ] Tests cover RP188 mock frames, host-clock fallback, frame-counter fallback, and unavailable TC.

### 16. Open Record Folder Action

Type: AFK

Blocked by: 9.

What to build:

Add a Working Mode OC action that opens the primary project recording folder containing `takes/`, `qc_fail/`, and `false_roll/`.

Acceptance criteria:

- [ ] OC includes Open Record Folder in Working Mode.
- [ ] Action opens the primary project recording folder, not only the latest take.
- [ ] Missing folder state is handled nonfatally.
- [ ] Action is disabled or clearly labeled when no project is active.
- [ ] Event log records the operator action.

## Phase 2 - Timecode, Program Overlays, Sensors, Stream, Reports

### 17. DeckLink/RP188 Capture Provider

Type: HITL

Blocked by: 15.

What to build:

Implement the production-certified DeckLink capture provider behind the frame spine. It must extract per-frame RP188 timecode, report signal state, source identity, frame timing, metadata quality, drops, and adapter reconnect state. AVFoundation/file playback remain degraded fallback sources.

Acceptance criteria:

- [ ] DeckLink provider emits frames with RP188 `CaptureTimecode` when present.
- [ ] Provider sets metadata quality to certified RP188 only for real RP188.
- [ ] Provider reports signal present/no frame/disconnected.
- [ ] Provider reports source identity and frame/drop counters.
- [ ] AVFoundation/file providers remain available but marked lower-confidence.
- [ ] Hardware certification checklist verifies all 9 SDI inputs.

### 18. Primary TC Source Failover and Drift Evidence

Type: AFK

Blocked by: 17.

What to build:

Implement TC source selection, failover, and drift evidence. Position 1 is the default proxy TC stamping source. If it loses valid RP188, fail over to the lowest-numbered valid RP188 source and warn/log. Drift compares cameras to array median/consensus.

Acceptance criteria:

- [ ] Setup/config exposes primary TC source with Position 1 default.
- [ ] Automatic failover chooses lowest-numbered valid RP188 source.
- [ ] No silent switchback occurs mid-take.
- [ ] RP188 drift uses array median/consensus.
- [ ] `>1 frame` drift emits `TC_DRIFT_SOFT`; `>5 frames` emits critical `SYNC_LOST`.
- [ ] HUD sync/genlock evidence and RP188 drift evidence are stored separately.
- [ ] Tests cover primary loss, failover, no RP188, one bad source, and threshold events.

### 19. Embedded QuickTime `tmcd` and Proxy TC Self-Test

Type: AFK

Blocked by: 17, 18.

What to build:

Write embedded QuickTime `tmcd` tracks into production-valid Live Stitch and 9-grid proxy MOVs using the active certified RP188 source. Add startup/self-test that records a short proxy and verifies embedded TC matches captured RP188.

Acceptance criteria:

- [ ] Production-valid proxy MOVs include true embedded `tmcd`.
- [ ] Sidecar TC fields match embedded TC start/end.
- [ ] Degraded no-RP188 proxy MOVs omit `tmcd` when possible or mark host-derived when unavoidable.
- [ ] Startup proxy TC self-test verifies a short output.
- [ ] Self-test failure emits `PROXY_TC_MISMATCH`.
- [ ] Tests or tooling inspect generated MOV TC metadata.

### 20. Live Stitch Program Overlay System

Type: HITL

Blocked by: 15.

What to build:

Add recordable Live Stitch program overlays for TC, roll/clip, and slate/client-facing metadata. Provide Clean, Production Review, Minimal, and Custom presets, lower-band layout, drag/reorder placement, live/record toggles, and red TC while recording. Live Stitch recording border is live-only.

Acceptance criteria:

- [ ] TC overlay supports RP188, visual/OCR TC, and confirmed GPS/system TOD fallback.
- [ ] TC displays `HH:MM:SS:FF`; no milliseconds as primary TC.
- [ ] TC turns red while recording and white while idle.
- [ ] Roll/clip overlay defaults to friendly `Roll 005  Clip 006` with compact option.
- [ ] Presets include Clean, Production Review, Minimal, and Custom.
- [ ] Live-display and record-overlay toggles are separate.
- [ ] Production-client mode records overlays by default; stock/clean can disable.
- [ ] Live-only red recording border is not recorded.
- [ ] Operator/design review signs off on legibility and layout.

### 21. Slate Service and Operator Console Slate Controls

Type: HITL

Blocked by: 9, 20.

What to build:

Add slate metadata controls in OC. Job-level project/director/DP/company are sticky. Scene/shot/take are quick-edit fields. Values snapshot at REC start, feed overlays, sidecars, manifest, and reports. Last-take retroactive edit updates data and logs history.

Acceptance criteria:

- [ ] OC exposes sticky job slate fields and quick scene/shot/take fields.
- [ ] Slate values snapshot at REC start.
- [ ] Live Stitch overlay reads current/snapshotted slate correctly.
- [ ] Last-take slate edit updates sidecar/manifest/report data.
- [ ] Slate edits are logged.
- [ ] Take auto-increments after Working Mode takes.
- [ ] Setup tests do not increment slate take.
- [ ] False roll asks keep/increment/revert.

### 22. GPS/IMU Motion Data Source and Per-Take Slicing

Type: HITL

Blocked by: 15, 17.

What to build:

Implement logging-first GPS/IMU support for the simpleRTK2B Fusion. Discover/configure USB serial, archive raw NMEA and UBX NAV/ESF with host timestamps, parse GPS route/time/fix/speed/course, align samples to camera TC, and write per-take JSON/CSV with 2-5 seconds pre/post context.

Acceptance criteria:

- [ ] GPS/IMU logging can be enabled/disabled.
- [ ] Missing hardware is yellow/non-blocking.
- [ ] Connected no-lock/not-logging is a more serious OC status but non-blocking.
- [ ] Raw NMEA and UBX NAV/ESF packets are archived with host timestamps when enabled.
- [ ] Parsed samples include time, fix, satellites/HDOP, speed/course, location, and available IMU fields.
- [ ] Samples align to RP188 when available, visual/overlay TC second, degraded fallback last.
- [ ] Each Working Mode take gets JSON and CSV motion slices with pre/post context.
- [ ] Report status classifications are emitted.
- [ ] Tests use recorded NMEA/UBX sample streams.
- [ ] Hardware review verifies real device behavior.

### 23. Stream Comment TC Stamping and Standby Blur

Type: AFK

Blocked by: 15, 20.

What to build:

Refine the existing stream system so comments are stamped with Live Stitch TC and persisted immediately to a project-level log. Add stream standby blur/message when cameras are not recording and clear feed while rolling.

Acceptance criteria:

- [ ] Comments store display TC, frame count when available, TC source/confidence, server receipt time, user identity, and message.
- [ ] Comments persist immediately to project-level log.
- [ ] Final association to takes can be recomputed from finalized take TC ranges.
- [ ] Stream is blurred/obscured with message when not recording.
- [ ] Stream becomes clear while rolling.
- [ ] Local MLS recordings remain source of truth; stream recording is not required.

### 24. End-of-Day Camera Report Package

Type: HITL

Blocked by: 9, 15, 21, 22, 23.

What to build:

Generate end-of-day report packages with a human-readable PDF camera report, machine-readable JSON report data, manifest snapshot, comments/chat log, GPS/IMU CSV summaries, and media inventory CSV. Shoot day boundaries are explicit operator actions.

Acceptance criteria:

- [ ] Setup Mode can start/confirm a shoot day boundary.
- [ ] End-of-day export produces a PDF report.
- [ ] Export also produces JSON report data.
- [ ] Export includes manifest snapshot, comments/chat log, GPS/IMU CSV summaries, and media inventory CSV.
- [ ] Report groups takes by shoot day.
- [ ] Report includes setup readiness summary, take table, false rolls, QC failures, anomalies, slate, TC status, GPS/IMU status, and comments.
- [ ] Human review confirms report is production-readable and ZoeLog/camera-report inspired.

## Phase 3 - Calibration Utilities And Quality

### 25. Analyzer Corpus and Simulator Harness

Type: AFK

Blocked by: 6, 7, 8.

What to build:

Create a hardware-free analyzer and REC/OCR test harness using saved RED overlay reference frames and simulated per-camera timelines.

Acceptance criteria:

- [ ] Corpus includes standby, recording, clip advancement, card missing, sync lost/degraded, high roll numbers, different letters, and supported output profiles.
- [ ] Expected outputs are stored as fixtures.
- [ ] Test runner validates REC bug, roll/clip OCR, status color fields, and settings/status fields.
- [ ] Simulator validates REC consensus edge cases.
- [ ] Harness can be run by agents without connected camera array.

### 26. PTGui Still Grab Support Tool

Type: HITL

Blocked by: 4, 5.

What to build:

Add one-button clean still grab for PTGui template creation. Normal path prompts overlays off, captures max practical quality stills, then prompts overlays back on. Rush path can crop/mask overlays. Save TIFF when possible under the project calibration stills folder with manifest.

Acceptance criteria:

- [ ] Tool captures all 9 camera stills.
- [ ] Normal path prompts overlays off before capture and on after capture.
- [ ] Rush path supports crop/mask with lower usable resolution warning.
- [ ] Stills save under `_CALIBRATION/stills/YYYYMMDD_HHMMSS/`.
- [ ] Filenames include camera letter and Pinwheel position/direction when known.
- [ ] TIFF is used when available; PNG fallback is explicit.
- [ ] Still-grab manifest records per-camera TC/frame index/host timestamp and timing consistency.
- [ ] Operator review verifies workflow.

### 27. System Calibration Mode

Type: HITL

Blocked by: 25.

What to build:

Add a deeper troubleshooting System Calibration mode that guides the team through broader settings/status changes while MLS analyzes transitions and updates analyzer/ROI confidence.

Acceptance criteria:

- [ ] System Calibration is distinct from per-job setup verification.
- [ ] Guided prompts cover relevant RED overlay statuses/settings.
- [ ] MLS captures evidence and analyzer confidence per camera/field.
- [ ] ROI/profile updates can be saved after validation.
- [ ] Summary report identifies weak OCR/status fields.
- [ ] Operator/hardware review confirms prompts are useful.

### 28. 9-Grid Camera Punch-Up

Type: AFK

Blocked by: 11.

What to build:

Add right-click or equivalent punch-up from the 9-grid to temporarily show one camera large for focus/settings inspection. Exit on next click/key stroke. Disable during recording.

Acceptance criteria:

- [ ] Punch-up replaces the grid view temporarily.
- [ ] Punch-up exits on click/key stroke.
- [ ] Punch-up is disabled during active recording.
- [ ] State does not affect recorded 9-grid output.
- [ ] Manual verification covers all 9 cells.

### 29. Flow Warp Validation Harness

Type: AFK

Blocked by: 4, 9.

What to build:

Validate existing flow warp path against real footage/template samples. Define objective before/after comparisons and a toggleable experimental path. Keep this post-MVP unless it proves safe.

Acceptance criteria:

- [ ] Test harness renders before/after frames from known footage and PTGui template.
- [ ] Comparison artifacts are saved for review.
- [ ] Objective acceptance metrics or visual review checklist are documented.
- [ ] Flow warp remains optional/toggleable.
- [ ] No change can regress baseline PTGui-template stitch path.

### 30. Radial Distortion Parity Experiment

Type: AFK

Blocked by: 4, 29.

What to build:

Investigate PTGui radial distortion convention and validate against PTGui output before enabling in renderer.

Acceptance criteria:

- [ ] Distortion math variants are tested against PTGui reference output.
- [ ] Boundary artifacts are measured/documented.
- [ ] Renderer keeps distortion disabled until parity is proven.
- [ ] Decision note records final convention or reason for deferral.

## Publishing Notes

- Apply a `ready-for-agent` label if these are published to GitHub issues.
- Publish in dependency order so issue references can replace numeric blockers.
- HITL tasks should include the required human review in the issue body.
- AFK tasks should include fixtures or simulators wherever hardware is not required.
- Do not assign tasks that require the full camera array unless the issue explicitly says HITL/hardware review.
