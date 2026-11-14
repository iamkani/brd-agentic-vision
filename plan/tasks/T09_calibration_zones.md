# T09: Calibration, pen coordinates, zones

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M20 | P0 | Haiku | 60 min | T05 |

- **Owns:** `cattle/calib/homography.py`, `cattle/calib/zones.py`, `scripts/click_points.py`.
- **Read:**
  - `02_contracts.md` §1 (row Pen map), §8.4, §9 (calib, zones), and the calib block of §10;
  - `03_pitfalls.md` row 15.
- **Fixtures:** none.

## Goal
Map each camera's pixels to one shared pen coordinate system, so cows seen by different cameras can be matched (T13) and assigned to zones (bunk, water, rest) for behaviour rules (T18).

**Pen coordinates** are normalised:
- x = 0 at the chute-side (left) fence and 1 at the right fence;
- y = 0 at the BACK fence and 1 at the FRONT (feed bunk) edge.

## Specification
`cattle/calib/homography.py`:
- **`fit_homography(px, pen) -> (H, rmse)`:**
  - Normalised DLT (Hartley normalisation) or `cv2.findHomography(px, pen, 0)`.
  - ≥ 4 point pairs, otherwise `ValueError`.
  - `rmse` is in pen units.
- **`to_pen(H, xy) -> ndarray`:** projects an N×2 array.
- **`foot_point(x1, y1, x2, y2) -> (x, y)`:** the bottom-centre `((x1 + x2) / 2, y2)` as floats.
- **`save_calib(cam, points, H, rmse, root="configs/calib")` / `load_calib(cam, root="configs/calib") -> dict`:** JSON as in contracts §9. `points` is a list of `{"name", "px", "pen"}`.

`cattle/calib/zones.py`:
- **`zone_of(pen_xy, zones) -> str`:**
  - `unknown` if either coordinate is NaN;
  - `outside` if x or y lies outside [−0.02, 1.02];
  - otherwise the first polygon containing the point, in priority order `water` > `bunk` > `rest`;
  - otherwise `open`.
- **`add_pen_coords(tracks, calibs, zones) -> DataFrame`:**
  - `calibs` maps `cam` to H.
  - Adds `foot_x`, `foot_y`, `pen_x`, `pen_y` (NaN when the camera has no H), `zone`, and `speed`. `speed` is pen units per second between consecutive rows of the same `track_id`, and NaN on a track's first row.

`scripts/click_points.py --image JPG --scale S --out JSON`:
- A manual helper. matplotlib shows the image; each click is followed by a typed name and pen coordinates in the terminal.
- It saves points with `px` multiplied back to full resolution (÷ scale).
- It is not tested beyond `--help`.

## Locked test
```python
# LOCKED TEST: tests/unit/test_calib.py
import numpy as np
import pandas as pd
import pytest

from cattle.calib.homography import fit_homography, foot_point, load_calib, save_calib, to_pen
from cattle.calib.zones import add_pen_coords, zone_of

H_TRUE = np.array([[0.00025, 0.00002, -0.1], [0.00001, 0.0003, -0.05], [0.000001, 0.000002, 1.0]])
PX = np.array([[500, 400], [3500, 420], [3600, 2800], [400, 2700], [2000, 1500], [1200, 900]], float)


def project(H, pts):
    p = np.c_[pts, np.ones(len(pts))] @ H.T
    return p[:, :2] / p[:, 2:3]


def test_fit_recovers_known_homography():
    pen = project(H_TRUE, PX)
    H, rmse = fit_homography(PX, pen)
    assert rmse < 1e-6 and np.allclose(to_pen(H, PX), pen, atol=1e-6)


def test_fit_needs_four_points():
    with pytest.raises(ValueError):
        fit_homography(PX[:3], project(H_TRUE, PX[:3]))


def test_foot_point():
    assert foot_point(100, 200, 300, 500) == (200.0, 500.0)


ZONES = {"water": [[0.8, 0.0], [1.0, 0.0], [1.0, 0.1], [0.8, 0.1]],
         "bunk": [[0.0, 0.92], [1.0, 0.92], [1.0, 1.0], [0.0, 1.0]],
         "rest": [[0.0, 0.0], [0.5, 0.0], [0.5, 0.5], [0.0, 0.5]]}


@pytest.mark.parametrize("xy, zone", [((0.9, 0.05), "water"), ((0.5, 0.95), "bunk"), ((0.2, 0.2), "rest"),
                                      ((0.7, 0.6), "open"), ((1.5, 0.5), "outside"), ((float("nan"), 0.5), "unknown")])
def test_zone_of(xy, zone):
    assert zone_of(xy, ZONES) == zone


def test_zone_priority_water_over_rest():
    assert zone_of((0.9, 0.05), dict(ZONES, rest=[[0.0, 0.0], [1.0, 0.0], [1.0, 0.5], [0.0, 0.5]])) == "water"


def test_save_load_roundtrip(tmp_path):
    pen = project(H_TRUE, PX)
    H, rmse = fit_homography(PX, pen)
    pts = [{"name": f"p{i}", "px": PX[i].tolist(), "pen": pen[i].tolist()} for i in range(len(PX))]
    save_calib("cam01", pts, H, rmse, root=tmp_path)
    c = load_calib("cam01", root=tmp_path)
    assert c["cam"] == "cam01" and np.allclose(np.array(c["H"]), H) and len(c["points"]) == 6


def test_add_pen_coords_and_speed():
    H, _ = fit_homography(PX, project(H_TRUE, PX))
    t0 = pd.Timestamp("2000-01-10 10:00")
    tr = pd.DataFrame([dict(cam="cam01", track_id="cam01-000001", file_id="F", frame_idx=i * 60,
                            ts=t0 + pd.Timedelta(seconds=2 * i), det_idx=0, x1=1800 + 100 * i, y1=1200,
                            x2=2200 + 100 * i, y2=1500, conf=0.9, cut_reason="") for i in range(3)])
    out = add_pen_coords(tr, {"cam01": H}, ZONES)
    for col in ["foot_x", "foot_y", "pen_x", "pen_y", "zone", "speed"]:
        assert col in out
    assert out.foot_x.tolist() == [2000.0, 2100.0, 2200.0]
    assert np.isnan(out.speed.iloc[0]) and out.speed.iloc[1] > 0
    assert np.allclose([out.pen_x.iloc[0], out.pen_y.iloc[0]], to_pen(H, np.array([[2000.0, 1500.0]]))[0])


def test_uncalibrated_camera():
    tr = pd.DataFrame([dict(cam="cam02", track_id="cam02-000001", file_id="F", frame_idx=0,
                            ts=pd.Timestamp("2000-01-10 10:00"), det_idx=0, x1=0, y1=0, x2=10, y2=10, conf=0.9,
                            cut_reason="")])
    out = add_pen_coords(tr, {}, ZONES)
    assert np.isnan(out.pen_x.iloc[0]) and out.zone.iloc[0] == "unknown"
```

## Done when
`python scripts/verify_task.py T09` passes, and `python -m scripts.click_points --help` exits 0.

## Do not
- Hard-code any camera's calibration. Real points come from the human task H02.
