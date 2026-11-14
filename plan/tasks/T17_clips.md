# T17: Evidence clip cutter

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M50 | P0 | Haiku | 45 min | T03 |

- **Owns:** `cattle/io/clips.py`.
- **Read:** the `cut_clip` block of `02_contracts.md` §10; the public API of `cattle/io/avi.py` (`decode_window`, `index`).
- **Fixtures:** `night_head.avi`.

## Goal
Many outputs need short, playable clips:
- labelling (H06);
- agent evidence (T20);
- risk cards (T24);
- live alerts (T26).

Cut a frame range from a GHEC AVI into a browser-playable H.264 MP4, optionally with boxes and labels drawn on it.

## Specification
- **`cut_clip(path, start_frame, end_frame, out_path, max_width=1280, boxes=None) -> Path`.**
  - Frames run from `start_frame` to `end_frame` **inclusive**. Raise `ValueError` if `end < start`.
  - Decode with `avi.decode_window(path, start, end, max_width=max_width)`.
  - `boxes` is a dict mapping frame_idx to a list of `(x1, y1, x2, y2, label)` in full-res px. Draw boxes with PIL, scaled to the output size.
  - Encode by piping raw RGB frames to `ffmpeg -f rawvideo -pix_fmt rgb24 -s WxH -r 30 -i pipe:0 -c:v libx264 -pix_fmt yuv420p -preset veryfast -crf 26 -movflags +faststart`.
  - Return `out_path`. Create parent folders.
- **`clip_for(cow_id, t_start, t_end, ...)`** is not required. Higher-level selection lives in T24 and T26.

## Locked test
```python
# LOCKED TEST: tests/unit/test_clips.py
import json
import subprocess

import pytest

from cattle.io.clips import cut_clip

NIGHT = "tests/fixtures_private/night_head.avi"


def probe(path):
    r = subprocess.run(["ffprobe", "-v", "error", "-select_streams", "v:0", "-show_entries",
                        "stream=codec_name,width,height:format=duration", "-of", "json", str(path)],
                       capture_output=True, text=True)
    return json.loads(r.stdout)


def test_cut_clip_h264_duration_and_size(tmp_path):
    out = cut_clip(NIGHT, 30, 119, tmp_path / "c.mp4", max_width=640)
    info = probe(out)
    s = info["streams"][0]
    assert s["codec_name"] == "h264" and (s["width"], s["height"]) == (640, 480)
    assert abs(float(info["format"]["duration"]) - 3.0) < 0.2


def test_boxes_change_the_video(tmp_path):
    a = cut_clip(NIGHT, 30, 59, tmp_path / "a.mp4", max_width=320)
    b = cut_clip(NIGHT, 30, 59, tmp_path / "b.mp4", max_width=320,
                 boxes={f: [(1000, 1000, 2000, 2000, "C07")] for f in range(30, 60)})
    assert a.read_bytes() != b.read_bytes()


def test_bad_range(tmp_path):
    with pytest.raises(ValueError):
        cut_clip(NIGHT, 50, 40, tmp_path / "x.mp4")
```

## Done when
`python scripts/verify_task.py T17` passes.
