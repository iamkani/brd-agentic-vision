# T12: Data-quality report

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M30 | P1 | Haiku | 45 min | T05 |

- **Owns:** `scripts/qa_report.py`.
- **Read:** `02_contracts.md` §8.1–§8.4; `03_pitfalls.md` rows 14, 16, 17.
- **Fixtures:** none (synthetic Parquet files).

## Goal
One command that tells the team, after each full-data run, whether the data and detections look sane. It reports coverage gaps, impossible counts (more than 10 cows), day versus night differences, and staff or vehicles in the pen. Problems should surface before they poison identity and labels.

## Specification
`scripts/qa_report.py --work DIR --out PATH.md`:
- **Inputs:** reads `DIR/keyframes.parquet` and `DIR/detections.parquet`. If present, it also reads `DIR/tracks.parquet` and `DIR/manifest.parquet`.
- **`PATH.md`** contains exactly these sections, as Markdown tables with at most 30 rows each:
  - `## Coverage`: keyframes per camera per hour against the expected 1,800 (one every 2 s). Hours below 90% are marked.
  - `## Detections per frame`: the distribution of cow counts per frame, by camera × picture (colour/grey_IR).
  - `## Frames with more than 10 cows`: the count, plus the first 20 frames listed.
  - `## Day vs night`: mean cows per frame and mean confidence, split by picture.
  - `## Staff and vehicles`: frames with `person`, `horse` or `vehicle` detections, per camera per hour.
- **`qa_flags.csv`**, written next to `PATH.md`, has columns `file_id, cam, frame_idx, ts, n_cows, flag`, where `flag` is one of `over_10`, `zero_cows`, `staff`.
- Exit 0 unless the inputs are missing.

## Locked test
```python
# LOCKED TEST: tests/unit/test_qa_report.py
import subprocess
import sys

import pandas as pd


def test_qa_report_sections_and_flags(tmp_path):
    t0 = pd.Timestamp("2000-01-10 10:00")
    kf = pd.DataFrame([dict(file_id="F", cam="cam01", frame_idx=i * 60, ts=t0 + pd.Timedelta(seconds=2 * i), day=1,
                            picture="colour", yavg=100.0, img_path=f"keyframes/cam01/F/{i * 60:05d}.jpg", crop_box="",
                            scale=0.4) for i in range(30)])
    dets = pd.DataFrame([dict(file_id="F", cam="cam01", frame_idx=i * 60, ts=t0 + pd.Timedelta(seconds=2 * i), det_idx=j,
                              cls="cow", conf=0.9, x1=10.0 * j, y1=0.0, x2=10.0 * j + 5, y2=5.0, model="m")
                         for i in range(30) for j in range(12 if i == 7 else 8)])
    dets = pd.concat([dets, pd.DataFrame([dict(file_id="F", cam="cam01", frame_idx=600, ts=t0 + pd.Timedelta(seconds=20),
                                               det_idx=99, cls="person", conf=0.8, x1=0.0, y1=0.0, x2=5.0, y2=5.0,
                                               model="m")])])
    kf.to_parquet(tmp_path / "keyframes.parquet")
    dets.to_parquet(tmp_path / "detections.parquet")
    r = subprocess.run([sys.executable, "-m", "scripts.qa_report", "--work", str(tmp_path), "--out", str(tmp_path / "qa.md")],
                       capture_output=True, text=True)
    assert r.returncode == 0, r.stderr[-1500:]
    md = (tmp_path / "qa.md").read_text()
    for section in ["## Coverage", "## Detections per frame", "## Frames with more than 10 cows", "## Day vs night",
                    "## Staff and vehicles"]:
        assert section in md, section
    flags = pd.read_csv(tmp_path / "qa_flags.csv")
    over = flags[flags.flag == "over_10"]
    assert list(over.frame_idx) == [420] and list(over.n_cows) == [12]
    assert list(flags[flags.flag == "staff"].frame_idx) == [600]
```

## Done when
`python scripts/verify_task.py T12` passes.
