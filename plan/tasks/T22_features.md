# T22: Behaviour features per cow-hour

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M70 | P0 | Sonnet | 60 min | T10 |

- **Owns:** `cattle/analytics/features.py`.
- **Read:**
  - `02_contracts.md` §6, §8.7, §8.8, §8.13;
  - `03_pitfalls.md` rows 37 and 38.
- **Fixtures:** none.

## Goal
Turn L1 records into the hourly behavioural features a veterinarian would recognise as early BRD signs:
- reduced feeding and fewer bunk visits;
- more lying;
- less walking;
- isolation from pen-mates.

Each feature carries a coverage value, so the risk engine (T23) can tell "she did less" from "we saw less".

## Specification
`FEATURES` is a module-level list, exactly:
`["feeding_min", "feeding_bouts", "bunk_visits", "lying_min", "lying_bouts", "standing_min", "walking_min", "drinking_events", "nn_dist_median", "isolation_frac", "visible_min"]`.

`hourly_features(minutes, samples=None, isolation_dist=0.25) -> DataFrame` returns long-format rows `cow_id, day, hour, feature, value, coverage`, one row per cow × day × hour × feature.

**From `minutes`** (activity_minutes rows, label names or codes):
- `*_min` features count the minutes with that behaviour in the hour.
  - Ruminating lying counts toward lying.
  - Ruminating standing counts toward standing.
- `feeding_bouts` and `lying_bouts` count runs that **start** in the hour.
- `drinking_events` counts drinking runs that start in the hour.
- `visible_min` counts minutes that are not `other / not visible`.
- `coverage` = `visible_min / 60`, the same value on every feature row of that hour.

**From `samples`** (2-s rows with `nn_dist`, `zone`, `visible`, `staff_present`), when given:
- Drop rows where `staff_present` is true, or `visible` is false.
- `nn_dist_median` = the median `nn_dist` over the remaining rows.
- `isolation_frac` = the share of those rows with `nn_dist > isolation_dist`.
- `bunk_visits` = the number of entries into zone `bunk` (an entry is a transition from not-bunk to bunk, or a run starting in bunk at the start of the hour).

**Without `samples`:**
- `nn_dist_median` and `isolation_frac` are NaN.
- `bunk_visits` = `feeding_bouts`.

## Locked test
```python
# LOCKED TEST: tests/unit/test_features.py
import pandas as pd
import pytest

from cattle.analytics.features import FEATURES, hourly_features


def minutes_for(cow, day, behaviours):
    start = pd.Timestamp("2000-01-10") if day == 1 else pd.Timestamp("2000-01-11")
    labels = (behaviours + ["lying"] * 1440)[:1440]
    return pd.DataFrame(dict(cow_id=cow, day=day, minute=pd.date_range(start, periods=1440, freq="min"), behavior=labels,
                             confidence=0.9, basis="observed", cameras="cam01", n_samples=30))


def samples_for(values, zones=None):
    t0 = pd.Timestamp("2000-01-10 00:00:00")
    zones = zones or ["open"] * len(values)
    return pd.DataFrame([dict(cow_id="C01", ts=t0 + pd.Timedelta(seconds=2 * i), day=1, visible=True, nn_dist=v, zone=z,
                              staff_present=False) for i, (v, z) in enumerate(zip(values, zones))])


def test_feature_names_contract():
    assert FEATURES == ["feeding_min", "feeding_bouts", "bunk_visits", "lying_min", "lying_bouts", "standing_min",
                        "walking_min", "drinking_events", "nn_dist_median", "isolation_frac", "visible_min"]


def test_hourly_counts():
    beh = ["feeding"] * 10 + ["standing"] * 5 + ["feeding"] * 5 + ["walking"] * 3 + ["drinking"] * 2 + ["lying"] * 35
    h = hourly_features(minutes_for("C01", 1, beh))
    one = h[(h.cow_id == "C01") & (h.day == 1) & (h.hour == 0)].set_index("feature").value
    assert one.feeding_min == 15 and one.feeding_bouts == 2 and one.standing_min == 5 and one.walking_min == 3
    assert one.drinking_events == 1 and one.lying_min == 35 and one.lying_bouts == 1 and one.visible_min == 60
    assert len(h) == 24 * len(FEATURES)


def test_not_visible_reduces_coverage():
    h = hourly_features(minutes_for("C01", 1, ["other / not visible"] * 30 + ["lying"] * 30))
    row = h[(h.hour == 0) & (h.feature == "visible_min")].iloc[0]
    assert row.value == 30 and row.coverage == pytest.approx(0.5)


def test_neighbour_features_from_samples():
    s = samples_for([0.05] * 1200 + [0.4] * 600)
    h = hourly_features(minutes_for("C01", 1, []), samples=s).set_index(["hour", "feature"]).value
    assert h[(0, "nn_dist_median")] == pytest.approx(0.05)
    assert h[(0, "isolation_frac")] == pytest.approx(600 / 1800)


def test_bunk_visits_count_entries():
    zones = ["open"] * 10 + ["bunk"] * 20 + ["open"] * 10 + ["bunk"] * 5 + ["open"] * 5
    s = samples_for([0.1] * len(zones), zones)
    h = hourly_features(minutes_for("C01", 1, []), samples=s).set_index(["hour", "feature"]).value
    assert h[(0, "bunk_visits")] == 2
```

## Done when
`python scripts/verify_task.py T22` passes.
