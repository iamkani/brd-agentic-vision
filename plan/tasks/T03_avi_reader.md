# T03: AVI reader (RIFF index, NAL keyframes, clock, decode)

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M10 | P0 | Sonnet | 90 min | T00, T01 (fixtures), T02 (clock) |

- **Owns:** `cattle/io/avi.py`.
- **Read:**
  - `02_contracts.md` §1 (rows Container, Chunk layout, Video, Keyframes, Clock text) and the `cattle/io/avi.py` block of §10;
  - `04_testing.md` §4;
  - `cattle/io/clock.py`.
- **Fixtures:** `tests/fixtures_private/night_head.avi` and `day_head.avi`. Copy them into your worktree.

## Goal
This module is the one and only way the project reads the raw videos. Neither ffmpeg nor OpenCV can open the AVIs directly, because the container labels standard HEVC with the private fourcc `GHEC`. Parse the RIFF container yourself, find keyframes from the HEVC NAL unit types, attach clock timestamps, and decode with ffmpeg reading raw HEVC from a pipe.

## Facts you can rely on
- **Layout:**
  - `RIFF....AVI ` file.
  - The `hdrl` list holds `avih`: microseconds per frame at byte 32, number of streams at 56, width at 64, height at 68 (all little-endian uint32).
  - The video `strh` fccHandler is at byte 112 and the `strf` biCompression at byte 188 (both `GHEC`).
  - The `movi` list type is at byte 2044 and the first chunk at 2048.
  - The `movi` LIST size is at byte 2040, so `movi` ends at `2044 + size`. An `idx1` chunk follows in complete files only.
- **Chunks:** every frame is `00db`|`00dc` (video), then `01db`|`01dc` (NUL-terminated clock text), then `02db`|`02dc` (size 0). Chunks are padded to an even size.
- **Keyframes:** a keyframe payload contains NAL types 32, 33, 34 and then 19, so detect any NAL type in 16–21 within its first 2 KB. Delta frames start with NAL type 1. Do **not** use idx1 flags; truncated files have no idx1.
- **Standalone decoding:** each keyframe payload decodes alone (`ffmpeg -f hevc -i pipe:0`). Delta frames need every frame back to their keyframe.

## Specification
Implement exactly the API in contracts §10:
- **`probe`:** read the header only.
- **`index`:** return a DataFrame with exactly the columns `frame_idx, offset, size, is_key, clock_text`. Include complete video chunks only, and stop at the first incomplete chunk. `clock_text` is the text chunk following that video chunk, or `None`.
  - **Speed:** on complete files, read the 8-byte chunk headers sequentially with `f.seek`, and read the first 2 KB of video payloads only.
- **`read_clock`:** use `to_timestamps` on `index(...).clock_text` and return `pd.Series(index=frame_idx, name="ts")`.
- **`read_payloads(path, frame_indices)`:** yield `(frame_idx, bytes)` with the exact payload bytes.
- **`iter_keyframes(path, max_width=None, crop=None)`:**
  1. Concatenate the keyframe payloads and feed them to one ffmpeg process: `-f hevc -i pipe:0`, then `-vf crop=w:h:x:y,scale=W:-2` if needed, `-pix_fmt rgb24 -f rawvideo pipe:1`, and `-fps_mode passthrough`.
  2. Read the frames back by their exact byte size and yield `Frame(frame_idx, ts, image)`.
  3. The output size is the crop size, or 4000×3000 without a crop, scaled so the width equals `max_width` with even dimensions.
- **`decode_window(path, start, end, step=1, max_width=None, crop=None)`:**
  1. Find the last keyframe ≤ `start` and feed payloads from that keyframe through `end`.
  2. Decode, then return frames `start..end` (inclusive) every `step`.
  3. Raise `ValueError` if `end < start` or the range exceeds the file.
- **Errors:** run ffmpeg with `-v error`, check its exit code, and raise `RuntimeError` with stderr on failure. Use `subprocess` with list arguments, never `shell=True`.

## Locked test
```python
# LOCKED TEST: tests/unit/test_avi.py
import time
from datetime import datetime

import numpy as np
import pandas as pd
import pytest

from cattle.io import avi

NIGHT = "tests/fixtures_private/night_head.avi"
DAY = "tests/fixtures_private/day_head.avi"
NIGHT_KEYS = [0, 19, 79, 139, 199, 259, 319, 379, 439, 499]
DAY_KEYS = [0, 19, 79, 139, 199, 259, 319, 379]


def test_probe_reads_riff_header():
    info = avi.probe(NIGHT)
    assert (info.width, info.height) == (4000, 3000)
    assert info.fourcc == "GHEC"
    assert info.fps == pytest.approx(30.0, abs=0.01)
    assert info.n_streams == 3
    assert info.has_index is False


def test_index_columns_and_first_chunks():
    idx = avi.index(NIGHT)
    assert list(idx.columns) == ["frame_idx", "offset", "size", "is_key", "clock_text"]
    assert (idx.offset[0], idx["size"][0], bool(idx.is_key[0])) == (2056, 79360, True)
    assert (idx.offset[1], idx["size"][1], bool(idx.is_key[1])) == (81468, 13160, False)
    assert idx.clock_text[0] == "Event#1-23:55:50(2000/1/10)"
    assert idx.clock_text[1] == "#1-23:55:50.197"
    assert idx.clock_text[19] == "#1-23:55:50.797"


def test_truncated_file_only_complete_frames():
    idx = avi.index(NIGHT)
    assert len(idx) == 515 and list(idx.frame_idx) == list(range(515))
    assert idx.clock_text[514] == "#1-23:56:7.297"


def test_keyframes_from_nal_types_not_index_flags():
    assert list(avi.index(NIGHT).query("is_key").frame_idx) == NIGHT_KEYS
    day = avi.index(DAY)
    assert len(day) == 439 and list(day.query("is_key").frame_idx) == DAY_KEYS
    assert day["size"][0] == 602040


def test_index_is_fast():
    t0 = time.perf_counter()
    avi.index(NIGHT)
    assert time.perf_counter() - t0 < 0.5


def test_read_clock():
    ts = avi.read_clock(NIGHT)
    assert isinstance(ts, pd.Series) and len(ts) == 515
    assert ts.loc[1] == pd.Timestamp("2000-01-10 23:55:50.197")
    assert ts.loc[19] == pd.Timestamp("2000-01-10 23:55:50.797")
    assert ts.loc[514] == pd.Timestamp("2000-01-10 23:56:07.297")
    assert ts.diff().dt.total_seconds().dropna().between(0.015, 0.060).all()
    assert avi.read_clock(DAY).loc[1] == pd.Timestamp("2000-01-10 09:04:52.017")


def test_read_payloads_exact_bytes():
    got = dict(avi.read_payloads(NIGHT, [0, 19]))
    assert len(got[0]) == 79360 and got[0][:6] == bytes.fromhex("000000014001")
    with open(NIGHT, "rb") as f:
        f.seek(2056)
        assert got[0] == f.read(79360)


def test_iter_keyframes_night_is_grey():
    frames = list(avi.iter_keyframes(NIGHT, max_width=800))
    assert [f.frame_idx for f in frames] == NIGHT_KEYS
    img = frames[1].image
    assert img.shape == (600, 800, 3) and img.dtype == np.uint8
    assert frames[1].ts == datetime(2000, 1, 10, 23, 55, 50, 797000)
    assert np.abs(img[..., 0].astype(int) - img[..., 2].astype(int)).mean() < 1.0


def test_iter_keyframes_crop():
    f = next(iter(avi.iter_keyframes(DAY, max_width=700, crop=(1200, 300, 1400, 900))))
    assert f.image.shape == (450, 700, 3)


def test_decode_window_inclusive():
    frames = avi.decode_window(NIGHT, 20, 25, max_width=400)
    assert [f.frame_idx for f in frames] == [20, 21, 22, 23, 24, 25]
    assert all(f.image.shape == (300, 400, 3) for f in frames)
    assert frames[0].ts == avi.read_clock(NIGHT).loc[20]
    with pytest.raises(ValueError):
        avi.decode_window(NIGHT, 30, 20)
```

## Done when
`python scripts/verify_task.py T03` passes, and `tests/unit/test_clock.py` stays green.

## Do not
- Rely on `ffprobe` packet flags or the AVI index for keyframes.
- Patch or copy the AVI file.
- Treat the demuxer's "Broken pipe" as a failure. You don't need a demuxer anyway: you read payloads yourself.
