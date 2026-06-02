# Nightshift FreeMoCap to SlimeVR Bridge Plan

## Goal

Use FreeMoCap as the webcam markerless mocap foundation, then add a small bridge
that turns FreeMoCap skeleton output into tracker data SlimeVR can consume. The
first target is software-only webcam tracking. Sensor fusion comes later, after
IMU hardware is available.

## Current Repo Read

- FreeMoCap already covers camera capture, synchronization, calibration, image
  tracking, 3D triangulation, data export, and Blender export.
- Processed skeleton output is available as `output_data/mediapipe_skeleton_3d.npy`
  for the default MediaPipe tracker.
- Split CSV outputs include `mediapipe_body_3d_xyz.csv`,
  `mediapipe_right_hand_3d_xyz.csv`, `mediapipe_left_hand_3d_xyz.csv`, and
  `mediapipe_face_3d_xyz.csv`.
- Blender export is already routed through the bundled FreeMoCap Blender addon.
- No native SlimeVR, VMC, OSC, SteamVR, or VRChat output path is present in this
  fork yet.

## Architecture

### Phase 1: FreeMoCap Baseline

Prove the unmodified fork works locally before adding bridge code.

Deliverables:

- Source install in a local Python 3.11 or 3.12 environment.
- GUI launches from `python -m freemocap`.
- Sample recording can be processed.
- Output folder contains:
  - `output_data/mediapipe_skeleton_3d.npy`
  - `output_data/mediapipe_body_3d_xyz.csv`
  - optional `.blend` export
- Document the local Blender executable path used by FreeMoCap detection.

Success condition:

- A known recording produces a visible skeleton in Blender and readable 3D joint
  data from Python.

### Phase 2: Offline FreeMoCap to VMC Prototype

Build a small script that reads a processed FreeMoCap recording and replays it as
VMC/OSC tracker data to SlimeVR.

Proposed module:

- `tools/freemocap_to_slimevr_vmc.py`

Responsibilities:

- Load `RecordingInfoModel(recording_path).data_3d_npy_file_path`.
- Read frame rate or infer timing from synchronized videos.
- Map FreeMoCap/MediaPipe joints to SlimeVR body roles.
- Estimate body-part positions from landmarks.
- Estimate bone rotations from parent-child joint vectors.
- Send VMC OSC messages to SlimeVR.

Initial mapped body parts:

- Head: nose, ears, shoulders midpoint, or head landmark cluster.
- Chest / upper chest: shoulder midpoint plus torso direction.
- Hip: left/right hip midpoint.
- Left/right knee: knee landmarks.
- Left/right foot: ankle/heel/foot-index landmarks.
- Left/right elbow: elbow landmarks.
- Left/right hand: wrist landmarks.

OSC strategy:

- Prefer VMC tracker messages for position plus rotation during prototype.
- Allow manual assignment in SlimeVR if trackers arrive unassigned.
- Keep message names stable so assignments persist between runs.

Success condition:

- SlimeVR shows FreeMoCap-derived trackers during replay.
- Assigned trackers can drive the SlimeVR skeleton without any IMU sensors.
- Timing and coordinate axes are understandable and documented.

### Phase 3: Blender Preview Addon

Only after Phase 2 works, add Blender workflow helpers. Blender should be the
preview, cleanup, and recording surface, not the first transport layer.

Possible addon features:

- Pick a FreeMoCap recording folder.
- Preview SlimeVR-mapped trackers over the FreeMoCap skeleton.
- Start/stop VMC replay from Blender.
- Show confidence/reprojection-error overlays later.
- Export a mapping preset.

Success condition:

- Blender can preview the same tracker mapping that SlimeVR receives.
- Artists can diagnose bad joints, bad axes, and bad calibration without reading
  raw NumPy arrays.

### Phase 4: Live Webcam Bridge

After offline replay works, adapt the bridge to real-time or near-real-time data.

Likely paths:

- Reuse FreeMoCap/skellytracker live camera capture if available enough.
- Or create a dedicated lightweight MediaPipe/skellytracker live loop outside
  the full recording pipeline.
- Keep output protocol identical to Phase 2 so SlimeVR integration does not
  change.

Key risks:

- Multi-camera real-time triangulation may be slower or less stable than offline
  processing.
- Camera synchronization matters more in live mode.
- Occlusion and missing landmarks need smoothing and fallback rules.

Success condition:

- One or more webcams update SlimeVR trackers live at a usable rate.
- Dropped landmarks do not teleport the skeleton.
- Latency is measured, not guessed.

### Phase 5: IMU Sensor Fusion

Do this after the new sensors arrive. SlimeVR currently selects a tracker per
body position; true fusion needs a deliberate policy.

Possible fusion models:

1. Camera-only tracker
   - Position and rotation come from FreeMoCap.
   - Useful before sensors exist.

2. IMU-priority rotation, camera-priority position
   - IMUs provide stable low-latency orientation.
   - FreeMoCap provides positional correction and drift anchors.
   - Requires either a SlimeVR server change or an external fused-tracker bridge.

3. Camera calibration / correction only
   - Use FreeMoCap to estimate body scale, pose quality, foot placement, or drift
     correction events.
   - Lower integration risk than full live fusion.

Recommended first sensor fusion target:

- External fused-tracker bridge that reads camera positions and SlimeVR tracker
  rotations, then emits a new fused set of VMC trackers. This avoids changing the
  SlimeVR server until the behavior is proven.

Success condition:

- With sensors connected, fused output is better than either camera-only or
  IMU-only tracking in at least one measured case: foot position, hip drift, body
  scale, or long-session yaw/pose stability.

## Dependencies To Add Later

For the first bridge prototype:

- `python-osc` for VMC/OSC packet output.
- `numpy` already exists through FreeMoCap dependencies.
- Optional: `scipy` for rotation math if existing dependencies are insufficient.

For live mode:

- Reuse existing FreeMoCap/skellytracker packages where possible.
- Avoid adding heavy realtime dependencies until the offline bridge proves the
  coordinate mapping.

## Files Worth Knowing

- `freemocap/system/paths_and_filenames/file_and_folder_names.py`
  - Output file names and folder names.
- `freemocap/data_layer/recording_models/recording_info_model.py`
  - Canonical path helper for processed recordings.
- `freemocap/core_processes/process_motion_capture_videos/process_recording_headless.py`
  - Headless processing entry point.
- `freemocap/core_processes/capture_volume_calibration/triangulate_3d_data.py`
  - Multi-camera 3D reconstruction.
- `freemocap/core_processes/export_data/blender_stuff/export_to_blender/export_to_blender.py`
  - Blender export path.

## Open Questions

- Which SlimeVR input path should become the long-term target: VMC/OSC,
  WebSocket API, native UDP tracker protocol, or a small SlimeVR bridge plugin?
- Should camera-derived trackers be manually assigned in SlimeVR, auto-assigned
  by stable VMC names, or mapped through a new config file?
- How much smoothing should happen before SlimeVR versus inside SlimeVR?
- What is the minimum acceptable live update rate for the intended use?
- Is the goal VRChat-style live FBT, Blender capture, or both?

## Recommended Next Step

Do not start with live fusion. Start with an offline replay bridge from a known
FreeMoCap recording into SlimeVR. That gives fast feedback on coordinate axes,
scale, body-part mapping, and SlimeVR assignment behavior without fighting camera
sync, sensor hardware, or live inference performance at the same time.
