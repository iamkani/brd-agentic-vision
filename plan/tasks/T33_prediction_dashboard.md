# T33: Interactive prediction dashboard (Streamlit)

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M80 | P0 | Sonnet | 90 min | T31, T32 |

- **Owns:** `app/__init__.py`, `app/logic.py`, `app/dashboard.py`. The orchestrator adds `streamlit` to `requirements.txt` and a Makefile target: `dashboard: .venv/bin/streamlit run app/dashboard.py`.
- **Read:**
  - `02_contracts.md` §8.17, §8.18;
  - the T31 and T32 cards (APIs only);
  - `03_pitfalls.md` row 36.
- **Fixtures:** none (a synthetic database built in the test).

## Why Streamlit
The team asked for "R Shiny or a more modern framework". Streamlit runs on the same Python codebase, so the probability model and database are called directly. It runs locally with no paid access, needs one command (`make dashboard`), and has a headless test harness (`streamlit.testing.v1.AppTest`), so a worker can prove the app works.
- **Alternative:** Shiny for Python, if someone prefers Shiny's reactive style.
- **Fallback:** T25 remains as a static offline export.

## Goal
A live decision aid for the herd manager and the judges, reading the state database (T31):
- **Herd:** every cow's **P(BRD)** with an interval, plus risk rank range, reasons, coverage and flags.
- **Assumptions:** sliders for the prior and the domain effects. Moving one recomputes probabilities instantly.
- **Cow:** time budgets for day 1 against day 2, an ethogram timeline, hourly features against the herd, and evidence clips.
- **Frames:** pick a camera and time, and see the annotated keyframe with each animal's ID, box and state, read from `v_frame_states`.
- **Data quality:** the shares of observed, estimated and assumed; and coverage by camera and hour.
- **Method and limits:** the honest wording.

## Specification

### `app/logic.py`
Pure functions, testable without Streamlit:
- **`load_overview(db_path) -> DataFrame`** joins `brd_probability` and `risk`.
  - Columns: `cow_id`, `P(BRD)`, `P low`, `P high`, `rank`, `rank range` (`"1–2"`), `top_reasons`, `coverage`, `flags`.
  - Sorted by `P(BRD)` descending.
- **`recompute_probability(db_path, prior=None, effects=None, n_boot=50) -> DataFrame`** reads `hourly_features` from the database and calls `brd_probability(...)`. It returns the same columns as `load_overview`.
- **`frame_states(db_path, cam, ts, max_offset_s=60) -> (frame: dict | None, states: DataFrame)`** returns the nearest frame on that camera within the offset, and its `v_frame_states` rows.
- **`annotate(image, states, scale, crop=(0, 0)) -> ndarray`** draws boxes and `C03 standing 0.90 (observed)` labels with PIL.
- **`cow_bouts(db_path, cow_id, day) -> DataFrame`** returns rows from `bouts`, or reconstructs them from `cow_minutes`.

### `app/dashboard.py`
- **Database path:** environment variable `CATTLE_DB`, defaulting to `outputs/pen_states_lite.sqlite`. If the file is missing, show a clear message and stop.
- **Title:** `"Study Pen BRD early-warning dashboard"`.
- **Sidebar sliders:** `key="prior"` (0.01–0.50, default from `configs/prob.yaml`), `key="eff_appetite"`, `key="eff_depression"`, `key="eff_hydration"`. Any change calls `recompute_probability`.
- **Herd tab:** the **first** `st.dataframe` is the overview table, including the `P(BRD)` column. Under it, a bar chart of P(BRD) with interval whiskers.
- **Every tab:** a caption with **"model-estimated probability under stated assumptions; not a diagnosis; veterinary review required"**.
- **Frames tab:**
  - Image root: environment variable `CATTLE_WORK_ROOT`, joined with `img_path`.
  - When the images are not available, show a "frame images not available" message and the states table only.
- **Speed:** cache database reads with `st.cache_data`. Keep runtime on the lite database under 3 s per interaction.

## Locked test
```python
# LOCKED TEST: tests/unit/test_dashboard.py
import numpy as np
import pandas as pd

from cattle.analytics.brd_prob import brd_probability
from cattle.db.state_db import build_state_db

COWS = [f"C{i:02d}" for i in range(1, 11)]
T0 = pd.Timestamp("2000-01-10 10:00:00")
BASE = dict(feeding_min=12.0, feeding_bouts=1.5, bunk_visits=2.0, lying_min=30.0, lying_bouts=1.0, standing_min=12.0,
            walking_min=3.0, drinking_events=0.3, nn_dist_median=0.12, isolation_frac=0.15, visible_min=58.0)
CFG = dict(prior_prevalence=0.10, prior_sd_logit=0.5, effects=dict(appetite=0.15, depression=0.10, hydration=0.05),
           domain_cap=15.0, version="test")


def hourly(seed=0, sick="C07"):
    rng = np.random.default_rng(seed)
    rows = []
    for cow in COWS:
        for day in (1, 2):
            for hour in range(24):
                for f, base in BASE.items():
                    v = max(0.0, base * rng.normal(1.0, 0.12))
                    if cow == sick and day == 2 and f in ("feeding_min", "feeding_bouts", "bunk_visits"):
                        v *= 0.6
                    rows.append(dict(cow_id=cow, day=day, hour=hour, feature=f, value=v, coverage=0.95))
    return pd.DataFrame(rows)


def make_db(tmp_path):
    kf = pd.DataFrame([dict(file_id="F1", cam="cam01", frame_idx=i * 60, ts=T0 + pd.Timedelta(seconds=2 * i), day=1,
                            picture="colour", yavg=120.0, img_path=f"keyframes/cam01/F1/{i * 60:05d}.jpg", crop_box="",
                            scale=0.4) for i in range(3)])
    tr = pd.DataFrame([dict(cam="cam01", track_id="cam01-000001", file_id="F1", frame_idx=i * 60,
                            ts=T0 + pd.Timedelta(seconds=2 * i), det_idx=0, x1=100.0, y1=100.0, x2=500.0, y2=400.0,
                            conf=0.9, cut_reason="", foot_x=300.0, foot_y=400.0, pen_x=0.3, pen_y=0.5, zone="open",
                            speed=0.0) for i in range(3)])
    inputs = dict(
        manifest=pd.DataFrame([dict(file_id="F1", folder_day=1, cam="cam01", rel_path="day1/cam01/F1.Avi", name_start=T0,
                                    size_bytes=1, md5="", channel=1, clock_start=T0, n_frames=200, n_keyframes=3,
                                    picture="colour", satavg=5.0, yavg=120.0)]),
        keyframes=kf, tracks=tr,
        gtracks=pd.DataFrame([dict(gtrack_id="g000001", cam="cam01", track_id="cam01-000001", t_start=T0,
                                   t_end=T0 + pd.Timedelta(seconds=4))]),
        identity=pd.DataFrame([dict(gtrack_id="g000001", t_start=T0, t_end=T0 + pd.Timedelta(seconds=4), cow_id="C03",
                                    confidence=0.8, basis="estimated", method="appearance", margin=0.2)]),
        samples=pd.DataFrame([dict(cow_id="C03", ts=T0 + pd.Timedelta(seconds=2 * i), day=1, label="standing",
                                   confidence=0.9, basis="observed", source="rule", trace_id="") for i in range(3)]),
        animals=pd.DataFrame([dict(cow_id=c, ear_tag="", tag_color="", marks="") for c in COWS]))
    h = hourly()
    prob = brd_probability(h, cfg=CFG, n_boot=20, seed=0)
    order = prob.cow_id.tolist()
    risk = pd.DataFrame(dict(cow_id=order, risk_score=np.linspace(0.9, 0.1, 10), rank=list(range(1, 11)),
                             rank_low=list(range(1, 11)), rank_high=list(range(1, 11)), top_reasons=["x"] * 10,
                             evidence_clips=[""] * 10, coverage=[0.95] * 10, flags=[""] * 10))
    return build_state_db(inputs, tmp_path / "db.sqlite", analytics=dict(risk=risk, probability=prob, hourly=h))


def test_overview_and_recompute(tmp_path):
    from app.logic import load_overview, recompute_probability
    db = make_db(tmp_path)
    ov = load_overview(db)
    assert len(ov) == 10 and {"cow_id", "P(BRD)", "P low", "P high", "rank"} <= set(ov.columns)
    assert ov.iloc[0].cow_id == "C07" and ov["P(BRD)"].is_monotonic_decreasing
    assert recompute_probability(db, prior=0.3, n_boot=10)["P(BRD)"].mean() > ov["P(BRD)"].mean()


def test_frame_states_nearest(tmp_path):
    from app.logic import frame_states
    frame, states = frame_states(make_db(tmp_path), cam="cam01", ts=T0 + pd.Timedelta(seconds=1.2))
    assert frame["frame_idx"] == 60 and list(states.cow_id) == ["C03"] and list(states.behavior) == ["standing"]


def test_app_smoke_and_prior_slider(tmp_path, monkeypatch):
    from streamlit.testing.v1 import AppTest
    monkeypatch.setenv("CATTLE_DB", str(make_db(tmp_path)))
    at = AppTest.from_file("app/dashboard.py", default_timeout=60).run()
    assert not at.exception
    assert any("BRD" in t.value for t in at.title)
    df = at.dataframe[0].value
    assert len(df) == 10 and "P(BRD)" in df.columns
    texts = " ".join(e.value for e in list(at.markdown) + list(at.caption) + list(at.info) + list(at.warning))
    assert "not a diagnosis" in texts
    before = df["P(BRD)"].mean()
    at.slider(key="prior").set_value(0.3).run()
    assert not at.exception and at.dataframe[0].value["P(BRD)"].mean() > before
```

## Done when
- `python scripts/verify_task.py T33` passes.
- A human runs `make dashboard` on the lite database and on a full real database, and checks that every tab renders.

## Do not
- Use any service that needs a paid account or the internet.
- Show `P(BRD)` without the caption.
