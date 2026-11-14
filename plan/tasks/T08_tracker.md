# T08: Tracker for 0.5 fps keyframes

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M20 | P0 | Sonnet | 75 min | T05 |

- **Owns:** `cattle/perception/track.py`.
- **Read:**
  - `02_contracts.md` §4 (track_id), §8.3, §8.4, and the `track` block of §10;
  - `03_pitfalls.md` row 18.
- **Fixtures:** none (synthetic detections).

## Goal
Link per-frame cow boxes into tracklets within each camera. Frames are 2 s apart (keyframes only), so motion between frames is large and ByteTrack-style defaults won't hold. **Cut tracks at pile-ups**, so an identity is never carried blindly through merged boxes. Short, honest tracklets are the goal; T13 and T14 join them later.

## Specification
`track(dets, max_gap_s=6.0, min_iou=0.1, max_jump=0.25, merge_iou=0.6) -> DataFrame`

- **Input:** rows in the `detections` schema. Only `cls == "cow"` rows are tracked. Each camera is tracked independently, ordered by `ts`.
- **Association** at each frame (Hungarian, `scipy.optimize.linear_sum_assignment`):
  - Candidate pairs between active tracks (last box) and new boxes need either IoU ≥ `min_iou`, or a centre distance ≤ `max_jump` × the image diagonal (5000 px).
  - Cost = `1 - IoU + centre_distance / diagonal`.
  - Infeasible pairs get a cost of 1e6 and are never matched.
- **Merge cut:** if two of this frame's boxes have mutual IoU > `merge_iou`, every track that would continue into either box ends at its previous row with `cut_reason="merge"`. Both boxes start new tracks.
- **Gap:** a track not matched for more than `max_gap_s` seconds ends with `cut_reason="gap"` on its last row. A shorter gap keeps the track alive.
- **End:** the last row of any other track gets `cut_reason="end"`. All other rows get `""`.
- **IDs:** `track_id = f"{cam}-{n:06d}"`, numbered per camera from 1 in order of first appearance. The output must be deterministic.
- **Output columns:** `cam, track_id, file_id, frame_idx, ts, det_idx, x1, y1, x2, y2, conf, cut_reason`.

## Locked test
```python
# LOCKED TEST: tests/unit/test_track.py
import pandas as pd

from cattle.perception.track import track

T0 = pd.Timestamp("2000-01-10 10:00:00")


def det_rows(frames, cam="cam01", step_s=2.0):
    rows = []
    for f, boxes in enumerate(frames):
        for j, box in enumerate(boxes):
            x1, y1, x2, y2 = box[:4]
            cls = box[4] if len(box) > 4 else "cow"
            rows.append(dict(file_id="F", cam=cam, frame_idx=f * 60, ts=T0 + pd.Timedelta(seconds=f * step_s),
                             det_idx=j, cls=cls, conf=0.9, x1=x1, y1=y1, x2=x2, y2=y2, model="fake"))
    return pd.DataFrame(rows)


def test_single_cow_one_track():
    t = track(det_rows([[(100 + 20 * i, 100, 400 + 20 * i, 300)] for i in range(10)]))
    assert t.track_id.nunique() == 1 and len(t) == 10 and t.track_id.iloc[0] == "cam01-000001"


def test_two_cows_stay_separate():
    frames = [[(100 + 10 * i, 100, 400 + 10 * i, 300), (2000 - 10 * i, 1500, 2300 - 10 * i, 1700)] for i in range(8)]
    t = track(det_rows(frames))
    assert t.track_id.nunique() == 2
    for _, g in t.groupby("track_id"):
        assert g.y1.nunique() == 1


def test_merge_cuts_tracks():
    frames = []
    for i in range(10):
        if 4 <= i <= 5:
            frames.append([(1000, 1000, 1400, 1300), (1010, 1000, 1410, 1300)])
        else:
            off = 600 if i < 4 else 650
            frames.append([(1000 - off, 1000, 1400 - off, 1300), (1000 + off, 1000, 1400 + off, 1300)])
    t = track(det_rows(frames))
    before = set(t[t.frame_idx < 4 * 60].track_id)
    after = set(t[t.frame_idx > 5 * 60].track_id)
    assert before and after and before.isdisjoint(after)
    assert "merge" in set(t.cut_reason)


def test_gap_rules():
    box = [(100, 100, 400, 300)]
    assert track(det_rows([box, box, [], box, box])).track_id.nunique() == 1
    t = track(det_rows([box, box, [], [], [], [], box]))
    assert t.track_id.nunique() == 2 and "gap" in set(t.cut_reason)


def test_only_cows_tracked_and_cameras_independent():
    d = pd.concat([det_rows([[(100, 100, 400, 300, "person")]] * 3),
                   det_rows([[(100, 100, 400, 300)]] * 3, cam="cam02")])
    t = track(d)
    assert set(t.cam) == {"cam02"} and t.track_id.iloc[0] == "cam02-000001"


def test_deterministic_and_columns():
    d = det_rows([[(100 + 20 * i, 100, 400 + 20 * i, 300)] for i in range(5)])
    a, b = track(d), track(d)
    pd.testing.assert_frame_equal(a, b)
    assert list(a.columns) == ["cam", "track_id", "file_id", "frame_idx", "ts", "det_idx", "x1", "y1", "x2", "y2",
                               "conf", "cut_reason"]
```

## Done when
`python scripts/verify_task.py T08` passes.

## Do not
- Use appearance features. That is T14's job.
- Merge tracks across cameras. That is T13's job.
