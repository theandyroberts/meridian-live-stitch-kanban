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
| 4 | Default PTGui Template and Active Calibration Lifecycle | AFK | Initial implementation complete in working tree; `swift build` passed. Default/global template selection, project `_CALIBRATION/` copy, manifest/take sidecar persistence, and recording-time calibration lock are wired. | Run app and verify new job starts from global default, resumed job restores project template, import/replace copies into `_CALIBRATION/`, and take sidecars include template metadata. |

## Ready Now

| ID | Card | Type | Why ready | Definition of done |
| --- | --- | --- | --- | --- |

## Phase 1 Blocked

| ID | Card | Type | Blocked by | Unblocks |
| --- | --- | --- | --- | --- |
| 5 | 9-Grid and PTGui Mapping Verification | HITL | 2 app verification, 4 app verification | 6, 26 |
| 6 | ROI Profile Store and Overlay Verification Roll | HITL | 2 app verification, 5 | 7, 8, 25 |
| 7 | 500 ms REC Consensus and Recording Classification | AFK | 6 | 8, 9, 10, 25 |
| 8 | Authoritative Roll/Clip OCR Take Identity | AFK | 6, 7 | 9, 25 |
| 9 | Take Lifecycle Sorting, Sidecars, and Manifest Rollup | AFK | 7, 8 | 12, 13, 14, 15, 16, 21, 24, 29 |
| 10 | Alarm Controller and Acknowledgement Model | AFK | 7 | 11, 12, 13 |
| 11 | Live 9-Grid Recording/Alarm Overlays | HITL | 10 | 28 |
| 12 | Independent Recorder Health and Protected 9-Grid Fallback | AFK | 9, 10 | Reliability baseline |
| 13 | Late Stop and False Roll Flows | AFK | 9, 10 | Cleaner take lifecycle |
| 14 | Setup Test Recording Sandbox | AFK | 2 app verification, 9 | Setup artifact isolation |
| 15 | Minimal TC Source Status and Degraded-TC Marking | AFK | 9 | 17, 20, 22, 23, 24 |
| 16 | Open Record Folder Action | AFK | 9 | Operator review workflow |

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

1. Get operator visual signoff on card 1's revised startup/new-job flow.
2. Get operator/design review on card 3's initial Operator Console shell.
3. Verify card 4 in the running app: global default starts new jobs, resume restores project template, and import/replace writes `_CALIBRATION/`.
4. After card 4 verification, start card 5: 9-Grid and PTGui Mapping Verification.
5. Keep card 25 in view; if REC/OCR work stalls, pull the analyzer harness forward before deeper UI work.

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
