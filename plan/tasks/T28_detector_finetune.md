# T28: Detector fine-tuning data (auto-label, pick, split) and trainer

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M30 | P1 | Sonnet | 75 min | T06, T07 |

- **Owns:** `cattle/perception/yolo_data.py`, `scripts/sample_frames.py`, `scripts/autolabel.py`, `scripts/train_detector.py`.
- **Read:**
  - `06_prior_work.md` §2 item 2;
  - `02_contracts.md` §8.2, §8.3, §9 (cameras.yaml);
  - the public APIs of T06 and T07.
- **Fixtures:** none (synthetic tables). Training itself runs in R03b on the GPU workstation, never in a sandbox.

## Goal
The COCO-pretrained detector misses cows at night, under fisheye distortion and in pile-ups. The prior work motivates curating auto-labelled frames before fine-tuning. Build the small toolchain to do the same here:
1. sample frames;
2. auto-label them;
3. let a person keep the good ones;
4. split by hour;
5. train.

## Specification
`cattle/perception/yolo_data.py`:
- **`to_yolo_lines(dets, img_w, img_h, crop_box=None, scale=1.0, classes=("cow", "person", "horse", "vehicle")) -> list[str]`.**
  - Full-res boxes → image coordinates: `(x - crop_x) * scale`.
  - Clip to the image. Drop boxes that end up with zero area.
  - Format each line `f"{cls_idx} {cx:.6f} {cy:.6f} {w:.6f} {h:.6f}"`, normalised to the image size.
- **`from_yolo_lines(lines, img_w, img_h, crop_box=None, scale=1.0, classes=...) -> list[dict]`** is the inverse, returning full-res `cls, x1, y1, x2, y2`.
- **`split_by_hour(df, val_frac=0.2, seed=0) -> (train, val)`.** Group rows by `ts.floor("h")` and assign whole hours to val until it holds ≥ `val_frac` of the rows. No hour appears in both splits.
- **`stratified_sample(keyframes, n, by=("cam", "picture"), seed=0)`.** An equal share per stratum (remainders go to the first strata in sorted order). Deterministic.

Scripts:
- **`scripts/sample_frames.py --n 400`** writes `work/finetune/frames.csv`, stratified by cam × picture, with minimum spacing 10 min.
- **`scripts/autolabel.py --frames ... --weights yolo11x.pt --conf 0.2`.**
  - Writes YOLO label files and a T15 sheet of the frames with boxes drawn, for H03b.
  - The answers CSV has the column `keep` (y/n).
  - `--autodistill` optionally uses GroundedSAM with the prompt `"cow"` if it is installed.
- **`scripts/train_detector.py --picked answers.csv --epochs 60 --imgsz 1280 --model yolo11s.pt`.**
  - Builds `work/finetune/data.yaml` from kept frames, split by hour.
  - Calls `ultralytics` training (lazy import), and prints the val mAP and the path to `best.pt`.
  - Appends the SHA-256 of the weights to `models/SHA256SUMS`.

## Locked test
```python
# LOCKED TEST: tests/unit/test_yolo_data.py
import pandas as pd

from cattle.perception.yolo_data import from_yolo_lines, split_by_hour, stratified_sample, to_yolo_lines

CLASSES = ["cow", "person", "horse", "vehicle"]


def test_round_trip_with_crop_and_scale():
    dets = [dict(cls="cow", x1=1220.0, y1=340.0, x2=1420.0, y2=440.0)]
    lines = to_yolo_lines(dets, img_w=700, img_h=450, crop_box=(1200, 300, 1400, 900), scale=0.5, classes=CLASSES)
    assert lines == ["0 0.085714 0.100000 0.142857 0.111111"]
    back = from_yolo_lines(lines, img_w=700, img_h=450, crop_box=(1200, 300, 1400, 900), scale=0.5, classes=CLASSES)
    assert back[0]["cls"] == "cow" and abs(back[0]["x1"] - 1220.0) < 0.5 and abs(back[0]["y2"] - 440.0) < 0.5


def test_boxes_outside_crop_are_dropped():
    assert to_yolo_lines([dict(cls="cow", x1=0.0, y1=0.0, x2=100.0, y2=100.0)], img_w=700, img_h=450,
                         crop_box=(1200, 300, 1400, 900), scale=0.5, classes=CLASSES) == []


def test_split_by_hour_shares_no_hour():
    t0 = pd.Timestamp("2000-01-10 00:00")
    df = pd.DataFrame(dict(img=[f"{i}.jpg" for i in range(240)], ts=[t0 + pd.Timedelta(minutes=12 * i) for i in range(240)]))
    tr, va = split_by_hour(df, val_frac=0.25, seed=0)
    assert set(tr.ts.dt.floor("h")).isdisjoint(set(va.ts.dt.floor("h"))) and len(tr) + len(va) == 240
    assert 0.2 <= len(va) / 240 <= 0.35


def test_stratified_sample_counts_and_determinism():
    rows = [dict(cam=c, picture=p, img_path=f"{c}/{p}/{i}.jpg", ts=pd.Timestamp("2000-01-10") + pd.Timedelta(minutes=i))
            for c in ["cam01", "cam02", "cam03"] for p in ["colour", "grey_IR"] for i in range(100)]
    kf = pd.DataFrame(rows)
    a = stratified_sample(kf, n=60, by=["cam", "picture"], seed=3)
    b = stratified_sample(kf, n=60, by=["cam", "picture"], seed=3)
    assert len(a) == 60 and a.equals(b) and a.groupby(["cam", "picture"]).size().tolist() == [10] * 6
```

## Done when
- `python scripts/verify_task.py T28` passes.
- `scripts/train_detector.py --help` exits 0 without ultralytics installed.

## Do not
- Train or download models inside the sandbox.
- Mix hours between train and val.
