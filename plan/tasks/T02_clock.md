# T02: Clock-text parser

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M10 | P0 | Haiku | 40 min | T00 |

- **Owns:** `cattle/io/clock.py`.
- **Read:**
  - `02_contracts.md` §1 (rows "Clock text" and "Clock channel"), §5, and the `cattle/io/clock.py` block of §10;
  - `04_testing.md` §4.
- **Fixtures:** none. This is pure string handling.

## Goal
Turn the recorder's per-frame clock text into exact timestamps, including the header record, unpadded fields and midnight rollover.

## Facts (verified on real files)
- **Header record** (frame 0): `Event#<ch>-<H>:<M>:<S>(<YYYY>/<M>/<D>)`. Examples: `Event#1-23:55:50(2000/1/10)`, `Event#1-9:4:51(2000/1/10)`.
- **Frame records** (frame ≥ 1): `#<ch>-<H>:<M>:<S>.<mmm>`. Examples: `#1-23:55:50.197`, `#1-9:4:52.017`, `#1-0:0:50.307`, `#1-23:56:7.297`.
- **Padding:** H, M and S are never zero-padded. Milliseconds are always three digits.
- **Trailing NUL:** records may end with `\x00`; strip it.
- **Midnight:** a file can cross midnight. The day advances when a stamp is more than 12 hours earlier than the previous stamp.

## Specification
```python
@dataclass(frozen=True)
class ClockRecord:
    channel: int
    seconds: float          # seconds since midnight (header: whole seconds)
    date: date | None       # only for the header
    is_header: bool

def parse_record(text: str) -> ClockRecord | None
    # None if the text matches neither pattern or a field is out of range (H > 23, M > 59, S > 59)

def to_timestamps(texts, start_date=None, fill_header=True) -> list[datetime | None]
    # texts: list of str or None, in frame order
```

**`to_timestamps` rules:**
1. The date comes from the first header record. If there is no header, use `start_date`. If there is neither, raise `ValueError`.
2. Frame records become `datetime(date) + seconds`. When a frame's seconds drop by more than 43,200 relative to the previous frame stamp, add one day from then on.
3. Unparseable entries and `None` become `None`. They do not reset the rollover logic.
4. **Header frame value:**
   - With `fill_header=True`, the header frame gets `first frame stamp − median step`. The median step is taken over consecutive frame stamps; use 1/30 s if fewer than two stamps exist.
   - With `fill_header=False`, the header frame gets `None`.

## Locked test
```python
# LOCKED TEST: tests/unit/test_clock.py
from datetime import date, datetime, timedelta

import pytest

from cattle.io.clock import ClockRecord, parse_record, to_timestamps


def test_header_record():
    r = parse_record("Event#1-23:55:50(2000/1/10)")
    assert isinstance(r, ClockRecord)
    assert r.is_header and r.channel == 1
    assert r.date == date(2000, 1, 10)
    assert r.seconds == pytest.approx(23 * 3600 + 55 * 60 + 50)


def test_header_unpadded():
    r = parse_record("Event#1-9:4:51(2000/1/10)")
    assert r.is_header and r.seconds == pytest.approx(9 * 3600 + 4 * 60 + 51)


def test_frame_record_padded_fields():
    r = parse_record("#1-23:55:50.197")
    assert not r.is_header and r.channel == 1 and r.date is None
    assert r.seconds == pytest.approx(86150.197)


def test_frame_record_unpadded_fields():
    assert parse_record("#1-9:4:52.017").seconds == pytest.approx(9 * 3600 + 4 * 60 + 52.017)
    assert parse_record("#1-0:0:50.307").seconds == pytest.approx(50.307)
    assert parse_record("#1-23:56:7.297").seconds == pytest.approx(23 * 3600 + 56 * 60 + 7.297)


def test_trailing_nul_and_other_channel():
    r = parse_record("#3-7:05:09.040\x00")
    assert r.channel == 3 and r.seconds == pytest.approx(7 * 3600 + 5 * 60 + 9.040)


@pytest.mark.parametrize("text", ["", "hello", "#x-1:2:3.004", "Event#1-25:00:00(2000/1/10)", "#1-10:61:00.000"])
def test_garbage_returns_none(text):
    assert parse_record(text) is None


def test_timestamps_header_fill():
    ts = to_timestamps(["Event#1-23:55:50(2000/1/10)", "#1-23:55:50.197", "#1-23:55:50.230", "#1-23:55:50.263"])
    assert ts[1] == datetime(2000, 1, 10, 23, 55, 50, 197000)
    assert ts[3] == datetime(2000, 1, 10, 23, 55, 50, 263000)
    assert abs((ts[1] - ts[0]).total_seconds() - 0.033) <= 0.002


def test_timestamps_no_fill():
    ts = to_timestamps(["Event#1-23:55:50(2000/1/10)", "#1-23:55:50.197"], fill_header=False)
    assert ts[0] is None and ts[1] == datetime(2000, 1, 10, 23, 55, 50, 197000)


def test_timestamps_midnight_rollover():
    ts = to_timestamps(["Event#1-23:59:59(2000/1/10)", "#1-23:59:59.934", "#1-23:59:59.967",
                        "#1-0:0:0.000", "#1-0:0:0.033"])
    assert ts[2] == datetime(2000, 1, 10, 23, 59, 59, 967000)
    assert ts[3] == datetime(2000, 1, 11, 0, 0, 0)
    assert ts[4] - ts[3] == timedelta(milliseconds=33)


def test_timestamps_with_start_date_and_no_header():
    ts = to_timestamps(["#2-10:00:00.000", "#2-10:00:00.033"], start_date=date(2000, 1, 11))
    assert ts == [datetime(2000, 1, 11, 10, 0, 0), datetime(2000, 1, 11, 10, 0, 0, 33000)]


def test_timestamps_without_any_date_raises():
    with pytest.raises(ValueError):
        to_timestamps(["#1-10:00:00.000"])


def test_unparseable_records_become_none():
    ts = to_timestamps(["Event#1-23:55:50(2000/1/10)", "#1-23:55:50.197", "garbage", None, "#1-23:55:50.296"])
    assert ts[2] is None and ts[3] is None
    assert ts[4] == datetime(2000, 1, 10, 23, 55, 50, 296000)
```

## Done when
`python scripts/verify_task.py T02` passes.

## Do not
- Use `\d\d` patterns. Fields are unpadded.
- Read files. This module parses strings only. T03 does the file I/O.
