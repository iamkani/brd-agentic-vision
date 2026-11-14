# T15: Labelling sheets (HTML + CSV templates)

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M40 | P0 | Haiku | 45 min | T05 |

- **Owns:** `cattle/tools/sheets.py`, `scripts/make_sheet.py`.
- **Read:** `02_contracts.md` §8.16; `tasks/HUMAN.md` (H03–H06), to see how people will use the sheets.
- **Fixtures:** none (synthetic images).

## Goal
Human labelling is on the critical path: seeds, the detection check, identity evaluation and the check set. Make it fast. Build a static HTML page of thumbnails or clips, numbered, with a matching CSV template to fill in. It needs no server, no network and no JavaScript frameworks.

## Specification
- **`make_sheet(items, out_html, title, answer_columns) -> (html_path, csv_path)`.**
  - `items` has columns `item_id` and `image`, a path relative to the HTML file. Optional columns are `clip` (relative `.mp4`) and `info` (free text).
  - The HTML shows a responsive grid of cards. Each card holds the item_id, the image (`<img loading="lazy">`) or clip (`<video controls muted>`), and the info text, **HTML-escaped**.
  - The CSV template sits next to the HTML as `<stem>_answers.csv`. Its columns are `item_id` plus `answer_columns`, with one row per item and empty answers.
  - The page includes a short instructions box at the top. Pass it as `instructions=` (optional argument).
- **`scripts/make_sheet.py`** has these modes:
  - **`--seeds --keyframes P --tracks P --hours 10`:** one image per track (the frame with the largest box, cropped with 10% padding and saved under `work/sheets/seeds/`). Answer columns: `cow_id, notes`.
  - **`--det-check --keyframes P --n 100`:** keyframes stratified by camera × picture. Answer columns: `n_cows_true, notes`.
  - **`--id-eval --gtracks P --identity P --exclude-hours 9,10,11,12 --n 120`.** Answer columns: `cow_id`.
  - **`--check-set --samples P --n-random 180 --n-targeted 120`:** 10-s clips (via `cattle.io.clips`, imported lazily) for stratified and targeted samples. Answer columns: `behavior, sure, notes`.

  Every mode prints the paths of the HTML and CSV it wrote.

## Locked test
```python
# LOCKED TEST: tests/unit/test_sheets.py
import csv

import numpy as np
import pandas as pd
from PIL import Image

from cattle.tools.sheets import make_sheet


def test_sheet_html_and_csv(tmp_path):
    imgs = []
    for i in range(5):
        p = tmp_path / "img" / f"{i}.jpg"
        p.parent.mkdir(exist_ok=True)
        Image.fromarray(np.full((40, 60, 3), 30 * i, np.uint8)).save(p)
        imgs.append(f"img/{i}.jpg")
    items = pd.DataFrame(dict(item_id=[f"g{i:06d}" for i in range(5)], image=imgs, info=["cam01 10:00 <b>"] * 5))
    html, tmpl = make_sheet(items, tmp_path / "sheet.html", title="Seeds", answer_columns=["cow_id", "notes"])
    text = html.read_text()
    assert text.count("<img") == 5 and "img/3.jpg" in text
    assert "&lt;b&gt;" in text and "Seeds" in text
    assert "http://" not in text and "https://" not in text
    rows = list(csv.DictReader(tmpl.open()))
    assert [r["item_id"] for r in rows] == list(items.item_id)
    assert list(rows[0].keys()) == ["item_id", "cow_id", "notes"]
```

## Done when
`python scripts/verify_task.py T15` passes.
