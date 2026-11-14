# 02: Contracts

**Codex review draft of v1.2, 2026-11-14 CST** (see §14). Only the orchestrator edits this file; every change bumps the
version and is logged in §14. Workers treat it as read-only truth. If reality disagrees with it, stop
and report. Do not improvise.

## 1. Verified data facts (measured 2026-11-14)

| Topic | Fact |
|---|---|
| Source | Organizer-provided event data, arranged by day and camera |
| Local copy | Use `<raw_root>/day{1,2}/cam0{1,2,3}/Event*.Avi` in the authorized local workspace. |
| Counts | Approximately 1,750 files. Reconcile counts and checksums by day and camera against the private delivery manifest. |
| File name | `Event` + `YYYYMMDDhhmmss` (recorder clock, CST) + 3-digit camera code. The extension is `.Avi`; match it case-insensitively. |
| Camera code | cam01 → `006`, cam02 → `017`, cam03 → `003` (constant within a folder) |
| Duration | Normally 9,009 frames (~300.2 s) at 30 fps. 11 files are under 8 MB (recorder restarts), so **never assume 300 s**. |
| Coverage | Derive coverage and gaps from the approved manifest and recorder timestamps. Files can cross midnight. The dates in this public copy are illustrative. |
| Container | RIFF AVI. `movi` list type at byte 2044, first chunk at byte 2048. An `idx1` index sits at the end of complete files and is **absent in truncated copies**. |
| Chunk layout | Each frame is 3 chunks: video `00db` (keyframe) or `00dc` (delta); clock text `01db`/`01dc` (NUL-terminated ASCII); empty `02db`/`02dc` (size 0). |
| Video | HEVC Main, 4000×3000, yuvj420p (full range), about 2.6 Mbps at night and 4 Mbps by day. **Private fourcc `GHEC`: ffmpeg, OpenCV and Ultralytics cannot open the AVI directly.** |
| Keyframes | Frame 0, then 19, then every 60 frames (2.000 s); ~151 per file. A keyframe payload starts with NAL types 32, 33, 34 (VPS, SPS, PPS) then 19 (IDR_W_RADL) and **decodes on its own**. Delta frames start with NAL 1. Night keyframe ≈ 80 KB, day keyframe ≈ 600 KB. |
| Clock text | Record 0 is a header, `Event#<ch>-<H>:<M>:<S>(<YYYY>/<M>/<D>)`. Records ≥ 1 are `#<ch>-<H>:<M>:<S>.<mmm>`. H, M and S are **never zero-padded**; milliseconds are always 3 digits. Within a file, consecutive stamps differ by 21–51 ms with no dropped frames. |
| Clock channel | cam01 prints `#1`. cam02 and cam03 are recorded by T01 in `tests/fixtures_private/FIXTURES.json`. |
| Audio | None |
| Light | First-keyframe `signalstats` SATAVG = **0.0 at night (grey IR)** and **≈ 5.3 by day** (muted colour). In RGB, IR frames have R = G = B. Threshold `GREY_SAT_MAX = 1.0`. |
| Light schedule | Classify light from frames. A time-of-day proxy is only suitable as a secondary check. |
| Pen map (`pen_map.png`) | Use the organizer-supplied map locally. Confirm camera roles, orientation, and pen polygons during H01 and H02. |
| exFAT | macOS writes `._<name>` AppleDouble stubs (4 KB) next to files on the external SSD. **Always skip names starting with `._`.** |
| Recorder and site | The decoder uses the container and codec signatures. Site and camera identifiers remain in the private setup notes. |
| Night infrared | Near-infrared **illumination**, not thermal. No camera here measures temperature. |

## 2. Machines and paths

All paths come from `configs/paths.yaml`, overridable by environment variables. Never hard-code a path.

```yaml
# configs/paths.yaml
raw_root: "<local event data path>"     # env CATTLE_RAW_ROOT
work_root: "<local intermediate data path>"     # env CATTLE_WORK_ROOT
outputs_root: "outputs"                          # submission files, inside the repo (env CATTLE_OUTPUTS_ROOT)
fixtures_private: "tests/fixtures_private"       # event-derived fixtures, git-ignored
```

- Read paths through `cattle.config.paths()`, which returns `pathlib.Path` objects.
- Mount paths can contain spaces. Quote them in shell commands.
- Tables store paths **relative** to `raw_root`, `work_root` or `outputs_root`, always with forward slashes.

## 3. Repository layout

```
brd-agentic-vision/
  README.md  Makefile  pyproject.toml  requirements.txt  requirements.lock  .gitignore
  configs/   paths.yaml cameras.yaml classes.yaml rules.yaml agent.yaml risk.yaml zones.json calib/cam0N.json
  cattle/                        python package
    constants.py config.py
    io/        clock.py avi.py manifest.py clips.py
    quality/   light.py
    validate/  schemas.py __main__.py
    extract/   keyframes.py
    perception/ detect.py track.py yolo_data.py
    calib/     homography.py zones.py
    identity/  link.py assign.py embed.py
    behavior/  cues.py rules.py bouts.py crop_classifier.py
    signals/   breathing.py
    db/        state_db.py
    eval/      metrics.py report.py
    agent/     client.py tools.py loop.py cache.py trace.py gate.py
    analytics/ features.py risk.py cards.py
    live/      runner.py alert.py
    tools/     sheets.py
  app/       dashboard.py logic.py   (Streamlit prediction dashboard, T33)
  scripts/   verify_task.py make_fixtures.py build_manifest.py extract_keyframes.py detect.py
             click_points.py run_slice.py qa_report.py id_check_frames.py evaluate.py
             run_agent.py build_viz.py validate_submission.py package.py write_l1.py make_sheet.py
             sample_frames.py autolabel.py train_detector.py
  tests/     unit/ contract/ integration/ acceptance/ fixtures/ (synthetic, committed)
             fixtures_private/ (event data, git-ignored)
  labels/    seeds.csv cows.csv det_check.csv id_eval.csv check_set.csv   (human-made, small)
  models/    weights + SHA256SUMS (git-ignored except the sums)
  reports/   Txx.md (worker reports)
  outputs/   submission files (activity.csv … viz/)
  plan/      this plan
```

## 4. Identifiers

| Thing | Format | Example |
|---|---|---|
| Camera | `cam01`, `cam02`, `cam03` (from folder `cam0N`) | `cam02` |
| Camera role | `back`, `front1`, `front2` (from `configs/cameras.yaml`, set by H01) | `back` |
| File | `file_id` = file stem (unique: contains the camera code) | `Event20000110235550006` |
| Frame | (`file_id`, `frame_idx`), 0-based index of the video chunk in the file | (`Event20000110235550006`, 19) |
| Track (one camera) | `track_id` = `<cam>-<6 digits>` | `cam01-000123` |
| Global tracklet (linked cameras) | `gtrack_id` = `g<6 digits>` | `g000045` |
| Cow | `C01` … `C10`; `UNK` when unknown | `C07` |
| Day | `1` = 2000-01-10, `2` = 2000-01-11 (calendar date of the timestamp, **not** the folder) | `2` |
| Evidence (agent) | `e<n>`, unique within one case | `e3` |

## 5. Time rules

1. **One clock:** the recorder clock (CST, timezone-naive). Never convert to UTC.
2. **Frame timestamp:** from the clock text (`cattle.io.clock`). Frame 0 (the header) is back-filled as frame 1 minus the median frame step. The file-name time is only used to sort files.
3. A sample belongs to day 1 or 2 by `ts.date()`. Anything outside 2000-01-10 00:00 → 2000-01-12 00:00 is dropped.
4. **Minute grid:** `minute = ts.floor("min")`. Each day has 1,440 minutes, 00:00 to 23:59.
5. **Sample grid:** keyframes, about one every 2 s per camera. `cow_samples` uses 2-second bins (`ts.floor("2s")`).
6. **Storage:** pandas `datetime64` with millisecond precision in Parquet; ISO-8601 strings (`2000-01-10T13:05:00`) in CSV. In `activity.csv`, `end` is exclusive.

## 6. Behaviour classes

The order in this table **is the priority**: when two labels compete within a second, the higher row wins. Output labels must be written exactly as shown; they match the brief.

| Code | Output label | Working definition | Assigned by |
|---|---|---|---|
| `drinking` | drinking | Muzzle at the waterer | rules (water zone + head down) and agent |
| `feeding` | feeding | Head lowered inside the feed-bunk zone | rules (bunk zone + head down or body at bunk) and agent |
| `ruminating_standing` | ruminating (standing) | Standing, away from the bunk, regular chewing rhythm | agent or bursts only, **never rules** |
| `ruminating_lying` | ruminating (lying) | Lying, regular chewing rhythm | agent or bursts only, **never rules** |
| `grooming` | grooming | Licking or scratching herself, rubbing on a post or fence | agent only |
| `social` | social interaction | Contact with another cow: licking, butting, pushing, mounting | agent (rules may only flag candidates) |
| `walking` | walking | Moving across the pen for most of the bout | rules (speed) |
| `standing` | standing | On her feet, staying in place, not eating or drinking | rules |
| `lying` | lying | Body on the ground | rules (posture) |
| `other` | other / not visible | Anything else (coughing, stretching), or no camera shows her | rules or fill |

- **Minute label:** the label with the most seconds in that minute; ties are broken by priority.
- **Bout:** a maximal run of consecutive minutes with the same label **and** the same basis.

## 7. Basis and confidence

| Basis | Use when |
|---|---|
| `observed` | Directly supported by visible evidence at that time, from a confident rule, the agent with ≥ 2 independent views, a tag read, or a manual label. Also used for "no camera shows her" (`other / not visible`, no footage). |
| `estimated` | Inferred with low confidence, at night with weak cues, interpolated within a short gap, extrapolated from sampled bursts, or the agent hit its limits |
| `assumed` | No direct evidence: identity carried by continuity only, or a label carried across a gap longer than 60 s |

- `confidence` is in [0, 1].
- When aggregating, a tie in basis resolves conservatively: `assumed` > `estimated` > `observed`.

## 8. Tables

Intermediate tables are Parquet files under `work_root`. Submission files are CSV under `outputs_root`. Every table has a validator in `cattle.validate.schemas` (T05).

### 8.1 `manifest.parquet`
`file_id` str (unique) · `folder_day` int (1|2) · `cam` str · `rel_path` str · `name_start` datetime · `size_bytes` int (> 0) · `md5` str (32 hex or "") · `channel` int · `clock_start` datetime · `n_frames` int · `n_keyframes` int · `picture` str (`colour`|`grey_IR`|`decode_failed`) · `satavg` float · `yavg` float

### 8.2 `keyframes.parquet`
`file_id` · `cam` · `frame_idx` int · `ts` datetime · `day` int (1|2) · `picture` (`colour`|`grey_IR`) · `yavg` float · `img_path` str (relative to work_root, `keyframes/<cam>/<file_id>/<frame_idx:05d>.jpg`) · `crop_box` str (`"x,y,w,h"` full-res pixels or `""`) · `scale` float (output px ÷ full-res px)

### 8.3 `detections.parquet`
`file_id` · `cam` · `frame_idx` · `ts` · `det_idx` int · `cls` str (`cow`|`person`|`horse`|`vehicle`) · `conf` float [0,1] · `x1` `y1` `x2` `y2` float (**full-res pixels**, x1<x2, y1<y2, inside 4000×3000) · `model` str

### 8.4 `tracks.parquet`
`cam` · `track_id` · `file_id` · `frame_idx` · `ts` · `det_idx` · `x1` `y1` `x2` `y2` · `conf` · `cut_reason` str (`""`|`gap`|`merge`|`end`). Calibration (T09) adds `foot_x` `foot_y` (px), `pen_x` `pen_y` (0–1, NaN if uncalibrated), `zone` (`water`|`bunk`|`rest`|`open`|`outside`|`unknown`), `speed` (pen units/s, NaN on the first row of a track).

### 8.5 `gtracks.parquet`
`gtrack_id` · `cam` · `track_id` · `t_start` · `t_end`. One row per member track. A gtrack groups tracks from different cameras that are the same animal at the same time.

### 8.6 `identity.parquet`
`gtrack_id` · `t_start` · `t_end` · `cow_id` (`C01`…`C10`|`UNK`) · `confidence` · `basis` · `method` (`tag`|`manual`|`appearance`|`geometry`|`continuity`|`agent`) · `margin` float (best minus second-best score)

### 8.7 `cow_samples.parquet` (2-second bins)
`cow_id` · `ts` · `day` · `visible` bool · `cams` str (`"cam01;cam03"`) · `pen_x` `pen_y` · `zone` · `posture` (`lying`|`standing`|`unknown`) · `speed` · `head_down` (bool or NA) · `nn_dist` float · `staff_present` bool · `label` (class code) · `confidence` · `basis` · `source` (`rule`|`agent`|`fill`) · `trace_id` str · `weight_s` float (default 2.0) · `picture` (`colour`|`grey_IR`) · `box_aspect` float · `posture_conf` float.
Optional, from T29: `clf_label` · `clf_conf` · `disagree` bool.

### 8.8 `outputs/activity_minutes.csv`
`cow_id` · `day` · `minute` (ISO) · `behavior` (output label) · `confidence` · `basis` · `cameras` · `n_samples`. Exactly 1,440 rows per cow-day; `(cow_id, minute)` unique.

### 8.9 `outputs/activity.csv` (one row per bout)
`cow_id` · `day` · `start` · `end` (ISO, whole minutes, end exclusive) · `behavior` · `minutes` int (= (end−start)/60) · `confidence` · `basis` · `cameras` · `trace_ids` (`;`-separated or `""`). Per cow-day, the bouts tile 00:00 → next day 00:00 with no gaps and no overlaps.

### 8.10 `outputs/time_budgets.csv`
`cow_id` · `day` · `behavior` · `minutes` · `fraction` (= minutes/1440) · `minutes_observed` · `minutes_estimated` · `minutes_assumed`. All 10 behaviours for every cow-day, zeros included. Minutes sum to 1,440 per cow-day.

### 8.11 `outputs/id_check/index.csv`
`day` · `cam` · `slot` (`HH:MM`, every 30 min, 48 per day) · `frame_ts` · `file_id` · `frame_idx` · `n_boxes` · `n_identified` · `image` (relative path `id_check/day<d>/<cam>/<HHMM>.jpg`). 288 rows; a slot with no footage gets a placeholder image and `n_boxes = 0`.

### 8.12 Traces: `outputs/traces/<case_id>.jsonl`
One JSON object per line:
`{"case_id", "step", "type": "plan"|"action"|"observation"|"verdict"|"error", "tool", "args", "evidence_id", "summary", "image", "label", "confidence", "basis", "model", "tokens_in", "tokens_out", "ts"}`.
Rendered selections go to `outputs/traces/selected/{success,failure,night}.md`.

### 8.13 `hourly_features.parquet` (long format)
`cow_id` · `day` · `hour` (0–23) · `feature` · `value` · `coverage` (fraction of that hour's minutes observed or estimated, 0–1). Features (exact names): `feeding_min`, `feeding_bouts`, `bunk_visits`, `lying_min`, `lying_bouts`, `standing_min`, `walking_min`, `drinking_events`, `nn_dist_median`, `isolation_frac`, `visible_min`.

### 8.14 `outputs/brd_risk.csv`
`cow_id` · `risk_score` [0,1] · `rank` (1–10, a permutation) · `rank_low` · `rank_high` (bootstrap 5th–95th percentile, 1 ≤ rank_low ≤ rank ≤ rank_high ≤ 10) · `top_reasons` (non-empty, ≤ 3 reasons joined by `"; "`) · `evidence_clips` (`;`-separated paths relative to outputs; 2–3 for ranks 1–3) · `coverage` [0,1] · `flags` (`;`-separated: `low_coverage`, `no_cow_stands_out`, `possible_estrus`, `staff_handling`)

### 8.15 L3: `outputs/live/<run_id>/alert.json`
`{"run_id", "clip", "cams", "mode": "warm"|"cold", "started", "finished", "tier": "none"|"watch"|"check_today"|"urgent", "cows": [{"cow_id" (or "unconfirmed-<n>"), "risk_before", "risk_after", "reasons": [...], "acute_events": [...], "clip_path", "crop_paths": [...]}], "checks_done": [...], "warnings": [...]}`, plus `alert.html`, or `no_alert.html` when the tier is `none`.

### 8.16 Human label files (`labels/`)
- `cows.csv`: `cow_id, ear_tag, tag_color, marks`
- `seeds.csv`: `cow_id, cam, file_id, frame_idx, x1, y1, x2, y2, lighting, notes`
- `det_check.csv`: `cam, file_id, frame_idx, n_cows_true, notes`
- `id_eval.csv`: `gtrack_id, cow_id, labeler`
- `check_set.csv`: `sample_id, cow_id, cam, start, duration_s, behavior (code), labeler, sure (1–3), split (tune|test), notes`

### 8.17 State database: `pen_states.sqlite` (full) and `outputs/pen_states_lite.sqlite` (every 15th keyframe)
SQLite with foreign keys on. Full schema in T31.

- **Tables:**
  - `meta`;
  - `animals` (C01–C10 plus `UNK`);
  - `cameras`;
  - `files`;
  - `frames` (one per kept keyframe);
  - `annotations` (one per animal per frame: ID, box, pen position, zone, behaviour code and label, confidence, basis, identity confidence and method, source, trace).
- **Optional tables:** `cow_minutes`, `bouts`, `risk`, `brd_probability`, `hourly_features`.
- **Views:** `v_frame_states` (the per-frame annotated state of every animal) and `v_cow_day_budget`.
- **Timestamps:** ISO text with milliseconds.

### 8.18 `brd_probability` (T32)
`cow_id` · `p_brd` · `p_low` · `p_high` (5th–95th percentile) · `prior` · `appetite_contrib` · `depression_contrib` · `hydration_contrib` (log-odds) · `basis` (always `assumed`) · `calibrated` (bool) · `assumptions_version`. Assumptions live in `configs/prob.yaml`. Every display carries "model-estimated probability under stated assumptions; not a diagnosis".

## 9. Config files

| File | Content |
|---|---|
| `configs/cameras.yaml` | `cameras: {cam01: {folder: "Cam 01", suffix: "006", channel: 1, role: TBD, roi: [x, y, w, h], masks: [[[x, y], ...]]}, ...}`. ROI and masks are set by H01. |
| `configs/classes.yaml` | `classes: [{code, label}, ...]` in priority order (§6) |
| `configs/calib/<cam>.json` | `{"cam", "points": [{"name", "px": [x, y], "pen": [x, y]}], "H": [[...]], "rmse": float, "heldout": [...]}` |
| `configs/zones.json` | Polygons in **pen coordinates**: `{"water": [[x, y], ...], "bunk": [...], "rest": [...]}`. Pen coordinates are normalised: x = 0 at the chute-side (left) fence, 1 at the right fence; y = 0 at the BACK fence, 1 at the FRONT (bunk) edge. Optional `pen_length_m` and `pen_width_m` convert to metres. |
| `configs/rules.yaml` | Thresholds for T18 (lying aspect ratio, walking speed, head-down, social distance, night confidence penalty) |
| `configs/agent.yaml` | `provider, base_url, model, api_key_env, max_steps: 6, max_tokens_per_case, daily_token_budget, max_cases, cache_path, offline_fallback` |
| `configs/risk.yaml` | Feature weights per domain, `min_coverage`, `z_alert`, `n_boot` |

## 10. Module APIs (exact names; locked tests import these)

```python
# cattle/constants.py  (T00)
COWS = [f"C{i:02d}" for i in range(1, 11)]; CAMS = ["cam01", "cam02", "cam03"]
CAM_SUFFIX = {"cam01": "006", "cam02": "017", "cam03": "003"}
DAY_DATES = {1: date(2000, 1, 10), 2: date(2000, 1, 11)}
BASIS = ("observed", "estimated", "assumed")
CLASSES = [(code, label), ...]          # priority order, §6
CODE_TO_LABEL, LABEL_TO_CODE, PRIORITY  # PRIORITY[code] = 0 for drinking ... 9 for other
GREY_SAT_MAX = 1.0

# cattle/config.py  (T00)
def paths() -> SimpleNamespace          # raw_root, work_root, outputs_root, fixtures_private (Path)
def load_yaml(name: str) -> dict        # configs/<name>.yaml

# cattle/io/clock.py  (T02)
@dataclass(frozen=True) class ClockRecord: channel: int; seconds: float; date: date | None; is_header: bool
def parse_record(text: str) -> ClockRecord | None
def to_timestamps(texts, start_date=None, fill_header=True) -> list[datetime | None]

# cattle/io/avi.py  (T03)
@dataclass(frozen=True) class AviInfo: width: int; height: int; fps: float; fourcc: str; n_streams: int; has_index: bool
@dataclass class Frame: frame_idx: int; ts: datetime | None; image: np.ndarray   # RGB uint8 H×W×3
def probe(path) -> AviInfo
def index(path) -> pd.DataFrame          # columns frame_idx, offset, size, is_key, clock_text; complete chunks only
def read_clock(path) -> pd.Series         # index frame_idx → Timestamp
def read_payloads(path, frame_indices) -> Iterator[tuple[int, bytes]]
def iter_keyframes(path, max_width=None, crop=None) -> Iterator[Frame]      # crop = (x, y, w, h) full-res px
def decode_window(path, start_frame, end_frame, step=1, max_width=None, crop=None) -> list[Frame]   # inclusive

# cattle/quality/light.py  (T04)
@dataclass class Light: picture: str; satavg: float; yavg: float
def probe_file(path) -> Light            # first keyframe, ffmpeg signalstats
def classify_rgb(img) -> Light           # numpy: satavg = mean(max(RGB)-min(RGB)), yavg = mean luma

# cattle/io/manifest.py  (T04)
def parse_name(stem) -> tuple[datetime, str]
def scan(raw_root) -> pd.DataFrame
def enrich(df, raw_root, workers=4, light=True) -> pd.DataFrame

# cattle/validate/schemas.py  (T05)
def validate(table: str, df, **kw) -> list[str]          # [] means valid; kw: cows, days, window, require_clips
def validate_submission(outputs_dir, slice_window=None) -> list[str]

# cattle/extract/keyframes.py  (T06)
def extract_file(path, file_id, cam, out_root, max_width=1600, crop=None, jpeg_quality=90) -> pd.DataFrame
def extract_many(manifest, raw_root, out_root, workers=4, **kw) -> pd.DataFrame

# cattle/perception/detect.py  (T07)
class Detector:  __init__(weights="yolo11s.pt", imgsz=1280, conf=0.25, device=None, backend=None, masks=None)
                 detect_image(img, crop_box=None, scale=1.0) -> list[dict]
                 run(keyframes, images_root, batch=16) -> pd.DataFrame
# cattle/perception/track.py  (T08)
def track(dets, max_gap_s=6.0, min_iou=0.1, max_jump=0.25, merge_iou=0.6) -> pd.DataFrame
# cattle/calib/homography.py, zones.py  (T09)
def fit_homography(px, pen) -> tuple[np.ndarray, float];  def to_pen(H, xy) -> np.ndarray
def foot_point(x1, y1, x2, y2) -> tuple[float, float];   def zone_of(pen_xy, zones) -> str
def add_pen_coords(tracks, calibs, zones) -> pd.DataFrame
# cattle/behavior/bouts.py  (T10)
def samples_to_minutes(samples) -> pd.DataFrame;  def fill_minutes(minutes, cows, days) -> pd.DataFrame
def minutes_to_bouts(minutes) -> pd.DataFrame;    def bouts_to_minutes(bouts) -> pd.DataFrame
def time_budgets(minutes) -> pd.DataFrame
# cattle/identity  (T13, T14)
def link(tracks, max_dist=0.08, bin_s=2.0, min_shared_bins=3) -> pd.DataFrame
def assign(segments, scores, cows=COWS, min_score=0.5, min_margin=0.05, slice_s=10.0) -> pd.DataFrame
def pad_to_square(img, fill=0) -> np.ndarray;  def build_prototypes(emb) -> pd.DataFrame
# cattle/behavior/cues.py, rules.py  (T18)
def build_cow_samples(tracks, gtracks, identity, bin_s=2.0) -> pd.DataFrame
def label_rules(samples, cfg=None) -> pd.DataFrame
# cattle/eval  (T19)
def evaluate_behavior(pred_minutes, labels) -> dict;  def evaluate_identity(identity, id_labels) -> dict
# cattle/agent  (T20, T21): see the cards
# cattle/analytics  (T22, T23)
def hourly_features(minutes, samples=None) -> pd.DataFrame
def score(hourly, cfg=None, n_boot=200, seed=0) -> pd.DataFrame
# cattle/io/clips.py  (T17)
def cut_clip(path, start_frame, end_frame, out_path, max_width=1280, boxes=None) -> Path
# cattle/live/runner.py  (T26)
def run_clip(path, out_dir, state_dir=None, cfg=None, detector_backend=None, client=None) -> dict
# cattle/perception/yolo_data.py  (T28)
def to_yolo_lines(dets, img_w, img_h, crop_box=None, scale=1.0, classes=(...)) -> list[str]
def from_yolo_lines(lines, img_w, img_h, crop_box=None, scale=1.0, classes=(...)) -> list[dict]
def split_by_hour(df, val_frac=0.2, seed=0) -> tuple[pd.DataFrame, pd.DataFrame]
def stratified_sample(keyframes, n, by=("cam", "picture"), seed=0) -> pd.DataFrame
# cattle/behavior/crop_classifier.py  (T29)
class CropClassifier: fit(X, y) · predict(X) · predict_proba(X) · save(path) · load(path)
def add_second_opinion(samples, clf, embeddings) -> pd.DataFrame
# cattle/signals/breathing.py  (T30)
def breathing_rate(frames, fps, roi=None, band=(0.25, 1.5)) -> tuple[float, float]   # (bpm, quality)
# cattle/db/state_db.py  (T31)
def build_state_db(inputs: dict, out_path, every_n=1, analytics=None) -> Path
# cattle/analytics/risk.py  (T23, used by T32)
def domain_scores(hourly, cfg=None) -> pd.DataFrame   # cow_id, appetite, depression, hydration, composite, coverage
# cattle/analytics/brd_prob.py  (T32)
def brd_probability(hourly, cfg=None, n_boot=200, seed=0, prior=None, effects=None) -> pd.DataFrame
# app/logic.py  (T33)
def load_overview(db_path); def recompute_probability(db_path, prior=None, effects=None, n_boot=50)
def frame_states(db_path, cam, ts, max_offset_s=60); def annotate(image, states, scale, crop=(0, 0)); def cow_bouts(db_path, cow_id, day)
# Agent (T20/T21), sheets (T15), ID check (T16), cards (T24): exact signatures are in their cards.
```

## 11. Makefile targets

`setup` · `test` · `verify TASK=Txx` · `fixtures` · `manifest` · `extract` · `detect` · `track` · `link` · `identify` · `label` · `l1` · `idcheck` · `eval` · `agent` · `features` · `risk` · `clips` · `cards` · `viz` · `live CLIP=<path>` · `slice` · `validate-submission` · `package`.

Added in v1.2 and wired by the orchestrator when T31–T33 merge: `db` (full and lite state database), `prob`, `dashboard`.

- T00 wires `setup`, `test` and `verify`. Other targets are stubs that print `not implemented (Txx)` and exit with status 2. Run feature CLIs from the repository root with `python -m scripts.<name>`.
- Each task that implements a script asks the orchestrator to wire the target. Workers do not edit the Makefile.

## 12. Environment

- **Python:** 3.12 in a venv (`python3.12 -m venv .venv`). The Mac's Homebrew 3.14 is too new for some ML wheels.
- **ffmpeg / ffprobe:** ≥ 6 on PATH on both machines.
- **Core packages:** numpy, pandas, pyarrow, pyyaml, pillow, scipy, opencv-python-headless, scikit-learn, jinja2, requests, pytest.
- **Heavy, optional:** ultralytics (detector), torch, transformers (DINOv2).
- `requirements.lock` comes from `pip freeze` after T00.
- **Tests must not need:** a GPU, network access, model downloads, or the raw dataset. Heavy dependencies are mocked behind the `backend=` / `client=` parameters.

## 13. Never do

- Open the raw AVIs with OpenCV, Ultralytics or plain `ffmpeg -i` decoding. Use `cattle.io.avi`.
- Modify anything under `raw_root`.
- Write full-resolution frames for every keyframe. Write keyframes at ≤ 1600 px wide, or crops.
- Put event data (videos, frames, crops, clips) or API keys in git.
- Send event data to any service not approved in `configs/agent.yaml` (see H09).
- Use the check set's `test` split to tune thresholds.
- Present the risk **index** (T23) as a probability, show `p_brd` (T32) without its prior and its "model-estimated under stated assumptions" label, or call anything a diagnosis.

## 14. Change log

- 1.0 (2026-11-14): first version.
- 1.1 (2026-11-14), after reading the prior research (`06_prior_work.md`) and checking locked tests numerically:
  - added the recorder, site and night-infrared facts (§1);
  - added `signals/`, `yolo_data.py`, `crop_classifier.py` and new scripts (§3);
  - added `picture`, `box_aspect` and `posture_conf` to `cow_samples` (§8.7);
  - added `Detector(masks=)` and `run_clip(detector_backend=)`;
  - added the T28–T30 APIs (§10);
  - validators now allow extra columns.
- 1.2 (2026-11-14), team request:
  - state database deliverable (§8.17, T31);
  - assumption-based P(BRD) (§8.18, T32);
  - Streamlit prediction dashboard (`app/`, T33);
  - `risk.domain_scores` exported for T32;
  - Makefile targets `db`, `prob`, `dashboard`;
  - the probability wording rule in §13.
- Codex review draft (2026-11-14). Clarified CLI entry points and T00 bootstrap, corrected the T23 null assertion, and revised R01/R02 index publication. Pending contract changes are recorded in `CODEX_REVIEW.md`.
