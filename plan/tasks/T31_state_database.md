# T31: State database (every animal's state on every annotated frame)

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M50 | P0 | Sonnet | 75 min | T10, T13, T14, T18 (T23 and T32 for the optional analytics tables) |

- **Owns:** `cattle/db/__init__.py`, `cattle/db/state_db.py`, `scripts/build_state_db.py`.
- **Read:** `02_contracts.md` §4, §5, §7, §8 (all tables, especially §8.17), §10.
- **Fixtures:** none (synthetic tables).

## Goal
One portable file that answers "what was each animal doing on this frame, and how sure are we?" for every annotated keyframe:
- the cow's identity, box, pen position, zone, behaviour state, confidence and basis (observed, estimated or assumed);
- links to the trace and evidence that produced it.

It is a team deliverable in its own right. It is the dashboard's data source (T33), and it lets a judge or vet query the whole study with SQL from R, Python or DB Browser.

## Specification
`build_state_db(inputs, out_path, every_n=1, analytics=None) -> Path` writes a SQLite file with `PRAGMA foreign_keys=ON`.

**Inputs**

`inputs` is a dict of DataFrames. Required keys:
- `manifest` (§8.1), `keyframes` (§8.2), `tracks` with pen columns (§8.4), `gtracks` (§8.5), `identity` (§8.6);
- `samples`: `cow_samples` with `label`, `confidence`, `basis`, `source` and `trace_id` (§8.7);
- `animals`: `labels/cows.csv`. The builder **adds a row `UNK`**, with marks "unidentified".

`analytics` is an optional dict:
- `risk` (§8.14), `probability` (§8.18), `hourly` (§8.13);
- `minutes` (§8.8), `bouts` (§8.9).

Each one present is written as the tables `risk`, `brd_probability`, `hourly_features`, `cow_minutes` and `bouts`.

**Core tables** (contracts §8.17)
- `meta(key PRIMARY KEY, value)`: `contract_version`, `built_at`, `every_n`, and row counts.
- `animals(cow_id PRIMARY KEY, ear_tag, tag_color, marks)`.
- `cameras(cam PRIMARY KEY, role, suffix, channel)`, from `configs/cameras.yaml` when present, else from `CAM_SUFFIX`.
- `files(file_id PRIMARY KEY, cam → cameras, folder_day, rel_path, name_start, n_frames, picture)`.
- `frames(frame_id INTEGER PRIMARY KEY, file_id → files, cam, frame_idx, ts, day, picture, img_path)`, with `UNIQUE(file_id, frame_idx)`.
- `annotations`: one row per tracked animal per kept frame. Columns:
  - `ann_id INTEGER PRIMARY KEY`, `frame_id` → frames, `cow_id` → animals;
  - `track_id`, `gtrack_id`;
  - `x1, y1, x2, y2`, `pen_x, pen_y`, `zone`;
  - `behavior` (class code or NULL), `behavior_label`, `confidence`, `basis`;
  - `identity_conf`, `identity_method`, `source`, `trace_id`.
- **View `v_frame_states`:** `frames ⋈ annotations ⋈ animals`, exposing `cam, file_id, frame_idx, ts, day, picture, img_path, cow_id, marks, track_id, gtrack_id, x1..y2, pen_x, pen_y, zone, behavior, behavior_label, confidence, basis, identity_conf, identity_method, source, trace_id`.
- **View `v_cow_day_budget`:** minutes per cow × day × behaviour, from `cow_minutes` (present only if `minutes` is given).
- **Indices:** `annotations(frame_id)`, `annotations(cow_id, frame_id)`, `frames(cam, ts)`.

**Join rules**
- A track row gets its `gtrack_id` from `gtracks` (same `track_id`) and its cow from the `identity` row covering its `ts`. With no identity it becomes `UNK`.
- Behaviour, confidence, basis, source and trace come from `samples` at (`cow_id`, `ts.floor("2s")`). `UNK` and missing samples get `behavior = NULL` with the identity's basis.
- Timestamps are stored as ISO text with milliseconds.

**Validation and options**
- Before writing, check the enums (basis, zone, picture, behaviour codes, cow IDs). Any violation raises `ValueError` listing the offending values.
- `every_n` keeps every n-th keyframe per file, by sorted `frame_idx`, together with its annotations (the "lite" build).

**CLI**

`scripts/build_state_db.py --work DIR --outputs outputs --every-n N --out PATH`
- **Full build:** `work/pen_states.sqlite`, every keyframe, about 250 MB.
- **Lite build:** `outputs/pen_states_lite.sqlite` with `--every-n 15`, one frame per ~30 s, < 50 MB. This one ships in the ZIP.
- Prints the counts and file size, then runs `PRAGMA integrity_check` and `PRAGMA foreign_key_check`.

## Locked test
```python
# LOCKED TEST: tests/unit/test_state_db.py
import sqlite3

import pandas as pd
import pytest

from cattle.db.state_db import build_state_db

T0 = pd.Timestamp("2000-01-10 10:00:00")
COWS = [f"C{i:02d}" for i in range(1, 11)]


def inputs():
    manifest = pd.DataFrame([dict(file_id="F1", folder_day=1, cam="cam01", rel_path="day1/cam01/F1.Avi", name_start=T0,
                                  size_bytes=1, md5="", channel=1, clock_start=T0, n_frames=200, n_keyframes=3,
                                  picture="colour", satavg=5.0, yavg=120.0)])
    keyframes = pd.DataFrame([dict(file_id="F1", cam="cam01", frame_idx=i * 60, ts=T0 + pd.Timedelta(seconds=2 * i), day=1,
                                   picture="colour", yavg=120.0, img_path=f"keyframes/cam01/F1/{i * 60:05d}.jpg",
                                   crop_box="", scale=0.4) for i in range(3)])
    tracks = pd.DataFrame([dict(cam="cam01", track_id="cam01-000001", file_id="F1", frame_idx=i * 60,
                                ts=T0 + pd.Timedelta(seconds=2 * i), det_idx=0, x1=100.0, y1=100.0, x2=500.0, y2=400.0,
                                conf=0.9, cut_reason="", foot_x=300.0, foot_y=400.0, pen_x=0.3, pen_y=0.5, zone="open",
                                speed=0.0) for i in range(3)])
    gtracks = pd.DataFrame([dict(gtrack_id="g000001", cam="cam01", track_id="cam01-000001", t_start=T0,
                                 t_end=T0 + pd.Timedelta(seconds=4))])
    identity = pd.DataFrame([dict(gtrack_id="g000001", t_start=T0, t_end=T0 + pd.Timedelta(seconds=4), cow_id="C03",
                                  confidence=0.8, basis="estimated", method="appearance", margin=0.2)])
    samples = pd.DataFrame([dict(cow_id="C03", ts=T0 + pd.Timedelta(seconds=2 * i), day=1, label="standing",
                                 confidence=0.9, basis="observed", source="rule", trace_id="") for i in range(3)])
    animals = pd.DataFrame([dict(cow_id=c, ear_tag="", tag_color="", marks="") for c in COWS])
    return dict(manifest=manifest, keyframes=keyframes, tracks=tracks, gtracks=gtracks, identity=identity,
                samples=samples, animals=animals)


def test_build_and_query(tmp_path):
    con = sqlite3.connect(build_state_db(inputs(), tmp_path / "s.sqlite"))
    names = {r[0] for r in con.execute("select name from sqlite_master where type in ('table', 'view')")}
    assert {"meta", "animals", "cameras", "files", "frames", "annotations", "v_frame_states"} <= names
    assert con.execute("select count(*) from frames").fetchone()[0] == 3
    assert con.execute("select count(*) from annotations").fetchone()[0] == 3
    row = con.execute("select cow_id, behavior, behavior_label, basis, zone, identity_method from v_frame_states "
                      "where cam = 'cam01' and frame_idx = 60").fetchone()
    assert row == ("C03", "standing", "standing", "observed", "open", "appearance")
    assert con.execute("pragma foreign_key_check").fetchall() == []
    assert con.execute("pragma integrity_check").fetchone()[0] == "ok"
    assert con.execute("select count(*) from animals where cow_id = 'UNK'").fetchone()[0] == 1


def test_lite_keeps_every_nth_frame(tmp_path):
    con = sqlite3.connect(build_state_db(inputs(), tmp_path / "lite.sqlite", every_n=2))
    assert [r[0] for r in con.execute("select frame_idx from frames order by frame_idx")] == [0, 120]
    assert con.execute("select count(*) from annotations").fetchone()[0] == 2


def test_rejects_bad_enum(tmp_path):
    bad = inputs()
    bad["samples"].loc[0, "basis"] = "guessed"
    with pytest.raises(ValueError):
        build_state_db(bad, tmp_path / "bad.sqlite")


def test_unknown_identity_is_unk(tmp_path):
    inp = inputs()
    inp["identity"].loc[0, "cow_id"] = "UNK"
    con = sqlite3.connect(build_state_db(inp, tmp_path / "u.sqlite"))
    assert [r for r in con.execute("select distinct cow_id, behavior from annotations")] == [("UNK", None)]


def test_analytics_tables(tmp_path):
    risk = pd.DataFrame(dict(cow_id=COWS, risk_score=[0.5] * 10, rank=list(range(1, 11)), rank_low=[1] * 10,
                             rank_high=[10] * 10, top_reasons=["x"] * 10, evidence_clips=[""] * 10, coverage=[0.9] * 10,
                             flags=[""] * 10))
    con = sqlite3.connect(build_state_db(inputs(), tmp_path / "a.sqlite", analytics=dict(risk=risk)))
    assert con.execute("select count(*) from risk").fetchone()[0] == 10
```

## Done when
`python scripts/verify_task.py T31` passes. The report includes the full and lite sizes from R05.
