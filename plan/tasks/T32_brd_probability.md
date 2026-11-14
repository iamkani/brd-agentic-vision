# T32: Assumption-based BRD probability per animal

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M70 | P0 | Sonnet (orchestrator reviews the wording) | 60 min | T22, T23 (`risk.domain_scores`) |

- **Owns:** `cattle/analytics/brd_prob.py`. The orchestrator adds `configs/prob.yaml` from the content below.
- **Read:**
  - `02_contracts.md` §8.13, §8.18;
  - the T23 card (method steps 1–4 and `domain_scores`);
  - `03_pitfalls.md` row 36;
  - `06_prior_work.md` §2.
- **Fixtures:** none (synthetic herds).

## Goal
The team wants a dashboard that shows **each animal's probability of having BRD**. These data have no health records, so a calibrated probability is impossible. What we can give honestly is a **model-estimated probability under stated, adjustable assumptions**:
- a prior prevalence;
- how strongly each DART domain deviation shifts the odds.

Show the uncertainty, and make every assumption visible and changeable in the dashboard (T33). If the organizers later share health outcomes, `calibrate()` can replace the assumptions with fitted values.

## Model
1. **Domain scores.** `domain_d` for d in {appetite, depression, hydration} comes from `cattle.analytics.risk.domain_scores(hourly)`. Positive means worse than the herd, which is T23 steps 1–4.
2. **Log-odds.** `logit(p) = logit(prior) + Σ_d effect_d × clip(domain_d, −cap, cap)`.
3. **Interval.**
   - For b = 1..n_boot, resample hours with replacement (the same hour indices for every cow) and recompute the domain scores.
   - Draw `logit(prior_b) ~ Normal(logit(prior), prior_sd_logit)`.
   - `p_low` and `p_high` are the 5th and 95th percentiles of p_b, clipped so `p_low ≤ p_brd ≤ p_high`.
4. **Contributions.** `<d>_contrib` = `effect_d × clip(domain_d)`, in log-odds units, for the "why" display.
5. **Labels.** Every row gets `basis = "assumed"` and `calibrated = False`. `assumptions_version` comes from the config.

**Optional:** `calibrate(hourly, labels)` fits the effects by L2-penalised logistic regression when `labels` (cow_id → 0/1) exist. It then sets `calibrated = True`. It is not tested here.

### `configs/prob.yaml` (content for the orchestrator)
```yaml
prior_prevalence: 0.10      # PLACEHOLDER. H10 replaces it with a literature value for adult cows in this setting.
prior_sd_logit: 0.5         # uncertainty about the prior
effects:                    # log-odds per unit of domain score (worse is positive)
  appetite: 0.15
  depression: 0.10
  hydration: 0.05
domain_cap: 15.0
version: "assumptions-v1"
```

## Specification
`brd_probability(hourly, cfg=None, n_boot=200, seed=0, prior=None, effects=None) -> DataFrame`
- `cfg` defaults to `configs/prob.yaml`. `prior` and `effects` override it; this is how the dashboard sliders work.
- Columns, exactly: `cow_id, p_brd, p_low, p_high, prior, appetite_contrib, depression_contrib, hydration_contrib, basis, calibrated, assumptions_version`.
- Rows are sorted by `p_brd`, descending. Deterministic for the same seed.

**Wording rule:** everywhere `p_brd` appears (dashboard, cards, brief), say **"model-estimated probability under stated assumptions (prior X%), not a diagnosis; veterinary review required"**.

## Locked test
```python
# LOCKED TEST: tests/unit/test_brd_prob.py
import numpy as np
import pandas as pd

from cattle.analytics.brd_prob import brd_probability

COWS = [f"C{i:02d}" for i in range(1, 11)]
BASE = dict(feeding_min=12.0, feeding_bouts=1.5, bunk_visits=2.0, lying_min=30.0, lying_bouts=1.0, standing_min=12.0,
            walking_min=3.0, drinking_events=0.3, nn_dist_median=0.12, isolation_frac=0.15, visible_min=58.0)
CFG = dict(prior_prevalence=0.10, prior_sd_logit=0.5, effects=dict(appetite=0.15, depression=0.10, hydration=0.05),
           domain_cap=15.0, version="test")


def herd(seed=0, sick=None, feed_mult=0.65, lying_mult=1.25, iso_mult=2.0):
    rng = np.random.default_rng(seed)
    rows = []
    for cow in COWS:
        for day in (1, 2):
            for hour in range(24):
                for f, base in BASE.items():
                    v = max(0.0, base * rng.normal(1.0, 0.12))
                    if cow == sick and day == 2:
                        if f in ("feeding_min", "feeding_bouts", "bunk_visits"):
                            v *= feed_mult
                        elif f == "lying_min":
                            v *= lying_mult
                        elif f == "isolation_frac":
                            v *= iso_mult
                    rows.append(dict(cow_id=cow, day=day, hour=hour, feature=f, value=v, coverage=0.95))
    return pd.DataFrame(rows)


def test_null_herd_stays_near_prior():
    p = brd_probability(herd(seed=4), cfg=CFG, n_boot=30, seed=0)
    assert p.p_brd.between(0.04, 0.25).all()


def test_injected_cow_stands_out():
    p = brd_probability(herd(sick="C07"), cfg=CFG, n_boot=30, seed=0).set_index("cow_id").p_brd
    assert p["C07"] > 0.6 and p["C07"] > 3 * p.drop("C07").median()


def test_zero_effects_return_the_prior():
    p = brd_probability(herd(sick="C07"), cfg=CFG, n_boot=10, seed=0,
                        effects=dict(appetite=0.0, depression=0.0, hydration=0.0))
    assert np.allclose(p.p_brd, 0.10)


def test_prior_is_monotonic():
    lo = brd_probability(herd(seed=1), cfg=CFG, n_boot=10, seed=0, prior=0.05).set_index("cow_id").p_brd
    hi = brd_probability(herd(seed=1), cfg=CFG, n_boot=10, seed=0, prior=0.20).set_index("cow_id").p_brd
    assert (hi > lo).all()


def test_columns_intervals_labels_determinism():
    p = brd_probability(herd(sick="C02"), cfg=CFG, n_boot=40, seed=3)
    assert list(p.columns) == ["cow_id", "p_brd", "p_low", "p_high", "prior", "appetite_contrib", "depression_contrib",
                               "hydration_contrib", "basis", "calibrated", "assumptions_version"]
    assert ((p.p_low <= p.p_brd) & (p.p_brd <= p.p_high)).all() and (p.p_high > p.p_low).all()
    assert set(p.basis) == {"assumed"} and not p.calibrated.any() and set(p.assumptions_version) == {"test"}
    assert p.p_brd.is_monotonic_decreasing
    pd.testing.assert_frame_equal(p, brd_probability(herd(sick="C02"), cfg=CFG, n_boot=40, seed=3))
```

**Reference check (orchestrator, 2026-11-14), numpy re-implementation:**
- null herds give p of 0.05–0.19 against a 0.10 prior;
- the injected cow reaches 0.83, against a median of 0.09 for the others;
- raising the prior raises every p.

## Done when
`python scripts/verify_task.py T32` passes. The report quotes the wording used.

## Do not
- Present `p_brd` without the prior and the "assumed / not calibrated" label.
- Fit effects on the same cows you then report, unless real outcome labels exist.
