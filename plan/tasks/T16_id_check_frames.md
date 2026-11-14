# T16: ID-check frames (288 images)

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M40 | P0 | Haiku | 60 min | T03, T05 |

- **Owns:** `cattle/tools/idcheck.py`, `scripts/id_check_frames.py`.
- **Read:** `02_contracts.md` §4, §5, §8.11; the public API of `cattle/io/avi.py`.
- **Fixtures:** none (synthetic provider).

## Goal
The brief requires "ID-check frames showing your cow IDs every 30 minutes on each camera". That is 2 days × 3 cameras × 48 slots = 288 images. Each slot shows the nearest keyframe with each box labelled by cow ID and confidence. A slot with no footage gets a clear placeholder. These images are how judges verify identity, so they must be honest: `UNK` boxes are drawn in a different colour.

## Specification
- **`make_id_check(boxes, provider, out_dir, max_offset_s=60, days=(1, 2), cams=CAMS) -> DataFrame`.**
  - **Slots:** 00:00, 00:30, …, 23:30 on each day (48 per day).
  - **`provider(cam, ts)`** returns `None` or a dict with:
    - `file_id`, `frame_idx`, `frame_ts`;
    - `image` (RGB uint8);
    - `scale` (image px ÷ full-res px);
    - `crop` (`(x, y)` offset in full-res px; `(0, 0)` if uncropped).

    It must return the nearest keyframe within `max_offset_s`.
  - **`boxes`** has columns `cam, ts, x1, y1, x2, y2` (full-res), `cow_id`, `confidence`. Use the rows whose `ts` equals the provider's `frame_ts` exactly, matched to the millisecond.
  - **Drawing:**
    - Convert each box to image coordinates: `(x - crop_x) * scale`.
    - Draw the box and a label `C04 0.80` (PIL ImageDraw, default font).
    - Colour: green for identified cows, red for `UNK`.
    - Header strip: camera, slot, frame time, and `n identified / n boxes`.
  - **No footage:** write a grey 800×600 placeholder that says `no footage within ±60 s`, with `n_boxes = 0`.
  - **Output:** images at `out_dir/day<d>/<cam>/<HHMM>.jpg`. The function returns the index (contracts §8.11) and also writes `out_dir/index.csv`.
- **`scripts/id_check_frames.py --work DIR --out outputs/id_check`.**
  - Builds the provider from `keyframes.parquet` (nearest keyframe per slot) and boxes from `tracks` joined with `gtracks` and `identity`.
  - Prints the counts of slots with footage, boxes and UNK.

## Locked test
```python
# LOCKED TEST: tests/unit/test_idcheck.py
import numpy as np
import pandas as pd
from PIL import Image

from cattle.tools.idcheck import make_id_check
from cattle.validate.schemas import validate


def provider(cam, ts):
    if cam == "cam01" and ts == pd.Timestamp("2000-01-10 10:00"):
        return dict(file_id="F", frame_idx=0, frame_ts=ts, image=np.zeros((300, 400, 3), np.uint8), scale=0.1, crop=(0, 0))
    return None


def test_288_slots_placeholders_and_overlays(tmp_path):
    boxes = pd.DataFrame([dict(cam="cam01", ts=pd.Timestamp("2000-01-10 10:00"), x1=100.0, y1=100.0, x2=2000.0,
                               y2=1500.0, cow_id="C04", confidence=0.8)])
    idx = make_id_check(boxes, provider, tmp_path, max_offset_s=60)
    assert len(idx) == 288 and validate("id_check_index", idx) == []
    row = idx[(idx.cam == "cam01") & (idx.day == 1) & (idx.slot == "10:00")].iloc[0]
    assert row.n_boxes == 1 and row.n_identified == 1
    assert np.asarray(Image.open(tmp_path / row.image)).max() > 0
    empty = idx[(idx.cam == "cam02") & (idx.day == 1) & (idx.slot == "10:00")].iloc[0]
    assert empty.n_boxes == 0 and (tmp_path / empty.image).exists()
    assert (tmp_path / "index.csv").exists()
```

## Done when
`python scripts/verify_task.py T16` passes.
