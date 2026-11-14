# T01: Private fixtures from real files

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M10 | P0 | Haiku | 40 min | T00 |

- **Owns:** `scripts/make_fixtures.py`; `tests/fixtures_private/*` (local only, git-ignored).
- **Read:** `02_contracts.md` §1 and §2; `04_testing.md` §4.
- **Runs on:** the Mac with the external SSD mounted. This is the **only** worker task allowed to read `raw_root`, and only to read.

## Goal
Create small, exact fixtures that every other task can test against without the full dataset. Record their facts so tests can assert real values.

## Fixtures to create
Each fixture is the first **6,000,000 bytes** of its source. Skip names starting with `._`. Obtain the filename mapping and independent reference SHA-256 values from the private fixture manifest before extracting files. The clock assertions below use an example recording date. The real-data verifier uses the matching test text and reference values from the authorized private plan.

| Fixture | Source (relative to `raw_root`) |
|---|---|
| `night_head.avi` | Approved Cam 01 night clip |
| `day_head.avi` | Approved Cam 01 day clip |
| `cam02_day_head.avi` | Approved Cam 02 day clip |
| `cam02_night_head.avi` | Approved Cam 02 night clip |
| `cam03_day_head.avi` | Approved Cam 03 day clip |
| `cam03_night_head.avi` | Approved Cam 03 night clip |

`scripts/make_fixtures.py` takes `--raw-root` (default from `cattle.config.paths()`) and `--out` (default `tests/fixtures_private`). It writes the six files and `FIXTURES.json`. Each fixture has one JSON object with these keys:
- `source`, `sha256`, `size`;
- `n_frames` (complete video chunks), `keyframes` (list of frame indices);
- `header` (clock text of frame 0), `first_clock` (frame 1), `channel` (int after `#` in the header);
- `width`, `height`, `fourcc`;
- `satavg_first_key`, `yavg_first_key`.

## How to compute the facts
Do not import `cattle.io.avi`: T03 may not exist yet. Use this reference walker, which was verified on the real files:

```python
import struct

def walk(data: bytes):
    """Return a list of video frames: dict(frame_idx, offset, size, is_key, clock)."""
    assert data[2044:2048] == b"movi"
    pos, frames = 2048, []
    while pos + 8 <= len(data):
        cid = data[pos:pos + 4].decode("latin1")
        size = struct.unpack("<I", data[pos + 4:pos + 8])[0]
        if pos + 8 + size > len(data):
            break                                   # incomplete trailing chunk
        payload = data[pos + 8:pos + 8 + size]
        if cid[:2] == "00":                          # video chunk (00db key / 00dc delta)
            head, i, types = payload[:2048], 0, []
            while True:
                j = head.find(b"\x00\x00\x01", i)
                if j < 0 or j + 3 >= len(head):
                    break
                types.append((head[j + 3] >> 1) & 0x3F)   # HEVC NAL unit type
                i = j + 3
            frames.append(dict(frame_idx=len(frames), offset=pos + 8, size=size,
                               is_key=any(16 <= t <= 21 for t in types), clock=None))
        elif cid[:2] == "01" and frames:             # clock text belongs to the previous video chunk
            frames[-1]["clock"] = payload.rstrip(b"\x00").decode("latin1")
        pos += 8 + size + (size & 1)
    return frames
```

- **Width, height and fourcc:** read the `avih` chunk in the header. Width and height are little-endian uint32 values at byte offsets 64 and 68 of the file. The fourcc is 4 ASCII bytes at offset 188.
- **SATAVG and YAVG:** pipe the first keyframe's payload to `ffmpeg -v error -f hevc -i pipe:0 -frames:v 1 -vf signalstats,metadata=print:file=- -f null -` and parse the first `lavfi.signalstats.SATAVG=` and `YAVG=` values.

## Locked test
```python
# LOCKED TEST: tests/unit/test_fixtures.py
import hashlib
import json
import pathlib

FIX = pathlib.Path("tests/fixtures_private")

ALL = ["night_head.avi", "day_head.avi", "cam02_day_head.avi", "cam02_night_head.avi",
       "cam03_day_head.avi", "cam03_night_head.avi"]


def test_golden_fixture_bytes():
    golden = json.loads((FIX / "GOLDEN_SHA256.json").read_text())
    assert all(name in golden for name in ("night_head.avi", "day_head.avi"))
    for name in ("night_head.avi", "day_head.avi"):
        digest = golden[name]
        p = FIX / name
        assert p.stat().st_size == 6_000_000, name
        assert hashlib.sha256(p.read_bytes()).hexdigest() == digest, name


def test_all_fixtures_present():
    for name in ALL:
        p = FIX / name
        assert p.exists() and p.stat().st_size == 6_000_000, name


def test_facts_file():
    facts = json.loads((FIX / "FIXTURES.json").read_text())
    for name in ALL:
        for key in ["source", "sha256", "size", "n_frames", "keyframes", "header", "first_clock", "channel",
                    "width", "height", "fourcc", "satavg_first_key", "yavg_first_key"]:
            assert key in facts[name], (name, key)
    n = facts["night_head.avi"]
    assert n["n_frames"] == 515 and n["keyframes"] == [0, 19, 79, 139, 199, 259, 319, 379, 439, 499]
    assert n["channel"] == 1 and n["first_clock"] == "#1-23:55:50.197" and n["fourcc"] == "GHEC"
    d = facts["day_head.avi"]
    assert d["n_frames"] == 439 and d["header"] == "Event#1-9:4:51(2000/1/10)"
    assert d["satavg_first_key"] > 4.0 and n["satavg_first_key"] < 0.5
    for cam in ("cam02", "cam03"):
        assert facts[f"{cam}_day_head.avi"]["satavg_first_key"] > 1.0, cam
        assert facts[f"{cam}_night_head.avi"]["satavg_first_key"] < 1.0, cam


def test_no_appledouble_stubs():
    assert not list(FIX.glob("._*"))
```

## Done when
- `python scripts/verify_task.py T01` passes.
- The report lists the camera channels (`#2`? `#3`?) and any surprises: a fixture that is not grey at 01:00, or a different keyframe pattern. The orchestrator copies those facts into `02_contracts.md` §1.

## Do not
- Commit anything in `tests/fixtures_private/`. It is event data.
- Copy whole files, or more than these six heads.
