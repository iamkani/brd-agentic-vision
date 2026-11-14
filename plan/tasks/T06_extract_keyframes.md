# T06: Keyframe extraction (parallel, resumable)

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M20 | P0 | Sonnet | 75 min | T03, T04, T05 |

- **Owns:** `cattle/extract/keyframes.py`, `scripts/extract_keyframes.py`.
- **Read:**
  - `02_contracts.md` §1 (rows Keyframes and Light), §2, §5, §8.2, and the `cattle/extract/keyframes.py` block of §10;
  - `03_pitfalls.md` rows 7–9;
  - the public functions of `cattle/io/avi.py` and `cattle/quality/light.py`.
- **Fixtures:** `night_head.avi`, `day_head.avi`.

## Goal
Turn every video into its keyframes, one image every 2 s, saved as JPEGs no wider than 1600 px or as camera-ROI crops. Each extraction comes with an index row that carries the exact clock timestamp and the light class. Measured cost: about 7.7 s per file single-threaded, roughly 1 h for all dataset with 4 workers.

## Specification
- **`extract_file(path, file_id, cam, out_root, max_width=1600, crop=None, jpeg_quality=90) -> DataFrame`.**
  - Uses `avi.iter_keyframes(path, max_width, crop)`.
  - Writes `out_root/keyframes/<cam>/<file_id>/<frame_idx:05d>.jpg`.
  - Returns rows in the `keyframes` schema (contracts §8.2):
    - `ts` from the frame;
    - `day` from `ts.date()`;
    - `picture` and `yavg` from `light.classify_rgb(image)`;
    - `crop_box` = `"x,y,w,h"` or `""`;
    - `scale` = output width ÷ (crop width, or 4000 without a crop).
  - **Resume:** also write `out_root/keyframes/<cam>/<file_id>/index.parquet` plus a `params.json` holding max_width, crop and quality. If both exist with the same params, return the stored index without decoding or rewriting anything. Different params mean re-extract.
  - Omit rows whose `ts` falls outside the analysis window in contracts §5 from the returned and published keyframe indices. Apply this rule to cached indices when reusing them.
- **`extract_many(manifest, raw_root, out_root, workers=4, **kw)`.**
  - Runs a process pool over the manifest rows.
  - Before starting, if `out_root` has less than 5 GB free, it raises `RuntimeError`.
  - Logs one line per 50 files (done/total, elapsed time, ETA).
  - Returns the concatenated index and atomically replaces `out_root/keyframes.parquet` with rows for this invocation’s manifest. Filtered runs are provisional. A full unfiltered pass reuses eligible per-file caches and publishes the canonical combined index before full-data detection.
  - A failing file is logged to `out_root/keyframes_errors.csv`; the run continues.
- **`scripts/extract_keyframes.py`.**
  - Flags: `--manifest` (default `<work_root>/manifest.parquet`), `--day`, `--cams cam01,cam02`, `--hours 9-12` (filter by `name_start` hour), `--workers 4`, `--max-width 1600`, `--use-roi` (take each camera's `roi` from `configs/cameras.yaml` as the crop), `--out` (default `work_root`).
  - Prints a 5-line summary at the end.

## Locked test
```python
# LOCKED TEST: tests/unit/test_extract.py
import os

import pandas as pd
from PIL import Image

from cattle.extract.keyframes import extract_file
from cattle.validate.schemas import validate

NIGHT = "tests/fixtures_private/night_head.avi"
DAY = "tests/fixtures_private/day_head.avi"


def test_extract_night(tmp_path):
    df = extract_file(NIGHT, file_id="Event20000110235550006", cam="cam01", out_root=tmp_path, max_width=800)
    assert list(df.frame_idx) == [0, 19, 79, 139, 199, 259, 319, 379, 439, 499]
    assert validate("keyframes", df) == []
    assert df.ts.is_monotonic_increasing and df.ts.iloc[1] == pd.Timestamp("2000-01-10 23:55:50.797")
    assert (df.picture == "grey_IR").all() and (df.day == 1).all()
    assert df.img_path.iloc[1] == "keyframes/cam01/Event20000110235550006/00019.jpg"
    assert Image.open(tmp_path / df.img_path.iloc[0]).size == (800, 600)
    assert abs(df.scale.iloc[0] - 0.2) < 1e-9 and (df.crop_box == "").all()


def test_extract_day_with_crop(tmp_path):
    df = extract_file(DAY, file_id="Event20000110090451006", cam="cam01", out_root=tmp_path, max_width=700,
                      crop=(1200, 300, 1400, 900))
    assert (df.picture == "colour").all()
    assert (df.crop_box == "1200,300,1400,900").all() and abs(df.scale.iloc[0] - 0.5) < 1e-9
    assert Image.open(tmp_path / df.img_path.iloc[0]).size == (700, 450)


def test_resume_does_not_rewrite(tmp_path):
    a = extract_file(NIGHT, file_id="Event20000110235550006", cam="cam01", out_root=tmp_path, max_width=400)
    p = tmp_path / a.img_path.iloc[0]
    before = os.stat(p).st_mtime_ns
    b = extract_file(NIGHT, file_id="Event20000110235550006", cam="cam01", out_root=tmp_path, max_width=400)
    assert os.stat(p).st_mtime_ns == before
    pd.testing.assert_frame_equal(a.reset_index(drop=True), b.reset_index(drop=True))
```

## Done when
`python scripts/verify_task.py T06` passes. The report requests a Makefile target: `extract: python -m scripts.extract_keyframes --use-roi`.

## Do not
- Decode delta frames.
- Write anything outside `out_root`.
- Write full-resolution images: `max_width` must stay ≤ 1600 unless a crop is used.
