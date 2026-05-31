# Meridian Live Stitch MVP PRD

Source material: current conversation decisions, `docs/MVP_KANBAN.md`, `docs/PHASE1_ISSUES.md`, `AGENTS.md`, and current codebase structure.

Status: draft for implementation planning and agent task generation.

## Problem Statement

Meridian Live Stitch needs to become a production-safe macOS app for a 9-camera Spheris XL 360 array. The core business risk is not only whether the Live Stitch looks good; it is whether the operator can trust that all nine RED Komodos are recording the correct roll/clip, that the 9-grid and Live Stitch proxies are written reliably, and that timecode, sync, GPS/IMU, comments, and reports are accurate enough for on-set review and downstream post.

Today the app has important foundations: Swift/Metal real-time stitching from PTGui `.pts`, a 9-grid, recording coordinator, manifest sidecars, event sink, visual analyzer foundations, RCP2/camera setup, streaming plumbing, GPS/IMU diagnostics, and calibration tooling. It does not yet have the production-grade operating workflow required on set: forced setup verification, 500 ms REC consensus, authoritative OCR roll/clip identity, reliable take classification, clear operator alarms, protected recording fallback, RP188-backed timecode, per-take GPS/IMU alignment, Live Stitch program overlays, and end-of-day camera reports.

The MVP must make the system operationally safe first. The app must never be blocked from recording because a secondary system is missing, but it must loudly mark degraded or risky conditions. The 9-grid and Live Stitch recordings are the primary assets; the 9-grid is the protected forensic fallback if anything else fails.

## Solution

Build Meridian Live Stitch around an explicit on-set workflow:

1. Startup prioritizes resuming existing jobs and recovering interrupted sessions.
2. Every new/resumed job enters Setup Mode first.
3. Setup Mode validates camera mapping, ROI/OCR profiles, RED overlays, REC bug detection, roll/clip advancement, calibration template, recording destination, timecode status, stream status, and GPS/IMU status.
4. Working Mode records production takes using the RED REC indicator as the gate and camera OCR roll/clip as the production identity.
5. The Operator Console coordinates readiness, recording health, alarms, calibration, mapping, slate, stream, GPS/IMU, reports, and diagnostics.
6. The 9-grid QC window remains the operator's primary shooting monitor and shows per-camera recording/alarm state without burning MLS UI overlays into the recorded 9-grid proxy.
7. The Live Stitch window supports client-facing program overlays for timecode, roll/clip, and slate metadata, with clean/stock modes able to turn overlays off.
8. Recording outputs are self-contained: per-take sidecars, project manifest, event log, calibration copy, comments, GPS/IMU slices, and end-of-day report packages.
9. RP188 over SDI is the certified production timecode path. If unavailable, MLS still records but marks the take degraded in UI, sidecars, and reports.

The MVP should be implemented as a sequence of vertical slices. Each slice should deliver operator-visible behavior plus the data/reporting needed to verify it. Agents should work on one slice at a time, with tests around stable deep modules rather than brittle UI details.

## User Stories

1. As an operator, I want startup to prioritize Resume Existing Job, so that lunch moves, location moves, and restarts do not waste time.
2. As an operator, I want a dedicated Recovery Startup screen after an interrupted session, so that I can see what happened before continuing.
3. As an operator, I want resumed and new jobs to open into Setup Mode, so that I can verify the rig before shooting.
4. As an operator, I want Setup Mode to look unmistakably different, so that setup/test rolls are never confused with production.
5. As an operator, I want Working Mode to be entered deliberately with a clear action, so that production take classification begins intentionally.
6. As an operator, I want non-green readiness states to warn but not block Working Mode, so that MLS cannot stop us from shooting in an emergency.
7. As an operator, I want a readiness dashboard, so that I can see job, mapping, overlays, calibration, record destination, stream, GPS/IMU, alarms, and reports in one place.
8. As an operator, I want saved readiness checks to stay green only while context is unchanged, so that stale verification does not create false confidence.
9. As an operator, I want to verify 9-grid SDI mapping separately from Live Stitch/PTGui mapping, so that camera order issues are visible.
10. As an operator, I want the normal mapping to derive Live Stitch camera order from the verified 9-grid order, so that setup stays simple.
11. As an operator, I want an advanced PTGui template mapping override, so that unusual template ordering can be corrected without changing the 9-grid.
12. As an operator, I want every job to open with a usable default PTGui template, so that the Live Stitch works immediately.
13. As an operator, I want to import or replace the active PTGui template outside recording, so that job-specific calibration can be used when available.
14. As an operator, I want calibration switches blocked during active recording, so that a take cannot change stitch geometry mid-file.
15. As an operator, I want saved ROI profiles by SDI/output profile, so that RED overlay detection does not require manual setup every job.
16. As an operator, I want a required setup verification roll, so that REC detection and roll/clip advancement are proven before real shooting.
17. As an operator, I want stable RED overlay fields checked passively during setup, so that FPS, shutter, ISO, WB, codec, media, sync, and power issues surface without slow forced tests.
18. As an operator, I want manual ROI repair tools, so that I can recover if the saved profile no longer matches the feed.
19. As an operator, I want MLS to detect recording from the red record bug, so that the analog REC distribution behavior is verified from the actual camera overlays.
20. As an operator, I want REC consensus to resolve in 500 ms, so that bad takes are flagged quickly.
21. As an operator, I want any red-REC-triggered event to write files, so that MLS does not choose to drop evidence.
22. As an operator, I want 9/9 cameras recording within the consensus window to create a normal take, so that production takes are cleanly classified.
23. As an operator, I want 8/9 recording to create `CAMERA_FAILED_TO_RECORD`, so that the missed camera is explicit.
24. As an operator, I want 1/9 recording to create `CAMERA_RECORDED_ALONE`, so that accidental solo records are explicit.
25. As an operator, I want ambiguous partial patterns to create `RECORD_CONSENSUS_FAILED`, so that bad states are not mislabeled.
26. As an operator, I want late-stopping cameras to warn loudly but not ruin the original take automatically, so that normal toggle behavior is handled correctly.
27. As an operator, I want `CAMERA_STOPPED_LATE` recorded with camera and offset, so that QC can understand longer-running cameras later.
28. As an operator, I want camera OCR roll/clip to be production-authoritative, so that folders match actual camera media even across arbitrary roll numbers.
29. As an operator, I want unreadable roll/clip OCR to be treated as a mission-critical analyzer problem, so that the app never silently invents production identity.
30. As an operator, I want clip-offset status to remain yellow/amber after known partial recording, so that we can keep shooting without a full-screen nuisance.
31. As an operator, I want a Verify Clip Sync / Check All Settings action, so that manually corrected cameras can be confirmed quickly.
32. As an operator, I want the 9-grid to show recording and alarm state per camera, so that I can see exactly which camera has a problem.
33. As an operator, I want red reserved for recording and orange/yellow reserved for errors/warnings, so that status is not visually ambiguous.
34. As an operator, I want critical alarms to flash visually and use sound/TTS, so that shoot-threatening issues cannot be missed.
35. As an operator, I want acknowledgement to silence audio/TTS while keeping visuals active, so that I can continue working without losing status.
36. As an operator, I want lower-priority warnings not to dominate the screen, so that shooting stays manageable.
37. As an operator, I want a temporary 9-grid camera punch-up, so that focus/settings can be inspected without leaving the QC view.
38. As production/post, I want the recorded 9-grid proxy clean of MLS UI overlays by default, so that it remains forensic evidence.
39. As a client/operator, I want a red border around the Live Stitch screen while recording, so that recording state is obvious.
40. As production/post, I want that red Live Stitch border to be live-only, so that it is not burned into recorded files.
41. As production/post, I want Live Stitch program overlays for TC, roll/clip, and slate metadata, so that review assets are useful on set.
42. As an operator, I want Live Stitch overlays to be toggleable separately for live display and recording, so that stock jobs can record clean.
43. As an operator, I want overlay presets, so that production review, minimal, clean, and custom layouts are fast to select.
44. As an operator, I want quick slate fields in the Operator Console, so that scene/shot/take can be updated between takes.
45. As an operator, I want slate values snapshotted at REC start, so that take metadata matches what was active when recording began.
46. As an operator, I want retroactive last-take slate edits logged, so that corrections are possible without hiding history.
47. As an operator, I want the slate take number to auto-increment after Working Mode takes, so that common take entry is fast.
48. As an operator, I want false rolls to ask whether to keep, increment, or revert the slate take, so that slate counting stays intentional.
49. As an operator, I want Live Stitch TC to come from RP188 first, visual/OCR TC second, and GPS/system TOD only with confirmation, so that timecode confidence is clear.
50. As production/post, I want production-valid MOV proxies to include embedded QuickTime `tmcd` from RP188, so that editing tools see real camera timecode.
51. As production/post, I want degraded no-RP188 takes not to look camera-certified, so that downstream users do not trust approximate timing.
52. As an operator, I want primary TC source failover, so that loss of Position 1 does not make eight good cameras look wrong.
53. As an operator, I want RP188 drift measured against array consensus, so that one bad source does not condemn the array.
54. As an operator, I want TC GEN / sync failures to be critical, so that master timing problems are not missed.
55. As an operator, I want TC drift and HUD sync/genlock evidence recorded separately, so that reports show why an alarm fired.
56. As an operator, I want to record even when RP188 is missing, so that emergency SDI capture still preserves assets.
57. As production/post, I want degraded TC status written to sidecar and report, so that timing confidence travels with the take.
58. As an operator, I want live recorder health for 9-grid and Live Stitch, so that writer stalls are visible during the take.
59. As production, I want Live Stitch and 9-grid recorder pipelines independent, so that one failure does not kill both assets.
60. As production, I want 9-grid prioritized as protected fallback, so that all camera evidence survives severe failure when possible.
61. As an operator, I want setup test recording under `_SETUP_TESTS/`, so that test rolls can be captured without polluting production data.
62. As an operator, I want setup artifacts cleared by default on entering Working Mode, so that disposable setup files do not clutter the job.
63. As an operator, I want an Open Record Folder button, so that recent outputs can be found quickly.
64. As an operator, I want clean PTGui still grab, so that job-specific templates can be built faster.
65. As an operator, I want still grab to prompt overlays off/on, so that captured stills maximize usable image area.
66. As an operator, I want a rushed still-grab mode with crop/mask fallback, so that urgent setups can proceed.
67. As an operator, I want stills saved as TIFF when possible, so that PTGui inputs preserve practical quality.
68. As an operator, I want GPS/IMU logging to be optional and non-blocking, so that missing GPS/IMU never prevents recording.
69. As production/post, I want GPS/IMU data sliced per take, so that downstream tools do not need to parse full-day logs.
70. As VFX/post, I want per-take GPS/IMU CSV and JSON, so that motion/location data is usable without custom parsing.
71. As VFX/post, I want GPS/IMU aligned to camera TC, so that sensor samples can be correlated to frames.
72. As an operator, I want GPS/IMU status in the Operator Console, so that missing, no-lock, dropout, disabled, and degraded alignment states are visible.
73. As an operator, I want route/map visualization later in the GPS/IMU tab, so that recorded movement can be reviewed.
74. As a stream viewer, I want the stream blurred when cameras are not recording, so that client viewers only see clear feed while rolling.
75. As a client/commenter, I want comments stamped with Live Stitch TC, so that liked moments can be found later.
76. As production/post, I want stream comments stored immediately in a project log, so that comments survive app/session issues.
77. As production/post, I want comments associated to finalized take ranges, so that report handoff can group notes by take.
78. As production/post, I want a human-readable end-of-day PDF camera report, so that camera reports can be delivered daily.
79. As production/post, I want a machine-readable JSON report package, so that automated tools can ingest the same facts.
80. As an operator, I want shoot day boundaries explicit, so that reports do not split days unexpectedly.
81. As production/post, I want report packages to include PDF, JSON, manifest snapshot, chat log, GPS/IMU CSV summaries, and media inventory, so that the day handoff is complete.
82. As a developer, I want an analyzer corpus and harness, so that RED overlay recognition can be regression tested.
83. As a developer, I want REC/OCR logic testable without live hardware, so that agents can safely improve the core loop.
84. As a developer, I want recording classification testable from simulated camera state timelines, so that edge cases are locked down.
85. As a developer, I want TC source/fallback logic testable from captured-frame metadata, so that degraded/certified status is reliable.
86. As a developer, I want GPS/IMU parsing and take slicing tested from sample streams, so that hardware quirks do not break reports.
87. As a developer, I want camera report generation tested from fixtures, so that PDF/JSON output remains stable.
88. As the business owner, I want MLS outputs self-contained and not blocked by NMM/MFS, so that MLS can ship while downstream systems are rethought.
89. As the business owner, I want manual expert final stitch to remain supported, so that final-stitch pipeline quality is not tied to unfinished automation.
90. As the business owner, I want flow warp and radial distortion treated as later quality experiments, so that MVP stability comes first.

## Implementation Decisions

- Build Phase 1 around the non-negotiable core loop: startup/resume/setup, Operator Console readiness, default PTGui template loading, mapping verification, overlay verification, REC/OCR consensus, take sorting, alarms, recorder health, minimal TC status, and sidecar/manifest updates.
- Build Phase 2 around production handoff: DeckLink/RP188 provider, embedded MOV `tmcd`, TC self-test, degraded TC fallbacks, Live Stitch program overlays, PDF reports, per-take GPS/IMU, and stream comment TC stamping.
- Build Phase 3 around setup/calibration utilities and quality improvements: PTGui still grab, System Calibration, analyzer corpus hardening, flow warp experiments, radial distortion experiments, and in-app media browser.
- Keep Phase 4/later for VPS stream recording/playback, stock pipeline automation, and in-app build checklist. MFS automation and NMM/Meridian Media Manager integration are not current MLS targets.
- Treat PTGui `.pts` as the native calibration format. Do not introduce a new Spheris-only calibration format.
- Use one global updateable Spheris XL default PTGui template for MVP, copied into each project when active.
- Disallow calibration/template switches during active recording.
- Make camera OCR roll/clip authoritative for Working Mode production folder identity.
- Treat unreadable roll/clip OCR as a mission-critical readiness/analyzer failure.
- Use the red record bug as the MVP recording state gate. TC numeral color is redundant and non-gating.
- Use a 500 ms start consensus window.
- Any red-REC-triggered event writes proxy files. Consensus decides classification, sort folder, metadata, and alarms.
- Classify normal 9/9 takes to `takes/`.
- Classify 8/9 `CAMERA_FAILED_TO_RECORD`, 1/9 `CAMERA_RECORDED_ALONE`, and middle partial `RECORD_CONSENSUS_FAILED` to `qc_fail/`.
- Classify `CAMERA_STOPPED_LATE` as warning; the original take can stay in `takes/` if the late camera eventually stops.
- Use Setup Mode for disposable tests and verification. Setup recordings go to `_SETUP_TESTS/`, not production take lists.
- Always route new/resumed jobs through Setup Mode before Working Mode.
- Never hard-lock Working Mode on non-green readiness. Require acknowledgement instead.
- Use an Operator Console with modules for Dashboard, Setup, Cameras/Mapping, Overlay/QC, Calibration, Slate, Stream, GPS/IMU, Reports, and Diagnostics.
- Keep 9-grid SDI mapping and Live Stitch/PTGui mapping as separate validations in the same module.
- Default Live Stitch mapping follows verified 9-grid positions into standard PTGui order 1-9.
- Add advanced Template Mapping Override for nonstandard PTGui templates and persist it visibly.
- Make 9-grid QC the primary operator shooting monitor.
- Do not burn MLS status/QC overlays into recorded 9-grid proxy by default.
- Add Live Stitch program overlays that are intentionally recordable when enabled.
- Default production-client Live Stitch recordings to overlays on; stock/clean workflows can turn recorded overlays off.
- Use RP188 over SDI as certified primary TC. Visual/OCR TC is backup. GPS/system time-of-day is last resort and requires confirmation.
- Use array median/consensus for RP188 drift detection; Position 1 is default proxy TC stamping source.
- Fail over primary TC to the lowest-numbered valid RP188 source and do not silently switch back mid-take.
- Production-valid proxy MOVs require true embedded QuickTime `tmcd` from RP188.
- Degraded no-RP188 recordings should not appear camera-certified.
- Keep GPS/IMU MVP scope, but never make it blocking.
- Store GPS/IMU per take with 2-5 seconds pre/post context, aligned to camera TC when possible, and export both JSON and VFX-friendly CSV.
- Refine the existing stream system rather than rebuilding it. Add TC-stamped comments and blurred standby when not recording.
- Generate end-of-day PDF camera reports plus JSON/report package artifacts.
- Keep MLS outputs self-contained. Do not block on NMM/MFS behavior.

### Deep Modules To Build Or Refine

- Startup/Session Recovery: owns startup routing, persisted app session state, interrupted session detection, and recovery choices.
- Mode/Readiness Controller: owns Setup Mode vs Working Mode, readiness context hashes, acknowledgements, and stale/green/yellow/orange/red status.
- Operator Console Model: owns dashboard sections and module state independent of AppKit rendering.
- Mapping Verifier: owns SDI/9-grid/PTGui position state, default mapping, override mapping, and persistence.
- Calibration Template Manager: owns default/imported PTGui template lifecycle, project copy, active-template state, and recording guards.
- ROI Profile Store and Overlay Verification Runner: owns saved ROIs, verification roll workflow, passive stable-field checks, and evidence.
- Recording Consensus Engine: owns REC bug state timelines, 500 ms consensus, anomaly classification, and start/stop rules.
- Take Lifecycle Coordinator: owns take identity, folder sorting, sidecar writes, manifest rollup, false roll, late stop, and interrupted media state.
- Alarm Controller: owns severity, visual priority, acknowledgement, sound/TTS hooks, active/resolved state, and event persistence.
- Recorder Health Monitor: owns independent Live Stitch and 9-grid writer health, stalls, dropped frames, disk state, and protected fallback behavior.
- Timecode Service: owns RP188 source selection, failover, degraded status, visual TC fallback, drift math, and MOV `tmcd` handoff.
- Live Stitch Program Overlay Renderer: owns overlay presets, placement, live/record toggles, TC/roll/clip/slate data binding, and live-only recording border.
- Slate Service: owns sticky job fields, quick scene/shot/take editing, REC-start snapshots, last-take edits, and auto-increment behavior.
- Motion Data Source and Take Slicer: owns GPS/IMU source lifecycle, raw packet archive, parsed samples, TC alignment, dropout/status, and per-take JSON/CSV.
- Stream Comment Service: owns TC-stamped comments, immediate project log persistence, stream standby/rolling state, and take association.
- Report Generator: owns end-of-day PDF, report JSON, media inventory, comments, GPS/IMU summaries, manifest snapshot, and shoot-day grouping.
- Analyzer Corpus Harness: owns reference frames, expected overlay outputs, RED OCR regression tests, and hardware-free simulator timelines.

## Testing Decisions

- Tests should verify external behavior and durable data contracts, not private AppKit layout details.
- Build core logic as deep modules with small interfaces so agents can test without live cameras.
- REC consensus tests should run from simulated per-camera REC timelines and OCR snapshots.
- Take lifecycle tests should verify folder sort, sidecar fields, manifest rollup, false roll, late stop, and interrupted media recovery.
- OCR/analyzer tests should use saved RED overlay reference frames for standby, recording, clip advancement, card missing, sync lost/degraded, high roll numbers, different letters, and supported output profiles.
- TC tests should use captured-frame metadata fixtures for RP188 present, primary failover, no RP188 degraded mode, visual TC fallback, GPS/system fallback confirmation, and drift thresholds.
- Alarm tests should cover severity priority, acknowledgement, clear/resolved states, audio/TTS hook suppression, and visual category mapping.
- Mapping tests should cover standard 9-grid-driven mapping, template override, mismatch/unverified states, and persistence.
- Calibration tests should cover default template load, project copy, import/replace, recording-time switch blocking, and per-take active template capture.
- Recorder tests should use fault injection for Live Stitch writer failure, 9-grid writer failure, disk low, writer stall, and dropped frames.
- GPS/IMU tests should use recorded NMEA/UBX samples and verify parsed samples, raw archive slices, TC alignment confidence, per-take pre/post buffers, and CSV output.
- Stream comment tests should verify immediate persistence, TC source confidence, standby/rolling visibility state, and final take association by TC range.
- Report tests should render deterministic report data from fixtures and verify PDF/package content at a data-contract level.
- UI tests should focus on mode routing and critical operator affordances: startup/recovery, Setup Mode orange state, Working Mode entry acknowledgement, Open Record Folder, alarm acknowledgement, and report export.
- Hardware-dependent DeckLink/RP188 and RED SDI behavior should have manual certification checklists plus mock/provider fixtures for CI-like local testing.

## Out of Scope

- NMM/Meridian Media Manager integration.
- MFS automation.
- Full automatic final stitch pipeline.
- Motion-base-grade playback guarantee from GPS/IMU data.
- Live stream/VPS session recording for MVP.
- Stock pipeline automation for MVP.
- In-app build checklist for MVP.
- Flow warp and radial distortion as required MVP quality gates.
- Full in-app media browser for MVP.
- Creating a new non-PTGui calibration format.
- Preventing recording because GPS/IMU is missing.
- Preventing Working Mode because readiness is non-green, except for explicit operator acknowledgement.

## Further Notes

- The app should bias toward preserving evidence. If there is any red REC evidence, write files and sort/flag later.
- The 9-grid and Live Stitch outputs are the main assets. If only one fallback can be protected, protect 9-grid.
- Degraded conditions should be loud during recording when operationally critical and durable after recording in sidecars/reports.
- MLS should not hide uncertainty. Timecode source confidence, OCR confidence, GPS/IMU lock/alignment, mapping override, and partial recording evidence must be visible and persistent.
- Agent work should start with Phase 1 slices because everything else depends on reliable setup, recording identity, take lifecycle, and operator alarms.
