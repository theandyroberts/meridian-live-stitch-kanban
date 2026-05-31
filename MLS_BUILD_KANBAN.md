# Meridian Live Stitch Build Kanban

Source: `docs/MLS_AGENT_TASKS.md` and `docs/MLS_MVP_PRD.md`.

This is the working build board for agent assignment. Status reflects the current repo state and dependency graph, not a formal external issue tracker.

Legend:
- AFK: agent can implement without live human/hardware review once dependencies are satisfied.
- HITL: needs operator, design, or hardware review before acceptance.
- Ready: dependencies are satisfied enough to start.
- Blocked: do not assign until blockers move forward.

## Done / Needs Verification

| ID | Card | Type | Current note | Next action |
| --- | --- | --- | --- | --- |
| 1 | Startup, Resume, and Recovery Shell | AFK | Refined in `1f62d11` after operator feedback. Actual app smoke pass: first screen now shows Open Existing Job, Start New Job, and recovery only when interrupted state exists; Existing Job routes through a picker into Setup Mode; New Job now includes save folder and PTGui template selection. `swift build` passed. | Operator visual signoff on revised startup/new-job flow, then verify archive/ignore recovery once a real crash state is present. |
| 2 | Setup Mode and Working Mode Gate | AFK | Implemented in `35cf7cb`; app smoke-verified 2026-05-30. Existing job opens Setup Mode, warning acknowledgement is shown for non-green readiness, and Enter Working Mode launches the Operator Console on existing-footage playback. | Operator signoff on visual treatment across all active windows; then use as dependency for Operator Console and mapping work. |
| 3 | Operator Console Readiness Dashboard | HITL | Initial OC shell implemented in `1994477`; `swift build` passed and actual app smoke pass rendered the OC. Adds Operator Console terminology, Working Mode/source/alarm header pills, module tabs, job/project context, readiness cards with green/yellow states, embedded previews, source telemetry, and existing REC controls. | Operator/design review for layout clarity and module terminology before deeper OC modules are accepted. |
| 4 | Default PTGui Template and Active Calibration Lifecycle | AFK | Core artifact path app-verified 2026-05-30. Launch copied active PTGui to `_CALIBRATION/calibration_1bb8cd94.pts`; `manifest.json` records project-local template path and SHA-256; fresh `take_007` sidecar records project-level and take-level `calibrationTemplate`. | Operator test import/replace in Setup Mode and between takes, plus disabled-during-recording lock. |
| 5 | 9-Grid and PTGui Mapping Verification | HITL | Initial mapping module implemented in `a41c49d`; `swift build` passed and actual app smoke pass rendered the Cameras tab. It shows SDI/playback source, 9-grid cell, OCR ID, Pinwheel position, PTGui slot, expected Live Stitch ID, status, standard-template indicator, and side-by-side 9-grid/stitch previews. Job/manifest now persist the template mapping override state. | Operator/hardware review on the live array, then add the advanced override editing flow if a nonstandard PTGui template is encountered. |
| 6 | ROI Profile Store and Overlay Verification Roll | HITL | Initial shell landed in `ae1bc2e`; close-loop landed in `9cefa66`. Overlay/QC now has context-keyed ROI fallback, fast/full verification actions, VisualAnalyzer ROI injection, repo-local DB resolution, 9/9 REC observation, majority roll/clip advance observation, and pass-state persistence. `swift build` passed; app smoke pass verified startup through Overlay/QC on existing footage. | Live-array verification: confirm Fast Test passes after a forced roll/clip advance, Full Test requires stop after REC, and stale/missing ROI stays yellow/non-blocking. |
| 7 | 500 ms REC Consensus and Recording Classification | AFK | Implemented in `feee6fa` after the 500 ms tolerance change in `9cefa66`; SDI dropout grace landed in `e6a5ca0`. Partial REC states that persist past 500 ms now start recording before posting per-camera CAMERA_FAILED_TO_RECORD or CAMERA_RECORDED_ALONE events. Missing SDI signal is handled separately with a 1s debounce and does not stop recording or create loud alarm spam. `swift build` passed. | Live-array verification across 9/9, 8/9, 1/9, 5/4 split, and bump/dropout patterns; confirm event log, sidecar flags, Operator Console REC state, and folder sorting remain coherent. |
| 9 | Take Lifecycle Sorting, Sidecars, and Manifest Rollup | AFK | Core lifecycle routing landed in `6d8340a`. Closed takes now get explicit `takeClassification` and `classificationReason`; critical REC consensus flags route to `qc_fail/`; false-roll flags route to `false_roll/`; warning-only issues such as SDI dropout remain in `takes/` with QC metadata. Sidecar filenames now follow the final folder name, clip-level QC flags are populated, and manifest rollup scans all sort folders. `swift build` passed. | Hardware/OCR validation: record 9/9, 8/9, 1/9, and middle partial patterns, then inspect folder placement, per-take `data.json`, and `manifest.json`. False-roll routing still needs the operator action from card 13. |
| 10 | Alarm Controller and Acknowledgement Model | AFK | Initial alarm model landed in `cebfcdd`. Critical REC/sync/genlock alarms are sticky, drive grid-cell visuals plus the Operator Console REC-pill alarm text, and trigger beep + AVFoundation TTS. Warning-only metadata such as clip mismatch and SDI dropout stays quiet/log-only. Operator Console now has a Silence Audio action that stops current TTS without clearing visual state. `swift build` passed. | Operator/audio review: verify alarm wording, sound level, silence behavior, and whether additional alarm classes should be loud or visual-only. |
| 12 | Independent Recorder Health and Protected 9-Grid Fallback | AFK | Initial protected fallback landed in `16340d6`. The 9-grid writer starts first and is treated as the required protected recorder. If Live Stitch writer start fails, the take can continue with 9-grid recording active. Production sidecars now include structured `recordingOutputs` for Live Stitch, 9-grid, and per-camera proxies, plus `protectedFallbackAvailable`. `swift build` passed. | Add live writer-stall/drop-frame health during active takes, then fault-inject Live Stitch/9-grid writer failures to verify alarms, sidecar fields, and 9-grid protection behavior. |
| 14 | Setup Test Recording Sandbox | AFK | Initial sandbox landed in `40cbb0f`. Project layout now includes `_SETUP_TESTS/`; setup-test recordings use `jobname_setup_test_0001` sequence naming, write Live Stitch/9-grid/per-camera proxy files plus a debug JSON, and skip production sidecars/manifest rollup. Operator Console Setup tab has a Record Setup Test action and blocks setup-test start while production recording is active. `swift build` passed. | App smoke verification: run a setup test, confirm files land only under `_SETUP_TESTS/`, and confirm `manifest.json` production `takes[]` is unchanged. |
| 15 | Minimal TC Source Status and Degraded-TC Marking | AFK | Minimal TC status service landed in `c016972`. The router now tracks the best observed timing source, and per-take sidecars store start/end `TimecodeStatus` with source, confidence, certified/degraded flags, primary slot/camera, display TC, frame count, and detail. Current AVFoundation/file-playback paths are explicitly degraded/approximate until RP188 providers land. Operator Console Timecode readiness copy now reflects non-certified/degraded modes. `swift build` passed. | Validate sidecar output on a short playback take; later cards still need DeckLink/RP188 capture, failover, and embedded QuickTime `tmcd`. |
| 16 | Open Record Folder Action | AFK | Implemented in `75d82cb`. The Operator Console project summary now has an Open Record Folder action that ensures the project layout exists and opens the current project root containing `takes/`, `qc_fail/`, and `false_roll/`. `swift build` passed. | App smoke verification: click the action in Working Mode and confirm Finder opens the active project folder. |

## Ready Now

| ID | Card | Type | Why ready | Definition of done |
| --- | --- | --- | --- | --- |
| 8 | Authoritative Roll/Clip OCR Take Identity | AFK | Slices landed in `fca62f6`, `c5fff9e`, and `984b1a6`: Overlay/QC has a Clip Sync report action, unreadable/mismatched OCR is logged as non-loud `CLIP_MISMATCH`, clip-sync readiness persists into Overlay/QC rows, and production take renaming now requires at least 5 matching stable OCR cameras. If consensus fails, the take keeps its interim folder and gets a sidecar QC flag. | Hardware/OCR validation with real overlays: prove OCR reaches 5/9+ reliably, tune ROI/Vision settings if needed, and then carry clip-sync status into daily reports. |
| 11 | Live 9-Grid Recording/Alarm Overlays | HITL | Alarm controller and REC consensus foundations are now in place. The remaining work is visual treatment on the primary 9-grid monitoring surface. | 9-grid shows red REC outlines/bugs for recording cameras and orange/yellow warning overlays by camera without recording app-only overlays into clean assets unless explicitly enabled. |
| 13 | Late Stop and False Roll Flows | AFK | Take lifecycle routing and alarm silence behavior are in place. | Late-stop cameras create `CAMERA_STOPPED_LATE` warning metadata while the original take can stay good; false-roll operator action moves the take to `false_roll/`. |

## Phase 1 Blocked

| ID | Card | Type | Blocked by | Unblocks |
| --- | --- | --- | --- | --- |

## Phase 2 Backlog

| ID | Card | Type | Blocked by | Notes |
| --- | --- | --- | --- | --- |
| 17 | DeckLink/RP188 Capture Provider | HITL | 15 | Needs hardware/capture certification. |
| 18 | Primary TC Source Failover and Drift Evidence | AFK | 17 | Position 1 default, median drift, failover without silent switchback. |
| 19 | Embedded QuickTime `tmcd` and Proxy TC Self-Test | AFK | 17, 18 | Production-valid proxy milestone. |
| 20 | Live Stitch Program Overlay System | HITL | 15 | TC, roll/clip, slate, presets, live/record toggles. |
| 21 | Slate Service and Operator Console Slate Controls | HITL | 9, 20 | Sticky job fields, quick scene/shot/take, REC-start snapshot. |
| 22 | GPS/IMU Motion Data Source and Per-Take Slicing | HITL | 15, 17 | Logging-first simpleRTK2B Fusion support; TC alignment. |
| 23 | Stream Comment TC Stamping and Standby Blur | AFK | 15, 20 | Refine existing stream system. |
| 24 | End-of-Day Camera Report Package | HITL | 9, 15, 21, 22, 23 | PDF-first report plus JSON/package artifacts. |

## Phase 3 Backlog

| ID | Card | Type | Blocked by | Notes |
| --- | --- | --- | --- | --- |
| 25 | Analyzer Corpus and Simulator Harness | AFK | 6, 7, 8 | Can move earlier if OCR/REC reliability becomes the limiting risk. |
| 26 | PTGui Still Grab Support Tool | HITL | 4, 5 | TIFF preferred, overlay-off/on prompts, calibration still manifest. |
| 27 | System Calibration Mode | HITL | 25 | Troubleshooting workflow beyond per-job verification. |
| 28 | 9-Grid Camera Punch-Up | AFK | 11 | Disabled during recording. |
| 29 | Flow Warp Validation Harness | AFK | 4, 9 | Post-MVP quality experiment unless it proves safe and high value. |
| 30 | Radial Distortion Parity Experiment | AFK | 4, 29 | Keep disabled until PTGui parity is proven. |

## Suggested Next Pulls

1. Implement card 13's late-stop and false-roll flows now that lifecycle routing exists.
2. Add live recorder stall/drop-frame health to extend card 12 beyond close-time/fallback metadata.
3. Validate cards 8, 9, 10, 12, 14, 15, and 16 in-app/on-array: OCR consensus, folder routing, sidecars, alarms, recorder fallback, setup-test sandbox, degraded TC metadata, and Finder folder opening.
4. Prepare card 17 DeckLink/RP188 capture provider once hardware is connected for certification.
5. Get operator/hardware signoff on cards 5, 6, and 7 with the physical array connected.

## Acceptance Gates Before Phase 2

- Startup/resume/recovery verified on the app.
- Setup Mode and Working Mode are explicit and visually distinct.
- Default PTGui template loads for every job and active template is recorded.
- Mapping and ROI/overlay verification are usable in Setup Mode.
- REC consensus uses 500 ms and classifies 9/9, 8/9, 1/9, and middle partial patterns.
- Roll/clip OCR drives production take identity.
- Take sidecars and manifest update after every closed Working Mode take.
- 9-grid and Live Stitch recording health are visible and independent.
- Degraded/non-certified TC status is visible and persisted.
