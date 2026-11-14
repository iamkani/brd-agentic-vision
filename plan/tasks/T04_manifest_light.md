# T04: Manifest scanner and light probe

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M10 | P0 | Haiku | 60 min | T03 |

- **Owns:** `cattle/io/manifest.py`, `cattle/quality/light.py`, `scripts/build_manifest.py`.
- **Read:**
  - `02_contracts.md` §1, §2, §8.1, and the blocks for `cattle/quality/light.py` and `cattle/io/manifest.py` in §10;
  - `cattle/io/avi.py` (its public functions only).
- **Fixtures:** `night_head.avi`, `day_head.avi`.

## Goal
Know every file before processing: its day, camera, time, size, MD5, clock channel, frame and keyframe counts, and whether it is a colour (day) or grey infrared (night) recording. This answers "which files are day or night" without anyone opening a video.

## Specification

### `cattle/io/manifest.py`
- **`parse_name(stem)`** returns `(datetime, suffix)` for `Event` + 14 digits (YYYYMMDDhhmmss) + 3-digit suffix. Anything else raises `ValueError`, including names starting with `._`.
- **`scan(raw_root)`:**
  1. Walk only `raw_root/day<d>/cam0<n>/`. Accept files matching `^Event\d{17}\.avi$` case-insensitively, and skip names starting with `._`.
  2. Return manifest rows. Columns not known yet get `md5 = ""`, `channel = -1`, `clock_start = NaT`, `n_frames = n_keyframes = -1`, `picture = ""`, and `satavg = yavg = NaN`.
  3. Raise `ValueError` if a file's suffix does not match its camera folder (`CAM_SUFFIX`).
- **`enrich(df, raw_root, workers=4, light=True)`:** fill in, per file and in parallel with a process pool:
  - `channel` (from the header record);
  - `clock_start` (timestamp of frame 1);
  - `n_frames` and `n_keyframes` (from `avi.index`);
  - `picture`, `satavg`, `yavg` (from `light.probe_file`, when `light` is true).

  A file that fails gets `picture = "decode_failed"` and is logged; the run does not crash.

### `cattle/quality/light.py`
- **`probe_file(path)`:** decode the **first keyframe** payload (`avi.read_payloads`) with `ffmpeg -f hevc -i pipe:0 -frames:v 1 -vf signalstats,metadata=print:file=- -f null -`. Parse the first `SATAVG` and `YAVG`. `picture = "grey_IR"` if `satavg <= GREY_SAT_MAX` (1.0), else `"colour"`.
- **`classify_rgb(img)`:** pure numpy.
  - `satavg = mean(max(R,G,B) - min(R,G,B))`;
  - `yavg = mean(0.299R + 0.587G + 0.114B)`;
  - the same threshold.

### `scripts/build_manifest.py`
- **Flags:** `--raw-root`, `--work-root` (defaults from config), `--drive-manifest PATH` (a CSV with columns `file` and `md5`, as supplied privately by the organizers), `--day {1,2}` (optional filter), `--workers 4`, `--no-light`.
- **Output:** writes `<work_root>/manifest.parquet` and prints a 6-line summary: files per folder, then counts per `picture`.

## Locked test
```python
# LOCKED TEST: tests/unit/test_manifest_light.py
import shutil
from datetime import datetime

import numpy as np
import pytest

from cattle.io.manifest import enrich, parse_name, scan
from cattle.quality.light import classify_rgb, probe_file

NIGHT = "tests/fixtures_private/night_head.avi"
DAY = "tests/fixtures_private/day_head.avi"


def test_parse_name():
    assert parse_name("Event20000110235550006") == (datetime(2000, 1, 10, 23, 55, 50), "006")
    assert parse_name("Event20000111090451017") == (datetime(2000, 1, 11, 9, 4, 51), "017")
    with pytest.raises(ValueError):
        parse_name("._Event20000110235550006")
    with pytest.raises(ValueError):
        parse_name("Event2000011023555")


def test_scan_skips_appledouble_and_reads_layout(tmp_path):
    for rel in ["day1/cam01/Event20000110235550006.Avi", "day1/cam01/._Event20000110235550006.Avi",
                "day2/cam03/Event20000111000039003.avi", "day2/cam03/notes.txt",
                "day1/cam02/Event20000110000427017.AVI"]:
        p = tmp_path / rel
        p.parent.mkdir(parents=True, exist_ok=True)
        p.write_bytes(b"x" * 10)
    df = scan(tmp_path)
    assert sorted(df.file_id) == ["Event20000110000427017", "Event20000110235550006", "Event20000111000039003"]
    row = df.set_index("file_id").loc["Event20000111000039003"]
    assert row.folder_day == 2 and row.cam == "cam03"
    assert row.rel_path == "day2/cam03/Event20000111000039003.avi"
    assert row.size_bytes == 10 and row.name_start == datetime(2000, 1, 11, 0, 0, 39)


def test_scan_rejects_suffix_camera_mismatch(tmp_path):
    p = tmp_path / "day1/cam01/Event20000110000427017.Avi"
    p.parent.mkdir(parents=True)
    p.write_bytes(b"x")
    with pytest.raises(ValueError):
        scan(tmp_path)


def test_probe_file_night_and_day():
    n = probe_file(NIGHT)
    assert n.picture == "grey_IR" and n.satavg <= 0.5 and 95 <= n.yavg <= 108
    d = probe_file(DAY)
    assert d.picture == "colour" and 4.5 <= d.satavg <= 6.5 and 115 <= d.yavg <= 130


def test_classify_rgb():
    grey = np.full((30, 40, 3), 90, np.uint8)
    assert classify_rgb(grey).picture == "grey_IR"
    colour = grey.copy()
    colour[..., 0] = 120
    colour[..., 2] = 60
    c = classify_rgb(colour)
    assert c.picture == "colour" and c.satavg > 1.0 and 85 <= c.yavg <= 100


def test_enrich_on_fixture_layout(tmp_path):
    for src, name in [(NIGHT, "Event20000110235550006.Avi"), (DAY, "Event20000110090451006.Avi")]:
        dst = tmp_path / "day1" / "cam01" / name
        dst.parent.mkdir(parents=True, exist_ok=True)
        shutil.copy(src, dst)
    df = enrich(scan(tmp_path), tmp_path, workers=2)
    for col in ["file_id", "folder_day", "cam", "rel_path", "name_start", "size_bytes", "md5", "channel",
                "clock_start", "n_frames", "n_keyframes", "picture", "satavg", "yavg"]:
        assert col in df.columns, col
    df = df.set_index("file_id")
    night = df.loc["Event20000110235550006"]
    assert night.channel == 1 and night.n_frames == 515 and night.n_keyframes == 10
    assert night.picture == "grey_IR" and night.clock_start == datetime(2000, 1, 10, 23, 55, 50, 197000)
    day = df.loc["Event20000110090451006"]
    assert day.n_keyframes == 8 and day.picture == "colour"
```

## Done when
- `python scripts/verify_task.py T04` passes.
- The report includes the output of `python -m scripts.build_manifest --help`.

## Do not
- Decode more than the first keyframe per file for the light probe.
- Read `raw_root` in tests. Tests use `tmp_path` and the fixtures.
