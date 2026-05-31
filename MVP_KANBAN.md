# Meridian Live Stitch MVP Kanban

Source of truth: `docs/Meridian_Live_Stitch_Master_Spec_2026_05_24.pdf`, `AGENTS.md`, and current Swift/Metal implementation.

This board tracks the gap between the current app and the MVP. Items are phrased as vertical slices where possible: a visible behavior, the data it produces, and the verification needed to trust it on set.

Related planning artifacts:
- PRD: `docs/MLS_MVP_PRD.md`
- Agent task list: `docs/MLS_AGENT_TASKS.md`
- Build kanban: `docs/MLS_BUILD_KANBAN.md`

## Decisions

- Production-valid MVP capture requires a source that provides real per-frame RP188 timecode. DeckLink/RP188 is the certified path for now. File playback and AVFoundation remain useful for development, preview, rehearsal, and degraded fallback, but recordings from lower-confidence sources must be visibly marked in sidecars/reports.
- Emergency fallback may record SDI video feeds even when RP188 timecode is unavailable. Those takes are lower-confidence for TC-dependent workflows and must be visibly marked in the UI, sidecar, and camera report.
- Degraded TC is not burned into the saved stitched proxy. During recording, MLS shows a clear control-panel warning; after recording, the sidecar and camera report carry the degraded-TC flag. The 9-grid remains the forensic source for camera overlay evidence.
- RP188 drift detection uses array median/consensus so one bad primary source does not make eight good cameras look wrong. Position 1 remains the default proxy timecode stamping source unless failover/config changes it.
- If the primary TC source loses valid RP188, MLS automatically fails over to the lowest-numbered Pinwheel position with valid RP188, warns the operator, and logs the source change. It should not silently switch back mid-take.
- If SDI video is present but no input exposes valid RP188, MLS still records in explicit degraded-TC mode using host-clock/frame-count timing only as fallback metadata. TC drift is unavailable rather than failed; stream comments and GPS/IMU alignment are approximate.
- The array's master timecode/genlock hardware owns shutter and TC synchronization. MLS does not correct sync; it monitors for evidence that the physical sync chain has failed. RED HUD sync/genlock indicators and RP188 numeric drift are independent evidence paths; either path can raise warnings/failures, and reports must preserve which evidence fired. Spec RP188 thresholds are alarm rules: `>1 frame` raises `TC_DRIFT_SOFT`, `>5 frames` raises critical `SYNC_LOST`.
- Production-valid stitched and 9-grid proxy MOVs must carry true embedded QuickTime `tmcd` timecode sourced from RP188. Sidecar/report TC is not enough for the production-valid milestone, though sidecar-only can be tolerated as a known temporary gap for internal tests.
- Degraded no-RP188 proxy MOVs should omit embedded `tmcd` when technically possible. If the writer/container requires timing metadata, it must be clearly marked as host-derived in sidecar/report and must not appear as camera-certified TC.
- Record start/stop detection for MVP uses the red record bug in the top-right overlay. TC numeral color is not required for the record-state gate.
- Record-state consensus uses a 500 ms window, not 2 seconds. If all 9 cameras show the red record bug inside 500 ms, the take is normal. If 8/9 record and one does not, or 1/9 records alone, MLS must log that fact clearly for downstream QC even if the proxy recording behavior differs by case.
- Any red-REC-triggered event writes proxy files; consensus never decides whether files are written. Consensus decides status, sort folder, and metadata. Normal 9/9 takes land in `takes/`; failed consensus takes land in `qc_fail/` with explicit per-camera evidence for MMM and the camera report.
- Middle partial-record patterns, such as 5/9 cameras showing REC inside the 500 ms window, use a new critical anomaly class: `RECORD_CONSENSUS_FAILED`. This avoids forcing ambiguous failures into inaccurate 8/9 or 1/9 labels.
- Stop consensus is different from start consensus. If one or more cameras keep recording after the others stop, MLS warns loudly on the control panel while they remain recording. Once they stop, the original take can still be considered good; sidecar/report should note the longer-running camera(s) and their stop offsets. Because the physical REC button is a toggle, a new take trigger while a camera is still recording will stop that camera while starting the others, presenting as an 8/9 `CAMERA_FAILED_TO_RECORD` condition on the new take rather than a separate overlap class.
- `CAMERA_STOPPED_LATE` is a warning: the take can stay in `takes/`, while sidecar/report record which camera(s) stopped late and by how many frames/seconds.
- Critical operator errors need multimodal alarms: flashing visual outline/text on the control panel, assigned warning/error sounds, and spoken TTS recitation of the issue so the operator cannot miss it while working the shoot.
- Critical alarms require operator acknowledgement. Visual flashing persists until acknowledged or superseded; audio/TTS repeats at intervals rather than continuously. Warning alarms may auto-clear visually when resolved, but remain in event logs, sidecars, and reports.
- TTS is for critical alarms only by default. Warnings get visual text and non-spoken sound unless the operator enables a verbose alert mode.
- Alarm sounds are grouped by severity/category rather than unique per event: critical recording failure, critical sync failure, warning, and info/no-sound by default. Text/TTS carries the specific error details.
- Acknowledging a critical alarm silences audio/TTS only. The visual indicator remains until the condition resolves, or changes to an acknowledged-still-active state.
- Alarm visual priority: red is reserved for recording state. Orange is reserved for ultra-critical shoot-threatening errors. Yellow/amber is for known or lower-urgency warnings that should inform the operator without dominating the shooting monitor. Clip mismatches after a known missed/partial recording are yellow/amber because the system already knows clips are out of sync and the crew may intentionally keep shooting until there is time to correct. Power warnings are low-urgency and should not be visually annoying. TC GEN / sync loss are critical; TC drift should not dominate if GEN/sync remain healthy, but is still logged and shown as a lower-priority warning.
- Initial current-take record failures, such as 8/9 REC or a middle partial-record pattern inside the 500 ms start window, are orange critical events with sound/TTS. The continuing aftermath, such as known clip-number offset on later takes, becomes yellow/amber status unless a new critical failure occurs.
- Clip-offset status remains yellow/amber until the operator manually catches up behind cameras and MLS verifies roll/clip consensus is restored. A `Verify Clip Sync` / `Check All Settings` action should show a large readable status report of tracked overlay fields and TC across all cameras.
- The 9-Grid QC should support a temporary single-camera punch-up for focus/settings inspection, such as right-clicking a camera cell to briefly make it full screen. The punch-up exits on next click/key stroke and should be disabled during recording.
- `Check All Settings` is available in both Setup Mode and Working Mode. During active recording it is disabled or read-only and must not interrupt recording.
- 9-grid punch-up replaces the 9-grid view temporarily rather than opening a separate floating window. It returns to the grid on the next click/key stroke and is disabled during recording.
- Each new job setup must run a forced overlay verification once the physical camera system, RED overlays, and baseline settings are established. The required test actively forces a roll/cut cycle and verifies REC bug detection plus roll/clip advancement, because REC depends on analog button/distribution behavior and is the most failure-prone path. Stable overlay fields such as FPS, shutter, ISO, WB, resolution, codec, sync/genlock, media, and power are passively verified against the current expected baseline during setup rather than requiring forced status changes every job.
- ROI positions are normally loaded from saved profiles keyed by SDI/output profile, not manually drawn every job. Overlay Verification confirms the saved profile still works. Manual ROI drawing/editing is an advanced repair/setup tool.
- MLS has a Setup Mode for building/testing the physical array before real shooting begins. In Setup Mode the app runs capture, analyzer, verification, alarms, and optional proxy recording, but test rolls/false rolls/formatting rolls are setup artifacts and do not count as production takes for MMM/MFS. Entering Job/Working Mode starts proper take classification into the job folders.
- Setup Mode UI must be unmistakable: full orange overlays/markers across all MLS windows, visually distinct from QC warnings. A large green button transitions into Working/Job Mode. Setup artifacts are cleared by default when entering Working Mode; the operator explicitly chooses to keep them only when needed.
- MLS must never hard-lock the operator out of Working Mode. The setup screen shows a readiness/status bar for all verification categories; green means verified/good-to-go, non-green requires visible warning/acknowledgement but does not block entering Working Mode. Opening an existing job at a new day/startup/after lunch should land in Setup Mode with previously verified items already green when still valid.
- Verification freshness is context-based. Previously verified items stay green only when the relevant context has not changed: ROI profile, SDI/output resolution profile, camera mapping, expected settings, calibration/job setup, capture device mapping, and verification timestamp. Changed/stale items become yellow `Needs Recheck`.
- Setup Mode rolls do not appear in the production `manifest.json` `takes[]` list, camera report clip list, or MMM/MFS ingest. Camera report setup content is limited to verification/readiness status: settings checked, overlay verification state, warnings acknowledged, and whether everything looked good when entering Working Mode. Setup clip filenames, setup roll counts, and setup recording details are not pertinent to the production camera report.
- In Working Mode, MLS writes each per-take `data.json` sidecar as the take closes and updates the project-level `manifest.json` after every closed take. Wrap finalizes/locks the manifest and report rather than being the first time the rollup exists.
- Setup Mode has a very visible `Record Setup Tests` toggle so the team can intentionally capture test drives/live stitch/9-grid/proxy/JSON evidence while building the system. Those artifacts are disposable/debug materials and are excluded from production manifests/reports unless the operator separately keeps/exports them for testing.
- Setup test recordings live inside the project root under `_SETUP_TESTS/` and use jobname-prefixed sequence naming, such as `jobname_setup_test_0001`. They can include live stitch, 9-grid, optional per-camera proxies, and debug JSON/event data, but never production roll/clip naming.
- A future v1.5/2.0 feature can bring the existing external build checklist into MLS: serial numbers, firmware versions, setup confirmations, camera settings, and completion checks as part of job build/setup. MVP only records that setup verification passed and readiness looked good.
- Working Mode production folder/file naming must respect camera OCR roll/clip values. MLS must not invent its own production counter because jobs may resume at arbitrary roll/clip values, such as day-three roll 701 or post-lunch roll 003 clip 015. Unreadable roll/clip OCR is a serious readiness/problem state that should be troubleshot rather than silently bypassed.
- REC detection and roll/clip OCR are mission-critical product functions. If a card is present, formatted properly, and the RED overlay is visible, MLS is expected to read roll/clip reliably. Unreadable roll/clip is a mission-critical app readiness/analyzer failure, not a normal workflow. Emergency containment can exist as a last resort, but the product goal is to fix OCR before shooting and make this path bulletproof.
- Working Mode needs a fast `Open Record Folder` action that opens the primary project recording folder containing `takes/`, `qc_fail/`, and `false_roll/`. A richer in-app media browser for fast playback belongs to v1.5/2.0.
- The third working window is named the Operator Console, abbreviated `OC`. It holds setup/status/settings/controls distinct from the 9-grid QC window and the live stitch window.
- Working Mode window roles are: Live Stitch for stitched panorama preview/output, 9-Grid QC for all camera feeds with overlays, and Operator Console for controls/readiness/alarms/recording/stream. Recording status and alarms must also overlay on the 9-Grid QC cells because that is the operator's primary shooting monitor and it must show exactly which camera has an issue.
- MLS status/QC UI overlays on the live 9-Grid QC, such as recording borders, alarm outlines, alarm text, and transient confirmations, are not burned into the recorded 9-grid proxy by default. Deliberate program overlays are different: SDI timecode, roll/clip, and slate/client-facing metadata must be available as recordable overlays for the Live Stitch output when enabled.
- v1 needs operator-controlled Live Stitch program overlays for on-set review assets. The crucial fields are SDI/RP188 timecode and roll/clip; slate fields such as director, DP, represented company, scene, shot, and take are also desired. Every program overlay must be removable/toggleable so stock jobs can record a clean Live Stitch. The preferred production-client layout uses the Live Stitch lower letterbox/negative space as an information band, keeping TC/roll/clip off the picture whenever possible. Placement should use a predetermined grid/drag system and overlays can be toggled independently for live display and recording.
- Live Stitch program overlays should ship with presets: `Clean`, `Production Review`, `Minimal`, and `Custom`. Presets provide quick setup, while drag/reorder remains available when placement needs to change.
- The Operator Console needs fast slate editing. Job-level fields such as director, DP, represented company, and project are sticky across the job; scene/shot/take must be quick to edit between takes and feed the Live Stitch program overlay plus manifest/report slate metadata.
- Slate values are snapshotted onto the take at REC start. The operator may retroactively edit the most recent take after cut; MLS updates sidecar/manifest/report and logs the slate edit.
- OC slate `take` auto-increments after each closed Working Mode take. Setup Mode recordings do not increment slate take. If a take is marked false roll, MLS asks whether to keep, increment, or revert the slate take value. Scene and shot remain sticky until edited.
- Production-client mode records Live Stitch program overlays by default. Stock/clean workflows can turn recorded overlays off. Live-display and record-overlay toggles remain separate.
- Live Stitch TC overlay source priority is: RP188 over SDI first; visual overlay/OCR/cropped camera TC second if RP188 is unavailable; last-resort time-of-day from GPS time or computer system time only with operator confirmation. Last-resort TC is useful as an approximate clip-selection reference but is not camera-certified embedded TC.
- Live Stitch TC program overlay should turn red while recording. The Live Stitch screen should also show a red recording border for client/operator confidence, but that border is a live system overlay and is not recorded into the file.
- If RP188 is unavailable and Live Stitch falls back to visual TC, MVP should prefer clean rendered text from OCR/parsed overlay data when reliable. Crop/enlarge of the original camera TC overlay remains an acceptable fallback, especially if fast-changing sub-frame/millisecond display is not OCR-stable relative to display refresh.
- Live Stitch TC overlay uses standard frame timecode format `HH:MM:SS:FF`. MVP should not display milliseconds/sub-frame values as primary TC.
- Live Stitch roll/clip overlay defaults to friendly text such as `Roll 005  Clip 006`, with compact `005_006` as an option. Raw RED-style camera identity remains available in metadata/status views. Showing all camera IDs in the Live Stitch overlay is a possible later option, not MVP-critical.
- Live Stitch program overlays should use a clean, legible, modern broadcast-style font treatment. Prefer white text for slate/roll/clip, TC red while recording and white when idle, subtle shadow/translucent band for contrast, and avoid heavy black RED-style boxes unless needed for readability.
- MVP Live Stitch is real-time GPU playback of a loaded PTGui `.pts` calibration template, with MLS applying remap/blend and optional photometric correction. Flow warp for close-object parallax and radial distortion parity are high-value quality experiments after MVP is working and stable.
- MVP stitch operation works directly from an existing PTGui template. Auto-build/live-frame solve and RED stills precision workflows are post-MVP calibration/stitching tools, but MVP should include a clean still-grab support tool: one button captures overlay-free stills from all camera inputs at the maximum practical resolution/color depth and saves them into the project as PTGui stills for fast template creation.
- PTGui still grab should normally prompt the operator to turn overlays off on all 9 cameras before capture, maximizing usable image area/resolution, then prompt overlays back on after capture. In a hurry, an alternate cropped-overlay mode may capture with overlays still on and crop/mask them out, accepting a little lower usable resolution.
- PTGui still grabs save under `[jobname]/_CALIBRATION/stills/YYYYMMDD_HHMMSS/` with filenames carrying camera letter and Pinwheel position/direction, such as `A_position1_forward.png`, when known.
- PTGui still grabs prefer TIFF when possible, preserving the maximum practical bit depth/color depth from the SDI/capture pipeline. PNG is an acceptable fallback if TIFF export is not available or reliable.
- PTGui still grabs should capture the 9 camera stills from the same frame/timecode as closely as practical, but exact timing is less important when the array/scene is stationary. TIFF encoding/export can happen after capture. Each grab writes a small manifest with per-camera TC/frame index/host timestamp for traceability and warns only on missing feeds or wildly inconsistent capture timing.
- Every job opens with a usable default PTGui template so Live Stitch works immediately, even if the operator intends to create or upload a better job-specific calibration later. During setup, the operator can capture stills or import a new PTGui file; MLS then copies the selected active template into the project `_CALIBRATION/` folder and records it in the manifest.
- Calibration/template switches are logged. Each Working Mode take sidecar records which PTGui template was active. If a new template is selected during setup or between takes in Working Mode, subsequent takes use the new template. Calibration switching is disallowed during active recording.
- MVP uses one global updateable Spheris XL default PTGui template. This can be replaced over time if a better base calibration is found. Creating a new job-specific template is preferred when time allows, but a proven global template, like the long-used Mercy PTGui template, is acceptable as the operational fallback.
- GPS/IMU is MVP scope and important for stock/backend workflows, but it must never block MLS from running or recording. Operator Console should include a GPS/IMU tab with logging enable/disable, map visualization of recorded route, and a small live vehicle/IMU visualization showing real-time orientation/motion.
- GPS/IMU status belongs in the Operator Console monitoring/status area, not the 9-grid. Missing GPS/IMU hardware is a yellow non-blocking status. If hardware is connected but not logging, has no GPS lock, or otherwise fails to produce expected data, show a more serious OC alarm/status, but still never block recording.
- Current GPS/IMU hardware is ArduSimple simpleRTK2B Fusion (`AS-RTK2B-FUSIONF9R-L1L2-NH-01`) with u-blox multiband antenna (`AS-ANT2B-ANN-L1L2-50SMA-00`). On macOS it appears as `/dev/cu.usbmodem21201` and emits NMEA sentences including `GNRMC`, `GNVTG`, `GNGGA`, and `GNGSA`; indoor/no-lock samples show `V`, fix quality `0`, and 0 satellites.
- F9R diagnostic confirmed GPS and IMU/fusion outputs. Near a window the board produced a valid NMEA fix. After requesting UBX NAV/ESF output on USB, it emitted `NAV-ATT`, `NAV-PVT`, `NAV-SAT`, `ESF-MEAS`, `ESF-RAW`, `ESF-STATUS`, and `ESF-INS`, confirming the IMU/fusion stream is available for the MLS driver.
- The quick IMU widget is not a production validation target. Roll appeared pinned at `0.0` and the parsed attitude display felt laggy. MVP GPS/IMU logging should prioritize reliable raw NMEA + raw UBX NAV/ESF archival with host timestamps, plus parsed GPS route/time/fix/speed/course. Smooth roll/pitch/yaw visualization is deferred until F9R configuration, ESF/INS parsing, and sensor mounting/calibration are validated.
- GPS/IMU data must be time-aligned to camera timecode. The preferred alignment source is exact SDI RP188 timecode from the camera frames. Visual/overlay TC is the backup alignment source if RP188 is unavailable. GPS time and computer/system time are useful fallback references, but they are not production-certified camera TC.
- Each Working Mode take sidecar gets its own GPS/IMU data for that take, including aligned samples and summary fields. Raw sensor data should also be take-scoped when kept; avoid full-day raw GPS/IMU dumps as the primary artifact because they are difficult to use downstream. A future MMM tool could parse full-day raw logs into clips if needed, but MVP should produce per-take data directly.
- MVP GPS/IMU sidecar data is targeted at VFX/post first, not motion-base playback. Include parsed, timecode-aligned samples for route/location/speed/course/context, with IMU fields present when valid. Also export a VFX-friendly per-take CSV alongside JSON so teams can inspect/import the data without writing a JSON parser.
- GPS/IMU per-take data should include a small pre-roll/post-roll buffer, roughly 2-5 seconds, while clearly marking the actual take TC range. This gives VFX context at clip edges without creating full-day raw logs.
- Camera report per-take GPS/IMU statuses are: `GPS/IMU OK`, `GPS NO LOCK`, `GPS DROPOUT`, `GPS/IMU DISABLED`, `GPS/IMU MISSING`, `GPS/IMU NOT LOGGING`, and `GPS/IMU DEGRADED ALIGNMENT`. Report summaries include start/end location, distance, speed min/max/avg, fix continuity, and alignment source (`RP188`, overlay TC, or degraded).
- The live stream system already exists with client login and comments. MVP should refine it rather than rebuild it: client comments should be stamped with the current Live Stitch timecode so the chat/comment log can identify parts clients liked and later associate comments with takes.
- Stream comments use the same TC source priority as the Live Stitch overlay: RP188 primary/failover first, visual/OCR TC fallback second, and explicit degraded GPS/system time-of-day only as last resort. Each comment stores display TC, `tcFrames` when available, TC source/confidence, server receipt time, user identity, and message.
- Live stream should show a blurred/obscured standby view with a message when cameras are not recording, and switch to a clear Live Stitch feed while rolling.
- Stream comments are stored immediately in a project-level comment/chat log. MLS may associate comments to takes live for visibility, but wrap/finalize recomputes the authoritative association from finalized take TC ranges.
- MVP does not require recording the VPS/live stream itself. Local MLS Live Stitch and 9-grid recordings are the source of truth. Stream/session recording is a good v1.5 option if useful.
- MVP camera report output should include a human-readable PDF report plus machine-readable JSON. HTML is not useful as the primary human deliverable. The human report should follow familiar production camera-report patterns and ZoeLog-style priorities: clean readable handoff, fast scan by production/post, custom job/crew header, entries organized like a paper camera log, and production-friendly fields rather than an engineering dump.
- Camera reports are typically delivered at end of shoot day, not only at final project/job wrap. MLS needs an end-of-day camera report export flow, while final wrap can still produce the full project package.
- Shoot day boundaries are explicit operator actions in Setup Mode. MLS may suggest a new shoot day on date change, but it should not auto-split without confirmation. Reports group takes by shoot day; each day can export its own camera report while the project manifest remains the whole job.
- End-of-day report export produces a report package folder containing the PDF camera report, JSON report data, manifest snapshot, comments/chat log, GPS/IMU CSV summaries, and media inventory CSV.
- Startup should prominently support resuming an existing job because crews often shut the system down for location moves, lunch, or long breaks. If MLS detects an interrupted/unclosed session, it shows a dedicated Recovery Startup screen, not the generic new/continue flow. Recovery shows job details, last mode, last closed take, in-progress take status, and offers Resume Session, Open Setup Mode for This Job, Start New Job, or advanced archive/ignore.
- Normal startup hierarchy is: Resume Existing Job first and most prominent, Start New Job second, then utilities/diagnostics. If an interrupted session exists, show Recovery Startup instead. New or resumed jobs enter Setup Mode first; Working Mode is entered deliberately via the green button.
- Operator Console / Setup Mode should feel like a clean modern dashboard, inspired by OBS-style operational panels, with Spheris branding. The existing capture-card mapping UI is directionally correct and should be extended with readiness sections for project/shoot day, camera mapping, overlay verification, REC/roll-clip verification, calibration template, recording destination/disk, stream, GPS/IMU, alarms/audio, and Ready to Enter Working Mode.
- OC should use tabs/modules for quick setup and operations. Recommended model: Dashboard, Setup, Cameras/Mapping, Overlay/QC, Calibration, Slate, Stream, GPS/IMU, Reports, and Diagnostics. The 9-grid/SDI camera mapping and Live Stitch/PTGui template mapping are separate validations and must both be visible side-by-side in the same Cameras/Mapping module.
- Cameras/Mapping module should show, per physical camera/slot: SDI input/capture-card channel, 9-grid cell position, OCR camera ID letter, Pinwheel position number, PTGui template camera slot, Live Stitch expected camera ID, and match status (`matched`, `mismatch`, `unverified`).
- Default mapping rule: orient cameras correctly for the 9-grid, and MLS maps those positions to the standard PTGui template camera order 1-9. Because a PTGui template can be created with an unexpected camera order, MLS also needs an override to reorder Live Stitch/PTGui camera assignment separately from the 9-grid when needed.
- Live Stitch/PTGui mapping override is an advanced `Template Mapping Override` control. Normal operators use the standard 9-grid-driven mapping. When override is active, MLS shows a clear warning/status and records the override in job setup/manifest.
- Interrupted/crash recovery must preserve Live Stitch and 9-grid media as first-class assets. If roll/clip identity is known, interrupted media stays in the correct roll/clip folder and is marked `INTERRUPTED_RECORDING` / `NEEDS_REVIEW`, not hidden in a generic folder. If identity is unknown, use an interim folder and surface it loudly. Under severe failure, 9-grid is the acceptable protected fallback output because it preserves all camera evidence.
- Live Stitch and 9-grid recording must use independent recorder pipelines and error states so one can fail without killing the other. 9-grid is the highest-priority protected output. If Live Stitch recording fails, keep 9-grid recording; if 9-grid recording fails, raise a critical alarm because fallback evidence is compromised. Sidecar/report records which outputs succeeded or failed.
- During recording, the Operator Console should show live recorder health: 9-grid writer active, Live Stitch writer active, frames written, dropped frames, disk free, take duration, and last write time. Writer stalls or severe frame drops alarm immediately rather than waiting until stop.
- NMM/Meridian Media Manager integration is not a current MLS implementation target. MLS should still output clean, self-contained folders, sidecars, reports, and sensor/comment data, but downstream NMM behavior will be handled separately later. MFS is on hold; final stitch remains a manual expert-stitcher pipeline for now.
- MLS needs an analyzer test corpus/harness for RED overlay recognition, covering standby/recording, clip advancement, card missing, sync lost/degraded, high roll numbers, different letters, and supported output profiles. A deeper System Calibration mode can guide the team through many settings/status changes and let MLS analyze transitions in real time as a fallback/troubleshooting process when per-job verification or OCR confidence has major issues.

## Now

### Implementation Phases

- Phase 1 issue-ready breakdown: `docs/PHASE1_ISSUES.md`.

- [ ] Phase 1: Non-negotiable core loop.
  - Decision: implement startup/resume/setup shell, OC readiness dashboard, default PTGui template loading, active template import/replace, REC bug + roll/clip OCR verification, Working Mode recording classification, OC + 9-grid warning/alarm overlays, independent 9-grid/Live Stitch recorder health, minimal TC plumbing/status, and per-take sidecar/manifest updates first.
  - Why: MLS must reliably identify takes and record the two primary assets before secondary systems add complexity.

- [ ] Phase 2: Timecode and production handoff.
  - Decision: implement DeckLink/RP188 provider, embedded MOV `tmcd`, proxy TC self-test, degraded TC fallbacks, Live Stitch program overlays, PDF camera report/end-of-day package, per-take GPS/IMU logging, and stream comment TC stamping after the core loop.
  - Why: RP188/timecode is the dependency for overlays, report accuracy, GPS alignment, and client comments, but full MOV timecode should not delay REC/OCR and recording reliability.

- [ ] Phase 3: Calibration utilities and quality improvements.
  - Decision: implement PTGui still grab, deeper System Calibration mode, analyzer corpus/harness hardening, flow warp experiments, radial distortion experiments, and in-app media browser after core loop and timecode/handoff work.
  - Note: analyzer corpus/harness can move earlier if REC/OCR reliability becomes the limiting risk.

- [ ] Phase 4/later: Deferred product extensions.
  - Decision: keep VPS stream recording/playback, stock pipeline automation, and in-app build checklist as later items.
  - Decision: drop/deprioritize MFS automation and NMM/Meridian Media Manager integration for now; those downstream systems need to be rethought separately.

### Capture, Timecode, and Sync

- [x] Certify the capture-source contract.
  - Recommended decision: MVP is certified on RP188-capable DeckLink capture, with AVFoundation/file playback retained as degraded/fallback sources.
  - Why: the spec requires frame-accurate SDI RP188 for proxy timecode tracks, drift detection, stream comments, and GPS/IMU alignment.
  - Decision: production-valid MVP capture requires real per-frame RP188 timecode; degraded sources must be labelled lower-confidence in sidecars/reports.

- [ ] Implement the DeckLink capture provider behind the existing frame spine.
  - Current state: `CapturedFrame` exists and carries optional timecode plus metadata quality.
  - Needed: RP188 extraction, signal state, source identity, frame timing, and adapter-level reconnect behavior.

- [ ] Choose and wire the primary TC source.
  - Spec default: Pinwheel Position 1.
  - Decision: Position 1 is the default proxy timecode stamping source. Drift detection compares cameras to array median/consensus.
  - Needed: setup UI/config, primary-source selection in capture routing, automatic failover to the lowest-numbered valid RP188 source, and no silent switch-back mid-take.

- [ ] Embed camera TC into ProRes MOV timecode tracks.
  - Current state: ProRes recording exists, but manifest comments mark TC fields as nil until RP188 is wired.
  - Decision: production-valid proxies require embedded QuickTime `tmcd` sourced from RP188.
  - Needed: VideoToolbox/AVAssetWriter timecode metadata, plus verification that `tmcd` matches captured RP188.

- [ ] Implement no-RP188 degraded timing mode.
  - Decision: if SDI video exists but no RP188 source is valid, continue recording with a clear `NO RP188 - DEGRADED TC` control-panel warning.
  - Needed: sidecar/report flag, host-clock/frame-count fallback metadata only when needed, omitted/non-certified `tmcd`, TC-dependent feature downgrades, and distinct `NO_RP188_SOURCE` style event.

- [ ] Enforce sync/genlock alarm contract.
  - Decision: hardware generator owns sync; MLS only monitors. HUD sync/genlock failure and RP188 drift are independent evidence paths. `>1 frame` RP188 drift is `TC_DRIFT_SOFT`; `>5 frames` RP188 drift is critical `SYNC_LOST`.
  - Needed: production defaults, visual analyzer evidence wording, RP188 evidence wording, sidecar/report evidence, and diagnostic-only tolerance override if kept.

- [x] Define degraded-TC fallback presentation.
  - Decision context: if SDI video is available but RP188 is not, MLS may still record in emergency degraded mode.
  - Decision: do not burn TC onto the stitch proxy. Show a prominent recording-time warning on the control panel and persist the degraded-TC flag to sidecar/report.

- [ ] Add startup proxy-TC self-test.
  - Spec: record 10 seconds, verify output embedded TC matches capture-time RP188.
  - Output: `PROXY_TC_MISMATCH` warning if verification fails.

### Recording and Take Lifecycle

- [ ] Make roll/clip identity authoritative from visual analyzer consensus.
  - Current state: `ReelClipOCR` exists and `RecordingCoordinator` can rename from interim folders when consensus exists.
  - Decision: camera OCR roll/clip values are authoritative for Working Mode production folder/file naming.
  - Decision: unreadable roll/clip OCR is mission-critical readiness/analyzer failure, not a normal routing case.
  - Needed: robust clip-start/clip-stop consensus, expected roll/clip transitions, OCR confidence diagnostics, verification tests, and no silent fallback when OCR confidence is too low.

- [ ] Sort take folders at clip stop.
  - Spec folders: `takes/`, `qc_fail/`, `false_roll/`.
  - Current state: folder enum exists; sidecars include `sortFolder`.
  - Decision: any red-REC-triggered event writes files. At close, classify and move finalized proxy/sidecar based on consensus/anomaly severity, then log `takeSortedToFolder`.
  - Needed: exact `qc_fail/` metadata for 8/9, 1/9, and middle partial-record groups.

- [ ] Detect critical record consensus failures.
  - Needed: `CAMERA_FAILED_TO_RECORD` for 8/9 advancing, one missing; `CAMERA_RECORDED_ALONE` for 1/9 advancing; `RECORD_CONSENSUS_FAILED` for middle partial-record patterns.
  - Decision: use a 500 ms red-bug consensus window. Log 8/9 and 1/9 failures clearly for downstream MMM QC.
  - Verification: simulated red-bug/OCR sequences for all 9 positions.

- [ ] Handle late-stop cameras without over-failing takes.
  - Decision: cameras that continue recording after array stop trigger a loud control-panel warning and metadata note, but the take can remain good once they stop.
  - Decision: final anomaly name is `CAMERA_STOPPED_LATE`, severity warning, folder stays `takes/`.
  - Needed: per-camera stop offsets and sidecar/report notes. If a new take starts while any camera is still recording, classify the new take through existing 8/9 start consensus failure because REC is a toggle.

- [ ] Add multimodal operator alarms.
  - Decision: critical errors need flashing control-panel outline/text, assigned warning/error sounds, and spoken TTS recitation of the issue.
  - Decision: critical alarms persist until operator acknowledgement; warning alarms may auto-clear visually after resolution.
  - Decision: TTS is critical-only by default; warnings use visual plus non-spoken sound unless verbose alerts are enabled.
  - Decision: use grouped sounds by severity/category, not unique sounds per event.
  - Decision: acknowledge silences audio/TTS but keeps visual state until resolution.
  - Decision: red means recording, orange means ultra-critical, yellow/amber means warning/status. Non-crucial warnings must not dominate the 9-grid or distract from shooting.
  - Needed: alarm taxonomy by severity/category, sound selection, TTS phrase templates, repeat interval, mute/acknowledge behavior, acknowledged-still-active visual state, verbose mode, and test mode.

- [ ] Add false-roll flow.
  - Needed: operator "discard last" affordance, confirmed heuristic path, and move to `false_roll/`.

- [ ] Harden independent recording pipelines.
  - Decision: Live Stitch and 9-grid recorders are independent; 9-grid is the highest-priority protected output.
  - Decision: OC shows live recorder health during takes and alarms on writer stalls/severe drops immediately.
  - Needed: separate writer lifecycle/error states, continue-on-partial-failure behavior, frame/write counters, dropped-frame detection, disk free monitoring, critical 9-grid failure alarm, output success/failure sidecar fields, and recovery/finalization tests.

### Visual Analyzer and QC

- [ ] Build per-camera ROI calibration UI.
  - Current state: `VisualAnalyzer` uses baked ROIs.
  - Decision: saved ROI profiles are the normal path; manual rectangle drawing/editing is an advanced repair/setup tool.
  - Needed: profile storage keyed by SDI/output profile, profile selection, verification status, and repair editor.

- [ ] Add startup overlay verification workflow.
  - Decision: run at every new job setup once camera system and overlays are established.
  - Decision: force a roll/cut cycle to verify REC bug detection and roll/clip advancement.
  - Decision: passively verify stable settings/status fields against expected baseline; do not require toggling sync box or changing WB/shutter every job.
  - Decision: previous verification remains green only while its relevant context is unchanged; changed/stale items become `Needs Recheck`.
  - Needed: guided operator flow, pass/fail evidence per camera, baseline-return prompt after test roll, verification timestamp/context hash, and verification result in sidecar/report.

- [ ] Add Setup Mode before Job/Working Mode.
  - Decision: during setup, MLS may record test rolls and analyzer evidence, but those artifacts are not production takes for MMM/MFS.
  - Decision: Setup Mode gets unmistakable orange overlays/markers across all windows and a large green button to enter Working/Job Mode.
  - Decision: setup artifacts are cleared by default on transition; operator can explicitly keep them.
  - Decision: Working Mode is never hard-blocked; non-green readiness items require warning/acknowledgement, not lockout.
  - Decision: setup rolls stay out of production `manifest.json` takes, camera report clip lists, and MMM/MFS ingest; camera report gets verification/readiness status only.
  - Needed: mode switch UI, readiness/status bar, `setup_tests/` or equivalent artifact area, clear-by-default transition prompt, and verification summary handling.

- [ ] Redesign Operator Console as readiness dashboard.
  - Decision: clean modern OBS-inspired operational dashboard with Spheris branding.
  - Decision: build on the existing capture-card mapping direction, adding readiness/status sections and drill-in details.
  - Decision: use tabbed/modules structure for Dashboard, Setup, Cameras/Mapping, Overlay/QC, Calibration, Slate, Stream, GPS/IMU, Reports, Diagnostics.
  - Decision: verify 9-grid/SDI mapping and Live Stitch/PTGui template mapping separately, side-by-side in the Cameras/Mapping module.
  - Decision: default Live Stitch/PTGui mapping follows validated 9-grid positions into standard template order 1-9; allow separate Live Stitch mapping override for nonstandard templates.
  - Decision: Live Stitch mapping override is advanced, visibly marked when active, and persisted to job setup/manifest.
  - Needed: dashboard layout, tab/module structure, mapping table fields/statuses, section status model, Spheris visual style, Setup Mode orange treatment, Working Mode treatment, and responsive behavior for the OC window.

- [ ] Add setup-test recording toggle.
  - Decision: Setup Mode analyzer/verification always runs, but setup proxy/live-stitch/9-grid recording is controlled by a visible `Record Setup Tests` toggle.
  - Decision: setup tests live under project-root `_SETUP_TESTS/` and use `jobname_setup_test_0001` style sequence names, not production roll/clip naming.
  - Needed: setup recording destination, media retention/clear behavior, JSON/debug artifact handling, and clear visual state when setup recording is enabled.

- [ ] Add Open Record Folder action.
  - Decision: MVP button opens the primary project recording folder containing `takes/`, `qc_fail/`, and `false_roll/`, not a specific latest take.
  - Needed: button placement, folder resolution, and empty/missing-folder state.

- [ ] Consider in-app media browser for v1.5/2.0.
  - Future idea: the Working Mode control/settings window could include a media viewer to quickly browse and play takes without using Finder.

- [ ] Rename/control language around Operator Console.
  - Decision: canonical name is Operator Console (`OC`) for the third working window.
  - Needed: align UI labels, docs, code comments, and future media-browser language.

- [ ] Add per-camera recording/alarm overlays to 9-Grid QC.
  - Decision: 9-Grid QC is the primary operator shooting monitor; it must show recording status and camera-specific alarms directly on affected cells.
  - Decision: live MLS status/QC overlays are not burned into the recorded 9-grid proxy by default; deliberate program overlays are separately toggleable/recordable.
  - Needed: red recording border/REC badge, orange ultra-critical treatment, yellow/amber warning/status treatment, per-cell status badges, alarm text, acknowledged/active visual states, and consistency with OC alarm state.

- [ ] Add recordable program overlay system.
  - Decision: v1 needs toggleable overlays for SDI/RP188 timecode, roll/clip, and slate/client-facing metadata on the Live Stitch output.
  - Decision: TC and roll/clip are the crucial fields; all overlays must be removable/toggleable for clean stock recordings.
  - Decision: use the lower letterbox/negative space as an information band when available instead of covering the stitched image.
  - Decision: provide overlay presets (`Clean`, `Production Review`, `Minimal`, `Custom`) plus drag/reorder placement.
  - Decision: production-client mode records Live Stitch overlays by default; stock/clean workflows can turn recorded overlays off.
  - Decision: overlays can be enabled/disabled separately for live display and recorded output.
  - Decision: TC overlay source priority is RP188, then visual overlay/OCR/cropped TC, then confirmed last-resort GPS/system time-of-day.
  - Decision: TC numbers turn red while recording; Live Stitch red recording border is live-only and not recorded.
  - Decision: visual TC fallback prefers clean rendered text when reliable; crop/enlarge remains fallback for OCR/refresh limitations.
  - Decision: TC overlay format is standard `HH:MM:SS:FF`.
  - Decision: roll/clip overlay defaults to friendly `Roll 005  Clip 006`, with compact option; all-camera ID display is future/optional.
  - Decision: overlay style is clean, legible, modern broadcast style.
  - Needed: OC slate fields, overlay source mapping, TC fallback selector/confirmation, placement editor with predetermined grid, lower-band layout preset, per-output toggles, persistence per job/mode, and renderer/recorder integration.

- [ ] Add fast slate controls to Operator Console.
  - Decision: director, DP, represented company, and project are sticky job-level fields; scene/shot/take are quick-edit fields for between takes.
  - Decision: snapshot slate at REC start; allow retroactive edit of the most recent take after cut with logged sidecar/manifest/report updates.
  - Decision: slate take auto-increments after Working Mode takes, not setup recordings; false rolls ask keep/increment/revert.
  - Needed: compact OC controls, keyboard-friendly editing, overlay binding, manifest/report slate binding, per-take snapshot timing, auto-increment behavior, and edit history.

- [ ] Add Verify Clip Sync / Check All Settings view.
  - Decision: operator can manually catch up behind cameras, then run a readable verification view to confirm roll/clip consensus and current overlay/TC state.
  - Decision: available in Setup Mode and Working Mode; disabled/read-only during active recording.
  - Needed: full-screen or large-modal status report, per-camera roll/clip/REC/sync/genlock/settings/media/power/TC fields, pass/fail summary, and correction event logging.

- [ ] Add temporary 9-grid camera punch-up.
  - Decision: right-click or equivalent on a 9-grid cell temporarily replaces the grid with that camera for focus/settings inspection, exits on next click/key stroke, and is disabled during recording.
  - Needed: input gesture, full-screen/punch-up layout, exit behavior, and recording-state guard.

- [ ] Add analyzer test corpus and harness.
  - Decision: REC detection and roll/clip OCR need saved-frame tests covering expected RED overlay states, roll/clip formats, high roll numbers, camera/position letters, and output profiles.
  - Needed: reference frame corpus, expected-label fixtures, automated test runner, OC/manual diagnostic entry point, and regression checks in build workflow.

- [ ] Add deeper System Calibration mode.
  - Decision: separate from per-job verification, System Calibration is a fallback/troubleshooting process that walks through broader settings/status changes while MLS analyzes transitions in real time.
  - Needed: guided prompts, captured calibration evidence, ROI/profile updates, analyzer confidence report, and clear distinction from daily/job setup verification.

- [ ] Expand visual analyzer fields.
  - Current state: color classification covers T/E SYNC, GEN TC, CF, DC-IN.
  - Needed: record bug, reel/clip OCR, FPS, shutter, ISO, color temperature, resolution, codec, CF remaining. TC numeral color is optional/non-gating for MVP record state.

- [ ] Implement settings consistency checks.
  - Needed: compare expected setup values with analyzer/RCP2 observations and emit `FPS_MISMATCH`, `SETTINGS_MISMATCH`, `ISO_MISMATCH`, `WB_MISMATCH`.

- [ ] Add card capacity checks.
  - Needed: `CARD_LOW` info event from overlay/RCP2 data.

### Data Handoff

- [ ] Fill v1.1 sidecars with real TC, clips, flags, slate, comments, GPS/IMU.
  - Current state: schema exists with placeholders.
  - Needed: write actual `timecodeStart`, `timecodeEnd`, frame counts, per-camera QC flags, `stitchPath`, and sidecar data for each closed take.

- [ ] Finalize project manifest rollup.
  - Current state: rollup exists.
  - Decision: update project-level `manifest.json` after every closed Working Mode take; wrap finalizes/locks it.
  - Needed: wrap/finalize action, crash recovery behavior, and MMM-compatible validation fixture.

- [ ] Generate and package calibration handoff files.
  - Spec: `_CALIBRATION/*.pts` and Mistika `.vrenv`.
  - Current state: `.pts` pipeline exists; `.vrenv` handoff still needs implementation/decision.
  - Decision: MFS automation is on hold; expert manual final stitch is the current downstream pipeline.

- [ ] Keep MLS outputs self-contained without blocking on NMM/MFS.
  - Decision: do not spend current implementation effort on NMM/MFS round-trip automation; focus on MLS folder/data/report correctness.
  - Decision: MFS automation and NMM integration are explicitly dropped/deprioritized for now.
  - Needed: stable local project layout, production take sidecars, report package, calibration files, comments, GPS/IMU exports, and clear media inventory.

## Next

### Stitch Quality

- [ ] Validate and tune the per-frame GPU flow warp path.
  - Current state: `FlowWarpEngine` exists and is wired, but current behavior appears to be a uniform overlap correction rather than validated optical-flow stitch quality.
  - Decision: post-MVP/high-value experiment after core MVP stability; could materially improve live stitch quality if validated.
  - Needed: test against `footage/Roll01_Clip04` and `config/mercy01`, compare before/after composites, and define an objective accept/reject metric.

- [ ] Keep MVP stitch source PTGui-template based.
  - Decision: MVP Live Stitch is real-time GPU playback of a loaded PTGui `.pts` calibration template, with MLS applying remap/blend and optional photometric correction.
  - Decision: MVP does not need live Path B solving; existing PTGui template is the operational source.
  - Needed: clear UI wording, manifest `stitchPath`, and verification that loaded template matches expected array/job.

- [ ] Add clean PTGui still-grab support tool.
  - Decision: one button captures clean overlay-free stills from all camera inputs for fast PTGui template creation.
  - Decision: normal path prompts overlays off, captures, then prompts overlays back on; rushed path can crop/mask overlays and accept lower usable resolution.
  - Decision: save under `[jobname]/_CALIBRATION/stills/YYYYMMDD_HHMMSS/` with camera letter plus position/direction filenames.
  - Decision: prefer TIFF export; PNG fallback if needed.
  - Decision: capture all 9 stills from the same frame/timecode as closely as practical; stationary camera/scene matters more than TIFF export timing.
  - Needed: clean feed capture path, overlay-off/on guided prompts, crop/mask fallback, max practical resolution/color depth TIFF export, synchronized capture timing/tolerance, still-grab manifest, project folder destination, naming convention, and operator feedback.

- [ ] Load default PTGui template at job open.
  - Decision: every job starts with a usable default `.pts` so Live Stitch works immediately; operator can replace it later with a captured/imported job-specific template.
  - Decision: calibration switches are logged and each take records the active PTGui template.
  - Decision: calibration switch controls are disabled during active recording.
  - Decision: MVP uses one global updateable Spheris XL default template; per-array defaults can come later if needed.
  - Phase: Phase 1 core loop.
  - Needed: default template selection/update flow, project `_CALIBRATION/` copy/update behavior, manifest source/current template tracking, per-take active template field, and UI for importing/replacing active template.

- [ ] Decide the Path A/Path B operator model.
  - Spec: path selection is a quality knob and the active path is recorded.
  - Needed: UI state, manifest `stitchPath`, hot-switch event logging, and graceful fallback from Path B to Path A.

- [ ] Investigate PTGui radial distortion application.
  - Current state: distortion parsed but zeroed before shader use.
  - Decision: post-MVP/high-value quality experiment after PTGui parity is validated.
  - Needed: isolated math validation against PTGui output before enabling in live renderer.

### Stream and Client Review

- [ ] Timecode-lock the live stream.
  - Current state: RTMP/HLS streaming exists.
  - Needed: stream time display from primary RP188 TC, not host clock or frame counter.

- [ ] Add client comments and likes.
  - Current state: VPS portal/login exists.
  - Decision: existing client login/comment system should be refined with TC stamping, not rebuilt.
  - Decision: comments use Live Stitch TC source priority and store TC confidence plus server receipt time.
  - Decision: store comments immediately in project log; final take association is recomputed at wrap/finalize.
  - Needed: comment persistence, current stream TC stamping, `tcFrames`, source/confidence, project-level comments/chat log, live optional association, and sidecar association by TC range at wrap.

- [ ] Add blurred standby while not recording.
  - Decision: stream is blurred/obscured with message when cameras are not recording; clear feed while rolling.
  - Needed: stream output mode follows record state without interrupting local recording.

- [ ] Add stream health events.
  - Current state: stream failure event kinds exist.
  - Needed: reconnect behavior and operator-visible state.

- [ ] Consider VPS/live-stream recording for v1.5.
  - Future idea: record full stream sessions server-side for client playback after shoot.
  - MVP scope: local MLS recordings plus comment/chat log are the source of truth.

### GPS / IMU

- [ ] Define `MotionDataSource`.
  - Spec: all consumers read normalized `MotionSample`, not sensor-specific UBX/NMEA.
  - Decision: GPS/IMU is MVP scope but never blocks running/recording; logging can be disabled.
  - Decision: missing hardware is yellow/non-blocking; connected-but-not-logging/no-lock is more serious OC status but still non-blocking.
  - Needed: protocol, sample model, source lifecycle, enable/disable state, and sample-rate metadata.

- [ ] Implement simpleRTK2B Fusion driver.
  - Current state: board observed on macOS as `/dev/cu.usbmodem21201`; NMEA fix output confirmed. UBX NAV/ESF output can be enabled on USB and emits `NAV-ATT`, `NAV-PVT`, `NAV-SAT`, `ESF-MEAS`, `ESF-RAW`, `ESF-STATUS`, and `ESF-INS`.
  - Decision: logging-first implementation. Archive raw NMEA and raw UBX NAV/ESF packets with host timestamps before relying on parsed attitude visualization.
  - Needed: USB serial discovery/config, NMEA parsing, raw UBX archiving, host arrival timestamping, fix quality, satellite/HDOP/no-lock state, GPS route/time/speed/course parsing, and later validated IMU/fusion parsing.

- [ ] Align motion samples to camera TC.
  - Decision: fuse GPS/IMU samples to camera TC using SDI RP188 as the primary source; visual/overlay TC is backup.
  - Needed: host-clock to RP188 mapping from captured frames, overlay-TC fallback path, per-clip slicing, dropout detection, confidence labels, and `GPS_DROPOUT`.

- [ ] Summarize GPS/IMU per take.
  - Decision: each take sidecar contains its own GPS/IMU samples/summaries rather than only referencing a project-level log.
  - Decision: target VFX/post use first; motion-base-grade playback is not an MVP promise.
  - Decision: export per-take VFX-friendly CSV alongside JSON.
  - Decision: raw GPS/IMU data should be take-scoped when kept; avoid full-day raw dumps as the primary workflow.
  - Decision: include 2-5 seconds of GPS/IMU pre-roll/post-roll around each take while marking actual take TC range.
  - Decision: camera report shows per-take GPS/IMU status and summary.
  - Needed: per-take sample slicing, aligned sample serialization, per-take raw packet slices when enabled, pre/post-roll range markers, CSV export, status classification, distance, speed, altitude, fix continuity, motion class, and optional reverse geocoding.

- [ ] Add Operator Console GPS/IMU tab.
  - Decision: OC should visualize route/map data and live IMU orientation/motion on a small vehicle graphic.
  - Decision: route/status visualization comes before smooth attitude/vehicle animation; roll/pitch/yaw display waits for validated ESF/INS parsing and mounting calibration.
  - Needed: logging toggle, connection/status display, no-device/no-lock/not-logging states, live map/route view, per-take route review, and later live vehicle/IMU widget.

## Later

- [ ] Build slate input panel.
  - Needed: sticky scene/shot/take, description, direction, lighting, subject, notes, quality flag, voice-to-text.

- [ ] Generate live-updating camera report.
  - Decision: MVP report includes human-readable PDF plus machine-readable JSON; HTML is not the primary human deliverable.
  - Decision: report style should reference typical camera reports and ZoeLog-like readable production logs.
  - Decision: report data updates after every take; human report can regenerate after every take or on demand.
  - Decision: support end-of-day camera report export, separate from final project wrap.
  - Decision: shoot day boundaries are explicit; MLS can suggest new day on date change but requires confirmation.
  - Decision: report package contains PDF, JSON report data, manifest snapshot, comments/chat log, GPS/IMU CSV summaries, and media inventory CSV.
  - Needed: job/crew header, shoot-day grouping, setup verification summary, take table, broken clips, false rolls, camera/lens changes, anomaly log, GPS/IMU summary, stream comments by TC/take, media inventory, machine-readable JSON, end-of-day PDF export flow, and report package folder.

- [ ] Consider in-app build checklist for v1.5/2.0.
  - Future idea: bring the external build checklist into MLS so setup can capture serial numbers, firmware versions, setting confirmations, and build-complete checks.
  - MVP scope: camera report records setup verification/readiness status only, not full checklist completion.

- [ ] Add wrap/finalize workflow.
  - Needed: lock report, run final validation, produce transfer-ready package for MMM.

- [ ] Add session persistence and restart recovery.
  - Decision: normal startup prominently offers resume existing job for breaks/location moves.
  - Decision: startup hierarchy is Resume Existing Job, Start New Job, then utilities/diagnostics; new/resumed jobs enter Setup Mode.
  - Decision: interrupted sessions use a dedicated Recovery Startup screen with job/mode/take details.
  - Decision: interrupted media stays in correct roll/clip folder when identity is known and is marked `INTERRUPTED_RECORDING` / `NEEDS_REVIEW`.
  - Decision: if only one output can be protected, prioritize 9-grid.
  - Needed: restore project state, shoot day, mode, source mapping, active PTGui template, active setup, ROI/verification status, stream settings, GPS/IMU state, closed-take sidecars, interrupted media recovery, and writer finalization after app restart.

- [ ] Add capture-card disconnect auto-reconnect.
  - Needed: detect SDI/card timeout, record what is possible, surface event, and auto-resume.

- [ ] Add disk-space preflight and operator block before next take.
  - Current state: recording checks minimum free disk before start.
  - Needed: visible meter, persistent warning, and clear "cannot start next take" state for file-init/disk-full cases.

## Blocked / Needs Coordination

- [ ] Confirm jam-sync practice.
  - Spec open item: all 9 Komodos genlocked from one master TC generator at day start, re-jam at lunch.

- [ ] Confirm Komodo SDI RP188 output settings.
  - Spec open item: RP188 embedding enabled on every camera.

- [ ] Coordinate MMM manifest v1.1/jobname support.
  - Spec open item: jobname field, folder export, `qc_fail/`, `false_roll/`, and MLS manifest ingest.

- [ ] Define MLS -> MMM -> MFS round-trip test.
  - Needed: manifest produced by MLS, ingested by MMM, consumed by MFS, with stable IDs, filenames, and TC math verified.

- [ ] Decide `.vrenv` generation scope.
  - Needed: whether MLS generates a real Mistika template in MVP or packages a reference stub until MFS integration.

## Done / Foundation

- [x] Add easy local build/run path.
  - `dev.sh` builds, bundles calibration tooling, and opens the app unless `SPHERIS_NO_OPEN=1`.

- [x] Add project-local context docs.
  - `AGENTS.md` and the MVP PDF are tracked.

- [x] Add frame spine for capture metadata.
  - `CapturedFrame` now carries pixels, slot, source identity, timecode placeholder, metadata quality, signal state, and host timestamp.

- [x] Add manifest v1.1 shapes.
  - Project, take, clip, QC flag, slate, GPS/IMU, and client comment models exist.

- [x] Add initial event sink.
  - Event/QCFlag taxonomy exists and feeds sidecar capture.

- [x] Add initial visual analyzer.
  - HSV color classifier handles several slow-changing overlay fields with default ROIs.

- [x] Add initial recording coordinator.
  - Live stitch, 9-grid, and per-camera proxy recorders exist; sidecar and manifest rollup are wired.

- [x] Add live stream plumbing.
  - RTMP/HLS settings, streamer, VPS app, and client login page exist.

## Grill Queue

We should answer these one at a time and update this board as decisions firm up. Each question should either produce a decision, a task, or an explicit "defer until integration test" note.

### Capture and Hardware

1. What counts as a valid MVP production capture source: DeckLink/RP188 only, or any source that can provide frames?
2. Is `AVFoundation` fallback allowed to record a production take, or only preview/debug?
3. If fallback capture records a take, what exact warning should the sidecar/report carry?
4. Do we certify only DeckLink Duo 2 for v1, or any DeckLink device exposing 9 synchronized inputs?
5. What is the expected behavior if one SDI input disappears mid-take?
6. Should MLS keep recording stitch/grid when one camera signal is lost, or only log the failure and keep files open?
7. Should capture-card reconnect be automatic during a take, or only between takes?
8. Do we need a pre-shoot hardware self-test screen before the operator reaches the main UI?
9. How should the app represent "signal present but wrong camera" versus "no signal"?
10. Is capture-card physical SDI order fixed enough to encode, or must the operator verify it each job?

### Timecode and Sync

1. Should TC drift compare every camera to Position 1, or to array median with Position 1 used only for proxy stamping?
2. If Position 1 loses signal, which camera becomes primary TC source?
3. Is primary TC source configurable per job, or an emergency-only setting?
4. What is the exact soft drift threshold: `>1 frame` as spec says, or operator-configurable from the start?
5. What is the exact critical sync threshold: `>5 frames`, or the current UI tolerance value?
6. Should the app distinguish RP188 drift from visual `T/E SYNC` red in the report?
7. What happens when RP188 is present but some cameras have missing or invalid TC?
8. How should Time of Day crossing midnight be represented in sidecars?
9. Is drop-frame TC needed for 29.97, or can MVP assume integer 24/25/30?
10. Do proxies need per-frame TC stamping, or is a correct starting `tmcd` plus frame rate acceptable for MVP?
11. How often should proxy embedded TC verification run after startup?
12. Should a proxy TC mismatch be warning only, or should it force `qc_fail/` because downstream comments and sync are compromised?

### Recording and Take Identity

1. What exact combination of visual signals starts recording: red bug only, red TC numerals only, or both?
2. How many consecutive analyzer samples are required before declaring a record-state transition?
3. Does recording start when all 9 cameras roll, or after majority consensus with a short wait for lagging cameras? Decision: 500 ms consensus window.
4. If 8/9 cameras roll inside the 500 ms window and the ninth does not, is that critical or warning?
5. If one camera's OCR is unreadable but record bug is red, can the clip still be considered valid?
6. Should interim `take_NNN` folders ever survive in production output, or must unresolved identity force operator intervention?
7. What should happen when OCR consensus disagrees with the previous expected roll/clip increment?
8. Should roll/clip values be taken from numeric consensus only, or should Camera ID/Position letters also gate validity?
9. Is `CAMERA_RECORDED_ALONE` detected only between takes, or also if one camera false-rolls while others stay idle?
10. What is the minimum take duration before false-roll heuristic is even considered?
11. Who confirms heuristic false rolls: operator during shoot, or technician at wrap?
12. Should manual record be available in MVP, and if so how is it marked differently from analyzer-triggered recording?

### Visual Analyzer and ROI Calibration

1. Do we block starting a take if ROI calibration is missing, or warn and allow manual override? Decision: require startup overlay verification for every job setup; force roll/cut for REC plus roll/clip, passively verify stable fields.
2. Is ROI calibration per project, per camera body, per SDI input, or per resolution/profile?
3. Should ROI calibration happen once for all fields via template offsets, or field-by-field rectangles?
4. How does the operator verify ROIs are correct before the first take?
5. What is the fallback if a field ROI is occluded or unreadable?
6. Which analyzer fields are MVP-critical on day one: record bug, TC color, reel/clip, sync colors, settings, card, power?
7. Should settings checks come from OCR, RCP2, or whichever source is available with confidence labels?
8. How much disagreement between OCR samples is tolerated before suppressing an event?
9. Do we need per-camera analyzer confidence in the sidecar/report?
10. Should color centroids be globally tuned, or saved per project after calibration?
11. Can the overlay layout vary by RED firmware or output mode, and do we need a firmware/profile selector?
12. Does MLS need to preserve the full QC feed with overlays in all recordings, or only the 9-grid proxy?

### QC and Severity

1. Are the only `CRITICAL` classes for MVP exactly `CAMERA_FAILED_TO_RECORD`, `CAMERA_RECORDED_ALONE`, and `SYNC_LOST`?
2. Is `CARD_MISSING` truly warning only even if detected before the first take?
3. Should disk-full/file-init failures be represented as `Blocking`, even though the spec anomaly table is mostly recording anomalies?
4. Should `GENLOCK_LOST` ever become critical if paired with TC drift?
5. Is `POWER_LOST` warning only, or critical if it causes sync loss?
6. Should warnings move a take to `takes/` with flags, or to a separate `qc_warn/` folder?
7. How does the operator clear or acknowledge an active warning?
8. Are recoveries logged as events, or only failures?
9. Should the report show transient warnings that recovered before recording started?
10. Do anomaly flags belong on the take only, per camera clip only, or both?
11. Should `FILE_WRITE_FAILED` be part of the same QC flag taxonomy or a separate system error section?
12. How long should alarm overlays persist if the event has no explicit recovery?

### Stitch Paths and Calibration

1. What exactly is Path A for MVP: current remap blend, remap blend plus flow warp, or operator-selectable fast variants?
2. Should flow warp ship enabled by default once validated, or remain a visible toggle through the first shoots?
3. What visual metric decides whether flow warp is "better" on real footage?
4. Should flow warp corrections be recorded into the proxy, the stream, both, or optional per output?
5. Does Path B Option 1 mean loading a PTGui `.pts` only, or invoking PTGui/Mistika-style precision output?
6. Is Path B Auto-build from live frames truly in v1, or can `tools/calibrate.py` from extracted stills satisfy the MVP?
7. Is RED stills precision stitch an operator workflow inside MLS, or a handoff workflow to PTGui/Mistika?
8. Is `.vrenv` generation a true MVP deliverable, or an MFS integration artifact that can be stubbed until round-trip testing?
9. Should radial distortion remain disabled until exact PTGui parity, or do we accept small edge artifacts if the center improves?
10. Do per-camera exposure/vignetting corrections need UI toggles in MVP?
11. Should calibration templates be copied into each project folder at job start, at first recording, or at wrap?
12. What is the fallback if Path B generation fails mid-shoot?

### Job Setup and Array Configuration

1. Which setup fields are hard-blocking before the main UI opens?
2. Should body serial and lens serial be required in early internal MVP, or allowed blank with report warnings?
3. Are Camera ID letters and Camera Position letters operator-entered every shoot, or usually loaded from saved array config?
4. Should operators be allowed to use `I` and `O`, or should MLS discourage them despite OCR supporting them?
5. How does MLS detect a stale saved setup?
6. Is jobname uniqueness a warning only, as spec says, or should duplicate folders require explicit resume/new choice?
7. Should mid-project rename be supported before first MVP shoot?
8. Do existing take files keep old jobname after rename exactly as spec says?
9. How should camera/lens swap UI behave during active recording?
10. Should camera/lens swaps be effective immediately, next take only, or user-selectable by TC?
11. Does setup validation against live SDI inputs block proceeding, or allow "accept mismatch" with `LETTER_MISMATCH`?
12. Do we need a dedicated array library for multiple future Spheris XL rigs now, or just `arrayID` text?

### File Layout, Manifest, and MMM Handoff

1. Should per-take sidecars use the same full `ProjectManifest` schema with one take, or a smaller `TakeSidecar` wrapper?
2. Does `manifest.json` roll up after every take, only at wrap, or both? Decision: after every closed Working Mode take; wrap finalizes/locks.
3. Should MMM be expected to recover from sidecars if `manifest.json` is absent, or should MLS always keep a rolling manifest current?
4. Are `fileURL` paths relative to project root including `takes/`, `qc_fail/`, etc.?
5. Should sidecar filenames be `[jobname]_[roll]_[clip]_data.json` exactly, even for `false_roll/`?
6. What should happen if sorting from temp to final folder fails?
7. Should `includesQCHolds` mean `qc_fail` exists, warning flags exist, or something MMM-specific?
8. Are stable IDs based only on numeric roll/clip, or should jobname be included to avoid cross-project collision?
9. How should client comments be inserted into already-written sidecars at wrap?
10. Should MLS write a schema version test fixture for MMM before implementation proceeds?
11. What is the minimum round-trip acceptance test with MMM/MFS?
12. Who owns resolving the MMM v1.1 open questions?

### Live Stream and Client Comments

1. Is the streamed output always the active stitch path, or can it be lower resolution/Path A while recording Path B?
2. Should the 9-grid also be streamable to clients, or operator-only?
3. Does "blurred standby" mean blur live stitch, show branded slate, or freeze last good frame?
4. Should stream start automatically with app launch, job start, recording start, or operator action?
5. Are comments tied to stream wall-clock, displayed TC, or server receipt time?
6. How do client comments handle stream latency relative to camera TC?
7. Should likes/comments be stored on the VPS first and pulled into MLS at wrap, or posted back to MLS live?
8. What is the login model for 5-10 viewers: shared password, email allowlist, or per-user sessions?
9. Does the stream need health monitoring inside MLS, on the VPS, or both?
10. What is the behavior if stream fails during a take?
11. Should stream failures appear in sidecars, camera report, or only event log?
12. Is playback after shoot in MVP hosted by the same VPS, or a local handoff feature?

### GPS / IMU

1. Is GPS/IMU required for the very first internal MVP shoot, or required before external MVP signoff?
2. Does MLS need to configure the simpleRTK2B Fusion, or only read an already-configured device?
3. Which protocol do we prefer first: UBX, NMEA, or both?
4. How should USB device selection work if multiple serial devices are connected?
5. What is the minimum sample schema needed for MMM/MFS versus nice-to-have report summaries?
6. How exact does host-clock to RP188 mapping need to be for MVP?
7. What happens if GPS/IMU is missing at job start?
8. What happens if GPS drops out during a take but dead reckoning continues?
9. Is `GPS_DROPOUT` based on no samples, low fix type, satellite count, HDOP, or all of these?
10. Should raw samples be written into every sidecar, or into separate files referenced by sidecar?
11. Do we need reverse geocoding in MVP, or can summaries omit place names?
12. How is the IMU mounting orientation calibrated and recorded?

### Slate and Camera Report

1. What slate fields are truly required before recording starts?
2. Should slate values auto-copy from previous take by default?
3. Does take number in slate auto-increment independently of camera clip number?
4. How does retroactive edit select the previous clip safely?
5. Does `qualityFlag=false_roll` immediately move files, or mark pending until confirm?
6. Is voice-to-text required for MVP or a convenience pass?
7. What is the minimum camera report format for MVP signoff: HTML only, PDF only, or both?
8. Should the report update after every take even before wrap?
9. What does "finalize locks the report" mean technically: read-only files, manifest flag, UI state, or all three?
10. Who is the report audience for the first internal shoot: Drew/producer, VFX, technician, or all?
11. Should the report include thumbnails/map renders in MVP?
12. Should MMM post-QC results ever flow back into the MLS report?

### Reliability and Test Strategy

1. What is the minimum "full-day uptime" test length before MVP signoff?
2. Can we accept an e2e test using recorded footage before testing the physical array?
3. Which tests require the actual 9-camera array, and which can use file playback?
4. Should the app include a simulation mode for record bug/OCR/sync events?
5. What is the minimum automated test suite before each shoot build?
6. Should `dev.sh` become the blessed way to build/run, or do we also need a packaged installer flow?
7. How should build artifacts be versioned in reports/manifests?
8. What logs should an operator be able to export after a failure?
9. What memory/CPU/GPU metrics should we monitor during long runs?
10. Should the app block starting a new take after a file-write failure until operator acknowledgement?
11. How much behavior must be deterministic across restart?
12. What is the smallest reliable e2e acceptance script for "no missed takes"?
