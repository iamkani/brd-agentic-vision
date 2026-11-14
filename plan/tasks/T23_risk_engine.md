# T23: Risk engine (score, uncertainty, reasons)

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M70 | P0 | Sonnet (orchestrator reviews the weights and wording) | 90 min | T22 |

- **Owns:** `cattle/analytics/risk.py`.
- **Read:**
  - `02_contracts.md` §8.13, §8.14, §9 (risk.yaml), §13;
  - `03_pitfalls.md` rows 36–40.
- **Fixtures:** none (synthetic herds generated in the test).

## Goal
Rank the 10 cows by a **relative behavioural risk index** (0–1) and explain each rank in terms a veterinarian recognises, with numbers. The index is not a probability of BRD: there are no health labels, and some or all cows may be healthy. Report rank uncertainty, and say plainly when no cow stands out. L2 is worth 20 points and breaks ties, so be conservative and honest.

## Method (implement exactly; weights come from `configs/risk.yaml`)
1. **Daily values.** For each cow-day, sum the `*_min`, `*_bouts`, `bunk_visits` and `drinking_events` features over hours. Take coverage-weighted means for `nn_dist_median` and `isolation_frac`. Cow-day coverage = mean hourly coverage.
2. **Herd-relative deviation on the latest day (day 2).** For each feature f:
   - `rel_f = value / herd_median − 1` (a percent change against the pen-mates' median);
   - `z_f = (value − herd_median) / (1.4826 × herd_MAD + ε)`, with ε = 1e-6 × herd_median + 1e-9.
3. **Change against own day 1.** `chg_f = day2 / day1 − 1`. Combine it as `z_f ← z_f + 0.5 × z(chg_f)`, where `z(chg_f)` is the robust z of the change against the herd's changes.
4. **Domains.** `domain_score = Σ weight_f × z_f` over the features listed in that domain. The signs in `risk.yaml` make "worse" positive (for example appetite: feeding_min −1.0). Then `composite = Σ domain_scores`.
5. **Scale to 0–1.** `risk_score = 1 / (1 + exp(−(composite − 3) / 3))`. **Do not round inside `score()`**: rounding to 3 decimals saturates strong cases at 1.000 and breaks the ordering. Round only when writing the CSV. Rank by `risk_score` descending; break ties by cow_id.
6. **Uncertainty.**
   - Bootstrap `n_boot` times: resample hours with replacement, **using the same hour indices for all cows**, and recompute the ranks.
   - `rank_low` / `rank_high` = the 5th / 95th percentile of ranks, clipped so `rank_low ≤ rank ≤ rank_high`.
7. **Flags.**
   - `low_coverage` if day-2 coverage < `min_coverage`.
   - `no_cow_stands_out` on **all** rows if no cow has both a composite ≥ `z_alert` **and** at least one feature with |rel_f| ≥ `min_effect` in the "worse" direction.
8. **Reasons.** Take the top 3 features by contribution (weight × z, worse direction only). Phrase each as `"<feature words> <+/-NN%> vs herd (<vet term>)"`, for example `feeding time -34% vs herd (reduced appetite)`. Vet terms:

   | Feature | Vet term |
   |---|---|
   | feeding_min, feeding_bouts, bunk_visits | reduced appetite / bunk attendance |
   | lying_min | increased recumbency |
   | walking_min | reduced activity |
   | isolation_frac, nn_dist_median | social isolation |
   | drinking_events | reduced water intake |

   Join the reasons with `"; "`. If nothing is in the worse direction: `"no behaviour below herd norms"`.

- **Output:** contracts §8.14 columns. Leave `evidence_clips` as `""`; T24 fills them. `coverage` = day-2 coverage.
- **Also export `domain_scores(hourly, cfg=None) -> DataFrame`** with columns `cow_id, appetite, depression, hydration, composite, coverage`. These are the per-domain sums from steps 1–4, worse direction positive. T32's probability model uses it, so keep it consistent with `score()`.

## Locked test
```python
# LOCKED TEST: tests/unit/test_risk.py
import numpy as np
import pandas as pd

from cattle.analytics.risk import score
from cattle.validate.schemas import validate

COWS = [f"C{i:02d}" for i in range(1, 11)]
BASE = dict(feeding_min=12.0, feeding_bouts=1.5, bunk_visits=2.0, lying_min=30.0, lying_bouts=1.0, standing_min=12.0,
            walking_min=3.0, drinking_events=0.3, nn_dist_median=0.12, isolation_frac=0.15, visible_min=58.0)


def herd(seed=0, sick=None, feed_mult=0.65, lying_mult=1.25, iso_mult=2.0, coverage=0.95):
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
                    rows.append(dict(cow_id=cow, day=day, hour=hour, feature=f, value=v, coverage=coverage))
    return pd.DataFrame(rows)


def test_injected_cow_ranks_first_with_tight_range():
    r = score(herd(sick="C07"), n_boot=200, seed=0).set_index("cow_id")
    assert r.loc["C07", "rank"] == 1 and r.loc["C07", "rank_high"] <= 2
    assert "no_cow_stands_out" not in str(r.loc["C07", "flags"])


def test_injection_robust_across_seeds():
    hits = sum(int(score(herd(seed=s, sick="C03"), n_boot=30, seed=s).set_index("cow_id").loc["C03", "rank"] == 1)
               for s in range(10))
    assert hits >= 9


def test_null_herd_says_no_cow_stands_out():
    r = score(herd(seed=4), n_boot=50, seed=0)
    assert r["flags"].fillna("").str.contains("no_cow_stands_out").all()


def test_schema_and_rank_ranges():
    r = score(herd(sick="C02"), n_boot=50, seed=0)
    assert validate("risk", r, require_clips=False) == []
    assert ((r.rank_low <= r["rank"]) & (r["rank"] <= r.rank_high)).all()


def test_reasons_have_numbers_and_vet_terms():
    text = score(herd(sick="C07"), n_boot=30, seed=0).set_index("cow_id").loc["C07", "top_reasons"].lower()
    assert "feeding" in text and "%" in text
    assert any(term in text for term in ["appetite", "recumbency", "isolation"])


def test_monotonic_in_feeding():
    a = score(herd(seed=1, sick="C05", feed_mult=0.9, lying_mult=1.0, iso_mult=1.0), n_boot=20, seed=0)
    b = score(herd(seed=1, sick="C05", feed_mult=0.6, lying_mult=1.0, iso_mult=1.0), n_boot=20, seed=0)
    assert b.set_index("cow_id").loc["C05", "risk_score"] > a.set_index("cow_id").loc["C05", "risk_score"]


def test_low_coverage_flag():
    h = herd(seed=2)
    h.loc[h.cow_id == "C09", "coverage"] = 0.3
    r = score(h, n_boot=20, seed=0).set_index("cow_id")
    assert "low_coverage" in r.loc["C09", "flags"]
```

## Reference check (orchestrator, 2026-11-14)
A numpy-only reference implementation of this exact method gave:
- the injected cow at rank 1 with range 1–1;
- 10 of 10 robustness seeds at rank 1;
- the null herd with no cow standing out (its maximum composite was 4.1, but no feature had a ≥ 15% effect);
- monotonicity: feed ×0.9 gives 0.721, feed ×0.6 gives 0.99999.

**The `min_effect` condition is what keeps the null test passing.** Do not drop it.

## Done when
- `python scripts/verify_task.py T23` passes.
- The report shows `top_reasons` for the injected cow and the flags for the null herd.
- The orchestrator reviews the wording before L2 runs.

## Do not
- Use the words "probability", "diagnosis" or "infected" in the risk-index outputs (`brd_risk.csv`, reasons, cards). P(BRD) belongs to T32, with its assumptions label.
- Rank on raw totals without herd-relative scaling.
