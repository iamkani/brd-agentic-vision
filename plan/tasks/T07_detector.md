# T07: Detector wrapper

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M20 | P0 | Sonnet | 60 min | T05 |

- **Owns:** `cattle/perception/detect.py`, `scripts/detect.py`.
- **Read:**
  - `02_contracts.md` §8.2, §8.3, §9 (cameras.yaml masks), and the `Detector` block of §10;
  - `03_pitfalls.md` rows 13, 16 and 17.
- **Fixtures:** none (fake backend).

## Goal
Run an object detector on keyframe images and return boxes for cows, people, horses and vehicles in **full-resolution pixel coordinates**, so every later stage can share one coordinate system. The detector backend can be swapped, so tests run without a GPU or model download.

## Specification
- **`Detector(weights="yolo11s.pt", imgsz=1280, conf=0.25, device=None, backend=None, masks=None)`.**
  - **`backend`:** a callable taking a list of RGB uint8 images and returning, per image, a list of tuples `(class_name, conf, x1, y1, x2, y2)` in that image's pixels.
  - If `backend` is `None`, build an Ultralytics backend lazily:
    - `from ultralytics import YOLO`;
    - device auto-detected as `cuda` → `mps` → `cpu`;
    - `model(images, imgsz=imgsz, conf=conf, verbose=False)`;
    - map class ids to names via `model.names`.
  - **`masks`:** a list of polygons in **full-resolution** pixels. A box is dropped if its foot point (bottom-centre) falls inside any mask.
  - **`CLASS_MAP`:** cow → cow, person → person, horse → horse, and truck, car, bus, motorcycle, tractor → vehicle. Every other class is dropped. Boxes below `conf` are dropped.
- **`detect_image(img, crop_box=None, scale=1.0) -> list[dict]`.** Maps image coordinates back to full resolution:
  - `X = crop_x + x / scale` and `Y = crop_y + y / scale`, with crop = (0, 0) when there is no `crop_box`;
  - clip to [0, 4000] × [0, 3000];
  - each dict has keys `cls`, `conf`, `x1`, `y1`, `x2`, `y2`.
- **`run(keyframes, images_root, batch=16) -> DataFrame`.**
  - Reads the JPEGs from `images_root / img_path` and calls the backend once per batch.
  - Returns rows in the `detections` schema; `det_idx` is 0..n−1 within each frame.
  - `model` = the weights file name plus `ultralytics.__version__` when available, or `"fake"`.
  - Resume: skip frames already present in an existing output Parquet when `--resume` is given (CLI).
- **`scripts/detect.py`:**
  - Flags: `--keyframes`, `--weights`, `--imgsz`, `--conf`, `--batch`, `--device`, `--out` (default `<work_root>/detections.parquet`), `--resume`, and `--benchmark N`.
  - `--benchmark N` runs N frames only and prints frames per second.
  - Loads masks per camera from `configs/cameras.yaml`.

## Locked test
```python
# LOCKED TEST: tests/unit/test_detect.py
import numpy as np
import pandas as pd
from PIL import Image

from cattle.perception.detect import Detector
from cattle.validate.schemas import validate


class FakeBackend:
    def __init__(self, boxes):
        self.boxes = boxes
        self.calls = 0

    def __call__(self, images):
        self.calls += 1
        return [list(self.boxes) for _ in images]


def test_coordinates_map_back_to_full_frame():
    det = Detector(backend=FakeBackend([("cow", 0.9, 10, 20, 110, 70)]))
    out = det.detect_image(np.zeros((450, 700, 3), np.uint8), crop_box=(1200, 300, 1400, 900), scale=0.5)
    assert len(out) == 1
    b = out[0]
    assert b["cls"] == "cow" and (b["x1"], b["y1"], b["x2"], b["y2"]) == (1220.0, 340.0, 1420.0, 440.0)


def test_class_map_and_conf_filter():
    boxes = [("cow", 0.9, 0, 0, 10, 10), ("person", 0.8, 0, 0, 10, 10), ("horse", 0.7, 0, 0, 10, 10),
             ("truck", 0.6, 0, 0, 10, 10), ("car", 0.6, 0, 0, 10, 10), ("dog", 0.9, 0, 0, 10, 10),
             ("cow", 0.1, 0, 0, 10, 10)]
    det = Detector(conf=0.25, backend=FakeBackend(boxes))
    got = sorted(b["cls"] for b in det.detect_image(np.zeros((100, 100, 3), np.uint8)))
    assert got == ["cow", "horse", "person", "vehicle", "vehicle"]


def test_masks_drop_boxes_by_foot_point():
    det = Detector(backend=FakeBackend([("cow", 0.9, 0, 0, 20, 20), ("cow", 0.9, 200, 200, 260, 260)]),
                   masks=[[(0, 0), (100, 0), (100, 100), (0, 100)]])
    got = det.detect_image(np.zeros((300, 300, 3), np.uint8))
    assert len(got) == 1 and got[0]["x1"] == 200.0


def test_run_produces_valid_detections(tmp_path):
    rows = []
    for i in range(3):
        rel = f"keyframes/cam01/F/{i:05d}.jpg"
        (tmp_path / rel).parent.mkdir(parents=True, exist_ok=True)
        Image.fromarray(np.zeros((300, 400, 3), np.uint8)).save(tmp_path / rel)
        rows.append(dict(file_id="F", cam="cam01", frame_idx=i, day=1, picture="colour", yavg=100.0, img_path=rel,
                         crop_box="", scale=0.1, ts=pd.Timestamp("2000-01-10 10:00") + pd.Timedelta(seconds=2 * i)))
    fake = FakeBackend([("cow", 0.9, 10, 10, 50, 40)])
    df = Detector(backend=fake).run(pd.DataFrame(rows), images_root=tmp_path, batch=2)
    assert len(df) == 3 and validate("detections", df) == []
    assert df.x1.iloc[0] == 100.0 and df.x2.iloc[0] == 500.0
    assert fake.calls == 2 and list(df.det_idx) == [0, 0, 0]
```

## Done when
- `python scripts/verify_task.py T07` passes.
- The report says which weights you suggest: `yolo11s.pt` (COCO) as the baseline, fine-tuning as a stretch.

## Do not
- Import `ultralytics` at module import time. Import it lazily, so tests run without it.
- Download models in tests.
