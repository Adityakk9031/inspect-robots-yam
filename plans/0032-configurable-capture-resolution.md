# Configurable camera capture resolution

**Goal:** let a rig request a native capture size other than 640 × 480 from
its cameras, so policies that need every pixel (AprilTag detection of 20 mm
tags from a 1.2 m top camera, for the `inspect-robots-jev` plugin, see
inspect-robots plan 0076) can get full-resolution frames. Defaults are
unchanged, so existing rigs behave exactly as before.

**Problem:** both camera paths captured at a hard-coded 640 × 480 and then
resized to `cam_width × cam_height`. Raising `cam_width`/`cam_height` only
upsampled: a 20 mm tag at 1.2 m is about 10 px wide at 640, below any
detector's floor. `_capture_proc.REALSENSE_CAPTURE_WIDTH/HEIGHT`,
`_RealsenseCameraReader`'s `enable_stream` calls, the intrinsics scaling in
both RealSense readers, and the OpenCV reader's `CAP_PROP_FRAME_WIDTH/HEIGHT`
all used the constants.

**Design (implemented in this PR):**

- `YamConfig.capture_width: int = 640`, `capture_height: int = 480`,
  validated as integers ≥ 16 (bool rejected) in `__post_init__`.
- `_CaptureProcess(serials, depth_fps, *, capture_size=(w, h))` allocates each
  shared-memory frame slot at that size; the child already enables both
  RealSense streams at the slot's size, so no child change was needed.
- `_RealsenseCameraReader(..., capture_size=...)` (inline path) enables both
  streams at the configured size.
- Intrinsics scaling in both readers uses `cfg.cam_width / cfg.capture_width`
  and `cfg.cam_height / cfg.capture_height` instead of the constants.
- `_OpenCVCameraReader(devices, capture_size=...)` negotiates the size with
  V4L2; `_opencv_camera_reader(cfg)` passes the configured size.
- `YamConfig.depth_capture_width/height` (optional, both or neither) give the
  RealSense depth stream its own size; `_CaptureSpec.depth_size` carries it to
  the child, and the inline reader honours it too.
- The constants remain as the defaults of every new parameter.

**Not changed:** the resize to `cam_width × cam_height` (set both to the
capture size for a no-downscale pipeline); `depth_fps` semantics; device
capability checks (an unsupported size fails at pipeline start with the
librealsense error, as an unsupported `depth_fps` already does).

**Tests:** config validation and defaults (`test_config.py`); inline streams
at the configured size and intrinsics scaled from it (`test_depth_reader.py`);
process slots allocated at the configured size (`test_capture_proc.py`); V4L2
`CAP_PROP_FRAME_WIDTH/HEIGHT` negotiated from the configured size
(`test_camera_reader.py`). Coverage stays at 100 %.

**Recipe (rig 6, Jev runs):**

```ini
[embodiment.args]
top_depth_serial = <D435 serial>
capture_width = 1920            # colour stream
capture_height = 1080
depth_capture_width = 1280      # D435 depth tops out at 1280 x 720; aligned to colour
depth_capture_height = 720
cam_width = 1920
cam_height = 1080
```

`depth_capture_width/height` (both or neither; default: same as the colour
capture) set the depth stream separately, because a D435 cannot stream depth
at 1920 × 1080 while its colour stream can. Depth is aligned to the colour
frame in the reader, so the published depth array is always colour-sized.
If a combination is rejected at pipeline start, fall back to 1280 × 720 for
both (about 18 px per 20 mm tag at 1.2 m, marginal) or lower the camera.

**Also changed after review:** the `yam-health` / `--watch` V4L2 probes use
the configured capture size (previously fixed at 640 × 480), so the health
check exercises the same stream mode as a run.
