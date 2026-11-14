# T13: Cross-camera linking

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M40 | P0 | Sonnet | 75 min | T08, T09 |

- **Owns:** `cattle/identity/link.py`.
- **Read:**
  - `02_contracts.md` §4 (gtrack_id), §8.4, §8.5, and the identity block of §10;
  - `03_pitfalls.md` row 19.
- **Fixtures:** none (synthetic tracks with pen coordinates).

## Goal
The brief wants one ID per cow across all three cameras. When two cameras see a cow at the same pen position at the same time, it is the same cow. Group per-camera tracks into **global tracklets** (gtracks) by pen-coordinate agreement over time. This shrinks the identity problem and lets one camera's good view name the cow in another's.

## Specification
`link(tracks, max_dist=0.08, bin_s=2.0, min_shared_bins=3) -> DataFrame` with columns `gtrack_id, cam, track_id, t_start, t_end`:
1. **Bin the data.** Drop rows with NaN `pen_x` or `pen_y`. Put rows into time bins `ts.floor(f"{bin_s}s")`. Within each bin, average each track's position.
2. **Match per bin and camera pair.** For each bin and each pair of cameras, solve a one-to-one assignment between the two cameras' tracks with the Hungarian algorithm on Euclidean pen distance. Accept a pair only if its distance is ≤ `max_dist`. Count the accepted pair occurrences across bins.
3. **Keep strong pairs.** A track pair from different cameras is linked if it was matched in ≥ `min_shared_bins` bins **and** matched in at least 60% of the bins where both are present.
4. **Group.** Union-find over the linked pairs gives the gtracks. Tracks of the same camera never link directly, but can join a gtrack through another camera's track (continuity bridging).
5. **Number deterministically.** Sort gtracks by their earliest `t_start`, then by their smallest `track_id`. Name them `g000001`, `g000002`, …. Unlinked tracks become singleton gtracks.
6. **Output.** One row per member track, with that track's own `t_start` and `t_end`.

## Locked test
```python
# LOCKED TEST: tests/unit/test_link.py
import numpy as np
import pandas as pd

from cattle.identity.link import link

T0 = pd.Timestamp("2000-01-10 10:00:00")


def rows(cam, tid, xs, ys, start_s=0, step=2.0, noise=0.0, seed=0):
    rng = np.random.default_rng(seed)
    return [dict(cam=cam, track_id=tid, file_id="F", frame_idx=i, ts=T0 + pd.Timedelta(seconds=start_s + i * step),
                 det_idx=0, x1=0.0, y1=0.0, x2=1.0, y2=1.0, conf=0.9, cut_reason="",
                 pen_x=x + rng.normal(0, noise), pen_y=y + rng.normal(0, noise), zone="open", speed=0.0)
            for i, (x, y) in enumerate(zip(xs, ys))]


def test_same_cow_two_cameras_linked():
    xs, ys = np.linspace(0.2, 0.3, 10), np.full(10, 0.5)
    g = link(pd.DataFrame(rows("cam01", "cam01-000001", xs, ys) + rows("cam02", "cam02-000001", xs, ys, noise=0.01, seed=1)))
    assert g.gtrack_id.nunique() == 1 and set(g.cam) == {"cam01", "cam02"}


def test_far_apart_not_linked():
    g = link(pd.DataFrame(rows("cam01", "cam01-000001", np.full(10, 0.2), np.full(10, 0.2)) +
                          rows("cam02", "cam02-000001", np.full(10, 0.8), np.full(10, 0.8))))
    assert g.gtrack_id.nunique() == 2


def test_one_to_one_per_bin():
    g = link(pd.DataFrame(rows("cam01", "cam01-000001", np.full(10, 0.50), np.full(10, 0.5)) +
                          rows("cam01", "cam01-000002", np.full(10, 0.56), np.full(10, 0.5)) +
                          rows("cam02", "cam02-000001", np.full(10, 0.51), np.full(10, 0.5))))
    members = g.groupby("gtrack_id").track_id.apply(set).tolist()
    assert {"cam01-000001", "cam02-000001"} in members and {"cam01-000002"} in members


def test_three_cameras_and_singleton():
    xs = np.linspace(0.4, 0.45, 10)
    g = link(pd.DataFrame(rows("cam01", "cam01-000001", xs, np.full(10, 0.3)) +
                          rows("cam02", "cam02-000001", xs, np.full(10, 0.3), noise=0.005, seed=2) +
                          rows("cam03", "cam03-000001", xs, np.full(10, 0.3), noise=0.005, seed=3) +
                          rows("cam03", "cam03-000002", np.full(10, 0.9), np.full(10, 0.9))))
    assert g.gtrack_id.nunique() == 2 and g.groupby("gtrack_id").track_id.nunique().max() == 3


def test_needs_min_shared_bins():
    g = link(pd.DataFrame(rows("cam01", "cam01-000001", np.full(10, 0.5), np.full(10, 0.5)) +
                          rows("cam02", "cam02-000001", np.full(2, 0.5), np.full(2, 0.5))), min_shared_bins=3)
    assert g.gtrack_id.nunique() == 2


def test_ids_and_columns():
    g = link(pd.DataFrame(rows("cam01", "cam01-000001", np.full(5, 0.3), np.full(5, 0.5))))
    assert g.gtrack_id.tolist() == ["g000001"]
    assert list(g.columns) == ["gtrack_id", "cam", "track_id", "t_start", "t_end"]
```

## Done when
`python scripts/verify_task.py T13` passes.
