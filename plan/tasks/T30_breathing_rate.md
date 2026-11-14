# T30: Breathing rate from short bursts (video "R" of DART)

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M70/M90 | P2 (stretch) | Haiku | 45 min | T03 (for real use only) |

- **Owns:** `cattle/signals/__init__.py`, `cattle/signals/breathing.py`.
- **Read:** `06_prior_work.md` §2 item 5; `02_contracts.md` §1 (rows Keyframes and Video); `03_pitfalls.md` row 26.
- **Fixtures:** none (synthetic frame stacks).

## Goal
An increased respiratory rate is a classic BRD sign that a veterinarian recognises, and the prior research proposed to measure it with frequency analysis. On a **still** cow (lying, in daylight), the flank rises and falls once per breath. A 10–20 s burst at 30 fps, decoded with `avi.decode_window`, is enough to find that frequency.

Uses:
- **L2:** an optional respiration domain, per cow-hour median, basis `estimated`.
- **L3:** an evidence item.

## Specification
`breathing_rate(frames, fps, roi=None, band=(0.25, 1.5)) -> tuple[float, float]`
- **Input:** `frames` is a `(T, H, W)` or `(T, H, W, 3)` array. `roi = (x, y, w, h)` in frame pixels; `None` means the whole frame.
- Raise `ValueError` if `T / fps < 5` seconds.
- **Method:**
  1. Take the per-frame mean intensity of the ROI (grey).
  2. Detrend linearly and apply a Hann window.
  3. Take an rFFT.
  4. Find the peak bin within `band` Hz (15–90 breaths per minute).
  5. Refine the peak with parabolic interpolation on the log power of the peak bin and its two neighbours.
  6. **bpm** = refined peak Hz × 60.
  7. **quality** = power in the peak bin **and its two neighbours** ÷ total power in `band`, in [0, 1]. A breathing frequency that falls between two FFT bins splits its energy across both, so a single-bin ratio would understate quality.
- **Return:** `(bpm, quality)`. The caller keeps estimates only with quality ≥ 0.6. A reference implementation measured about 0.98 on clean signals and 0.28–0.40 on pure noise.

`burst_for(path, frame_idx, seconds=15) -> np.ndarray` is a convenience wrapper around `decode_window` (inclusive range, `max_width=640`, grey). Not tested here.

## Locked test
```python
# LOCKED TEST: tests/unit/test_breathing.py
import numpy as np
import pytest

from cattle.signals.breathing import breathing_rate


def stack(freq_hz, seconds=15, fps=30, amp=4.0, noise=1.0, seed=0, size=(48, 64)):
    rng = np.random.default_rng(seed)
    t = np.arange(int(seconds * fps)) / fps
    base = 100 + amp * np.sin(2 * np.pi * freq_hz * t)
    return (base[:, None, None] + rng.normal(0, noise, (len(t), *size))).astype(np.float32)


@pytest.mark.parametrize("freq, bpm", [(0.5, 30.0), (0.9, 54.0)])
def test_recovers_breathing_frequency(freq, bpm):
    est, q = breathing_rate(stack(freq), fps=30)
    assert abs(est - bpm) <= 3.0 and q > 0.6


def test_roi_isolates_signal():
    f = stack(0.6)
    f[:, :, 32:] = 100 + np.random.default_rng(1).normal(0, 5, f[:, :, 32:].shape)
    est, q = breathing_rate(f, fps=30, roi=(0, 0, 32, 48))
    assert abs(est - 36.0) <= 3.0 and q > 0.6


def test_static_noise_has_low_quality():
    for seed in range(4):
        _, q = breathing_rate(stack(0.5, amp=0.0, noise=2.0, seed=seed), fps=30)
        assert q < 0.5, seed


def test_too_short_raises():
    with pytest.raises(ValueError):
        breathing_rate(stack(0.5, seconds=3), fps=30)
```

## Done when
`python scripts/verify_task.py T30` passes.

## Do not
- Run this on walking or night (grey, low-contrast) cows without reporting the quality.
- Report a respiratory rate without its quality and the basis `estimated`.
