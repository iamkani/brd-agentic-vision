# T24: Risk cards with clips

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M80 | P0 | Haiku | 60 min | T17, T23 |

- **Owns:** `cattle/analytics/cards.py`, `cattle/analytics/templates/card.html.j2`.
- **Read:** `02_contracts.md` §8.10, §8.14; `03_pitfalls.md` rows 36 and 39.
- **Fixtures:** none (clips generated with ffmpeg `testsrc` in the test).

## Goal
The brief asks for "a one-page risk card for each of your top 3 cows, linking 2–3 video clips as evidence". A card is what the veterinarian on the panel reads. Make it clear, honest, printable on one page, and fully offline.

## Specification
`render_cards(risk, budgets, outputs_dir, top_n=3) -> list[Path]` writes `outputs_dir/risk_cards/<cow_id>.html` for ranks 1..`top_n`, returning the paths in rank order. Each card contains:
- **Header:** cow ID, rank with its range (`rank 1, range 1–2`), risk index 0–1, the coverage, and any flags (shown as badges).
- **"Why this cow":** the `top_reasons`, one bullet each.
- **Evidence:** one `<video controls preload="metadata">` per clip in `evidence_clips` (2–3). The `src` is relative to the card (`../clips/<file>.mp4`), with a caption.
- **Time budget:** an inline SVG bar chart of minutes per behaviour, day 1 against day 2. No external chart library.
- **Footer, mandatory text:**
  > Relative behavioural risk index from video, not a diagnosis. Findings require veterinary review before any treatment decision. Rectal temperature and clinical exam are not available from video.
- **Print layout:** a `@media print` style that fits one A4/Letter page.
- **Self-contained:** no external URLs (fonts, CSS or JS).

## Locked test
```python
# LOCKED TEST: tests/unit/test_cards.py
import subprocess

import pandas as pd

from cattle.analytics.cards import render_cards


def make_clip(path):
    subprocess.run(["ffmpeg", "-v", "error", "-f", "lavfi", "-i", "testsrc=size=320x240:rate=10", "-t", "1",
                    "-pix_fmt", "yuv420p", "-y", str(path)], check=True)


def test_cards_for_top_three(tmp_path):
    out = tmp_path / "outputs"
    clips = out / "clips"
    clips.mkdir(parents=True)
    rows = []
    for i in range(10):
        cow = f"C{i + 1:02d}"
        ev = []
        if i < 3:
            for k in (1, 2):
                make_clip(clips / f"{cow}_{k}.mp4")
                ev.append(f"clips/{cow}_{k}.mp4")
        rows.append(dict(cow_id=cow, risk_score=round(1 - i / 10, 3), rank=i + 1, rank_low=i + 1, rank_high=i + 1,
                         top_reasons="feeding time -34% vs herd (reduced appetite); lying time +25% vs herd (increased recumbency)",
                         evidence_clips=";".join(ev), coverage=0.9, flags=""))
    budgets = pd.DataFrame([dict(cow_id=f"C{i + 1:02d}", day=d, behavior=b, minutes=m, fraction=m / 1440,
                                 minutes_observed=m, minutes_estimated=0, minutes_assumed=0)
                            for i in range(10) for d in (1, 2) for b, m in [("lying", 700), ("feeding", 200), ("standing", 540)]])
    paths = render_cards(pd.DataFrame(rows), budgets, outputs_dir=out, top_n=3)
    assert [p.name for p in paths] == ["C01.html", "C02.html", "C03.html"]
    html = paths[0].read_text()
    assert html.count("<video") == 2 and "clips/C01_1.mp4" in html
    assert "veterinary review" in html.lower() and "reduced appetite" in html and "<svg" in html
    assert "@media print" in html and "http://" not in html and "https://" not in html
```

## Done when
`python scripts/verify_task.py T24` passes.
