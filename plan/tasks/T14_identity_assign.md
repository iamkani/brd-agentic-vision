# T14: Closed-set identity assignment

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M40 | P0 | Sonnet (orchestrator reviews) | 90 min | T13 |

- **Owns:** `cattle/identity/assign.py`, `cattle/identity/embed.py`.
- **Read:**
  - `02_contracts.md` §4, §7, §8.6, and the identity block of §10;
  - `03_pitfalls.md` rows 19–25;
  - `reference/cow_reid.py`, a prior script the orchestrator copies into the worktree. Port its good parts and fix the three bugs below.
- **Fixtures:** none (synthetic scores and embeddings).

## Goal
Give every global tracklet (gtrack) one of the 10 cow IDs, or `UNK`, with a confidence, a basis and a method. The set is **closed** (exactly 10 cows), and tracks that overlap in time can never be the same cow. Tracks that do **not** overlap can be the same cow.

## Bugs found in `cow_reid.py` that must not come back
1. **Fragments forced apart.** It treated every track in a 30-s window as simultaneous. When one cow's track broke in two while another cow was hidden, the second fragment got the hidden cow's ID. Apply the one-to-one constraint only to tracks whose time spans overlap.
2. **DINOv2 centre-crop.** The default processor resizes the short side to 256 and centre-crops 224, which cuts the head (ear tag) and rump from wide crops. Use `pad_to_square` before embedding.
3. **Gallery collapse.** Adding video crops averaged all of them into one prototype. Keep a prototype per (cow, camera, lighting).

## Specification
`cattle/identity/assign.py`:

`assign(segments, scores, cows=COWS, min_score=0.5, min_margin=0.05, slice_s=10.0) -> DataFrame` with columns exactly `gtrack_id, t_start, t_end, cow_id, confidence, basis, method, margin`.
- **Inputs.** `segments` has columns `gtrack_id, t_start, t_end`. `scores` has columns `gtrack_id, cow_id, score` (0–1) and `source` (`appearance`, `tag` or `manual`). Several rows per pair are allowed; take the max per (gtrack, cow, source).
- **Overrides.** A `tag` or `manual` score ≥ 0.9 fixes that gtrack's cow, with `method` = the source, `basis="observed"` and `confidence = 1.0`.
- **Greedy with conflicts.** Take every remaining (gtrack, cow) pair sorted by score descending. Assign a pair if the gtrack is still unassigned and no gtrack **overlapping in time** already holds that cow. `slice_s` only bounds the search; overlap decides conflicts.
- **Thresholds.**
  - If a gtrack's best feasible score is below `min_score`, it becomes `UNK` with `confidence = 0.0`, `basis="estimated"` and `method="appearance"`.
  - `margin` = best feasible score − second-best feasible score.
  - `confidence` = best score, halved if `margin < min_margin`.
  - Appearance assignments get `basis="estimated"` and `method="appearance"`.
- **Optional (not tested):** `propagate_continuity(identity, gtracks, max_gap_s=6, max_dist=0.05)` labels an `UNK` gtrack that continues an identified one, with `method="continuity"` and `basis="assumed"`.

`cattle/identity/embed.py`:
- `pad_to_square(img, fill=0)` centres the image on a square canvas whose side is the longer image side.
- `build_prototypes(emb)` takes columns `cow_id, cam, lighting, vector` and returns one L2-normalised mean vector per (cow_id, cam, lighting), with columns `cow_id, cam, lighting, vector, n`.
- `embed_crops(images, model="facebook/dinov2-small", device=None)` is lazy-imported and used on the GPU workstation. It applies `pad_to_square`, uses the CLS token concatenated with the mean patch token, and L2-normalises. It is not unit-tested here.
- `similarity_scores(track_vectors, prototypes)` gives each gtrack, per cow, the mean of the top-3 cosine similarities, rescaled to 0–1. The output follows the `scores` schema with `source="appearance"`.

## Locked test
```python
# LOCKED TEST: tests/unit/test_assign.py
import numpy as np
import pandas as pd
import pytest

from cattle.identity.assign import assign
from cattle.identity.embed import build_prototypes, pad_to_square

T0 = pd.Timestamp("2000-01-10 10:00:00")
COWS = [f"C{i:02d}" for i in range(1, 11)]


def seg(gid, a, b):
    return dict(gtrack_id=gid, t_start=T0 + pd.Timedelta(seconds=a), t_end=T0 + pd.Timedelta(seconds=b))


def scores_for(gid, best, own=0.9, other=0.62, source="appearance"):
    return [dict(gtrack_id=gid, cow_id=c, score=(own if c == best else other), source=source) for c in COWS]


def test_fragments_of_one_cow_share_id_while_another_cow_is_hidden():
    segs = [seg(f"g{i:06d}", 0, 30) for i in range(1, 9)] + [seg("gA", 0, 12), seg("gB", 15, 30)]
    sc = sum((scores_for(f"g{i:06d}", COWS[i]) for i in range(1, 9)), [])
    sc += scores_for("gA", "C01") + scores_for("gB", "C01")
    out = assign(pd.DataFrame(segs), pd.DataFrame(sc)).set_index("gtrack_id")
    assert out.loc["gA", "cow_id"] == "C01" and out.loc["gB", "cow_id"] == "C01"
    assert "C10" not in set(out.cow_id)
    assert out.loc["gA", "method"] == "appearance" and out.loc["gA", "basis"] == "estimated"
    assert out.loc["gA", "confidence"] == pytest.approx(0.9)


def test_overlapping_tracks_cannot_share_a_cow():
    sc = scores_for("gX", "C01", own=0.9) + scores_for("gY", "C01", own=0.85)
    sc = [dict(r, score=0.8) if (r["gtrack_id"] == "gY" and r["cow_id"] == "C02") else r for r in sc]
    out = assign(pd.DataFrame([seg("gX", 0, 20), seg("gY", 5, 25)]), pd.DataFrame(sc)).set_index("gtrack_id")
    assert out.loc["gX", "cow_id"] == "C01" and out.loc["gY", "cow_id"] == "C02"


def test_low_score_is_unknown():
    row = assign(pd.DataFrame([seg("gZ", 0, 10)]), pd.DataFrame(scores_for("gZ", "C03", own=0.45, other=0.3))).iloc[0]
    assert row.cow_id == "UNK" and row.confidence == 0.0


def test_tag_overrides_appearance():
    sc = scores_for("gT", "C03", own=0.95) + [dict(gtrack_id="gT", cow_id="C05", score=1.0, source="tag")]
    row = assign(pd.DataFrame([seg("gT", 0, 10)]), pd.DataFrame(sc)).iloc[0]
    assert row.cow_id == "C05" and row.method == "tag" and row.basis == "observed" and row.confidence == 1.0


def test_small_margin_halves_confidence():
    sc = scores_for("gM", "C04", own=0.80, other=0.60)
    sc = [dict(r, score=0.78) if r["cow_id"] == "C06" else r for r in sc]
    row = assign(pd.DataFrame([seg("gM", 0, 10)]), pd.DataFrame(sc)).iloc[0]
    assert row.cow_id == "C04" and row.margin == pytest.approx(0.02, abs=1e-6)
    assert row.confidence == pytest.approx(0.40)


def test_output_columns():
    out = assign(pd.DataFrame([seg("gQ", 0, 10)]), pd.DataFrame(scores_for("gQ", "C07")))
    assert list(out.columns) == ["gtrack_id", "t_start", "t_end", "cow_id", "confidence", "basis", "method", "margin"]


def test_pad_to_square_keeps_whole_image():
    img = (np.arange(100 * 200 * 3) % 251).astype(np.uint8).reshape(100, 200, 3)
    sq = pad_to_square(img)
    assert sq.shape == (200, 200, 3) and (sq[50:150] == img).all()


def test_prototypes_not_collapsed():
    rng = np.random.default_rng(0)
    emb = pd.DataFrame([dict(cow_id="C01", cam=cam, lighting=light, vector=rng.normal(size=8))
                        for cam, light in [("cam01", "day")] * 5 + [("cam02", "night")] * 5 + [("cam02", "day")] * 3])
    protos = build_prototypes(emb)
    assert len(protos[protos.cow_id == "C01"]) == 3
    assert np.allclose(np.linalg.norm(np.stack(protos.vector), axis=1), 1.0)
```

## Done when
- `python scripts/verify_task.py T14` passes.
- The report states how `similarity_scores` rescales cosine similarity to 0–1.

## Do not
- Import torch or transformers at module import time.
- Force a cow onto every gtrack.
