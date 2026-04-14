# Fix: Use pyav video backend in openpi to avoid torchcodec ABI mismatch

## Problem

Training pi0.5 fails in the DataLoader with:
```
RuntimeError: Could not load libtorchcodec.
```

lerobot 0.4.4 defaults to `torchcodec` as the video decoder. The `torchcodec 0.5` wheel resolved from PyPI has a C++ ABI mismatch with `torch 2.7.1+cu126`, and the Ubuntu 22.04 container only ships FFmpeg 4.4 (missing `libavutil.so.57`–`.60` for FFmpeg 5–8).

## Fix

**Repo:** openpi fork (`jtopaz1/openpi`)  
**File:** `src/openpi/training/data_loader.py`, line ~141

Add `video_backend="pyav"` to the `LeRobotDataset()` constructor call:

```python
# Before
dataset = lerobot_dataset.LeRobotDataset(
    data_config.repo_id or "unknown",
    delta_timestamps={
        key: [t / dataset_meta.fps for t in range(action_horizon)] for key in data_config.action_sequence_keys
    },
)

# After
dataset = lerobot_dataset.LeRobotDataset(
    data_config.repo_id or "unknown",
    delta_timestamps={
        key: [t / dataset_meta.fps for t in range(action_horizon)] for key in data_config.action_sequence_keys
    },
    video_backend="pyav",
)
```

## Why this works

- `pyav` is already a dependency of lerobot and is installed in the container.
- FFmpeg 4.4 from Ubuntu 22.04's `apt-get install ffmpeg` is sufficient for pyav.
- This bypasses torchcodec entirely, avoiding the ABI mismatch.

## After applying

Update the pinned commit in `modal/pi05/app.py` (Dockerfile clone step) to point to the new openpi commit.
