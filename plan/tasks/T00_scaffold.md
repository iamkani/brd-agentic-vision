# T00: Repo scaffold, configs, verify harness

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M10 | P0 | Sonnet (or the orchestrator itself) | 60 min | none |

- **Owns:** everything listed under Outputs. After merge, the shared files (`Makefile`, `configs/`, `requirements*`, `pyproject.toml`, `cattle/constants.py`, `cattle/config.py`) pass to the orchestrator.
- **Read:**
  - `02_contracts.md` §2, §3, §4, §6, §9, §10 (the `constants.py` and `config.py` blocks), §11, §12
  - `04_testing.md` §2
- **Runs in:** The team’s designated implementation repository on the Mac, with the approved plan at `plan/`. Lane A bootstraps T00 before the generic worker procedure applies. This task may use the network once, for `pip install`.

## Goal
Create a runnable, testable skeleton so every later card can be built in isolation and verified with one command.

## Outputs
1. **Git and ignores.** Run `git init -b main` if the implementation directory is new. Confirm `python3.12`, `ffmpeg` and `ffprobe` are on PATH. Create `.gitignore` containing at least these lines:
   `tests/fixtures_private/`, `work/`, `*.avi`, `*.Avi`, `*.AVI`, `*.mp4`, `outputs/clips/`, `outputs/live/`, `.env`, `.venv/`, `models/*.pt`, `__pycache__/`, `.DS_Store`, `._*`.
2. **`pyproject.toml`.** Project `cattle`, `requires-python = ">=3.11"`. Under `[tool.pytest.ini_options]` set `pythonpath = ["."]`, `testpaths = ["tests"]` and `addopts = "-q --import-mode=importlib"`, and declare the markers `slow`, `data`, `gpu`.
3. **Requirements.**
   - `requirements.txt` (core): numpy, pandas, pyarrow, pyyaml, pillow, scipy, opencv-python-headless, scikit-learn, jinja2, requests, matplotlib, streamlit, pytest.
   - `requirements-ml.txt`: ultralytics, torch, torchvision, transformers.
   - Create `.venv` with `python3.12 -m venv .venv`, install `requirements.txt`, then `pip freeze > requirements.lock`.
4. **Package `cattle/`** with an empty `__init__.py` in every subpackage of contracts §3: `io`, `quality`, `validate`, `extract`, `perception`, `calib`, `identity`, `behavior`, `eval`, `agent`, `analytics`, `live`, `tools`.
5. **`cattle/constants.py`**, exactly as in contracts §10:
   - `COWS`, `CAMS`, `CAM_SUFFIX`, `DAY_DATES`, `BASIS`;
   - `CLASSES` as `(code, label)` tuples in priority order;
   - `CODE_TO_LABEL`, `LABEL_TO_CODE`, `PRIORITY` (code → 0..9);
   - `GREY_SAT_MAX = 1.0`.
6. **`cattle/config.py`:**
   - `paths()` reads `configs/paths.yaml`. Environment variables `CATTLE_RAW_ROOT`, `CATTLE_WORK_ROOT` and `CATTLE_OUTPUTS_ROOT` override it. Returns a `SimpleNamespace` of `pathlib.Path` objects.
   - `load_yaml(name)` reads `configs/<name>.yaml`.
7. **`configs/`:**
   - `paths.yaml` (contracts §2);
   - `cameras.yaml` (below);
   - `classes.yaml` (a mapping with key `classes` and a list of `{code, label}` matching `CLASSES`);
   - `rules.yaml`, `agent.yaml`, `risk.yaml` (initial values below);
   - `zones.json` (`{"water": [], "bunk": [], "rest": []}`);
   - `calib/README.md` (one paragraph: format from contracts §9).
8. **`Makefile`** with every target from contracts §11:
   - `setup` creates the venv and installs requirements;
   - Set `PYTHON ?= .venv/bin/python` after declaring `setup` as the default goal. Worktrees and the PC can override `PYTHON` with their active interpreter.
   - `test` runs `"$(PYTHON)" -m pytest`;
   - `verify` runs `"$(PYTHON)" scripts/verify_task.py $(TASK)`;
   - every other target is a stub: `@echo "not implemented (Txx)"; exit 2`.
9. **`scripts/verify_task.py`.** Spec below. Add `scripts/__init__.py` so repository CLIs can run as modules.
10. **Empty folders and placeholders:**
    - `tests/unit/`, `tests/contract/`, `tests/integration/`, `tests/acceptance/`, `tests/fixtures/synthetic/`. Keep a `.gitkeep` in each initially empty directory so verification works from a clean worktree;
    - `reports/.gitkeep`;
    - `labels/` holding header-only CSVs for each file in contracts §8.16;
    - a placeholder `README.md` saying "see plan/".
11. **Bootstrap the verifier.** From the repository root, create the files above, run `make setup`, then run `.venv/bin/python scripts/verify_task.py T00 --install` followed by `make verify TASK=T00`. Stage the approved `plan/` and the scaffold, then commit with the message `T00: scaffold`. Verify T00 again from a clean worktree using the scaffold venv.

### `configs/cameras.yaml`
Quote the suffixes so YAML does not turn `006` into `6`.

```yaml
cameras:
  cam01: {folder: "Cam 01", suffix: "006", channel: 1, role: TBD, roi: null, masks: []}
  cam02: {folder: "Cam 02", suffix: "017", channel: null, role: TBD, roi: null, masks: []}
  cam03: {folder: "Cam 03", suffix: "003", channel: null, role: TBD, roi: null, masks: []}
```

### Initial `rules.yaml`, `agent.yaml`, `risk.yaml`
```yaml
# rules.yaml (T18 tunes these on the check-set 'tune' split only)
lying_aspect: 1.9
walk_speed: 0.02
night_conf_penalty: 0.3
social_dist: 0.04
isolation_dist: 0.25
```
```yaml
# agent.yaml (H09 fills base_url and model)
provider: openai_compat
base_url: "https://REPLACE-ME/v1"
model: "REPLACE-ME"
api_key_env: AGENT_API_KEY
max_steps: 6
max_tokens_per_case: 6000
daily_token_budget: 2000000
max_cases: 1500
cache_path: "work/agent_cache.sqlite"
offline_fallback: true
```
```yaml
# risk.yaml
domains:
  appetite: {feeding_min: -1.0, feeding_bouts: -0.5, bunk_visits: -0.5}
  depression: {lying_min: 1.0, walking_min: -0.5, isolation_frac: 0.8, nn_dist_median: 0.5}
  hydration: {drinking_events: -0.3}
min_coverage: 0.5
z_alert: 3.0
min_effect: 0.15
n_boot: 200
```

### `scripts/verify_task.py` specification
- **Usage:** `verify_task.py [-h] [--install] [--allow-skip] [--list] TASK [TASK ...]`.
- For each TASK, find exactly one card `plan/tasks/<TASK>_*.md`. Zero or several matches is an error.
- Extract every fenced Python code block (a fence of three backticks followed by the word `python`) whose first line matches `^# LOCKED TEST: (\S+)$`. The captured group is the repo path. Fail if a selected card has zero locked blocks or a duplicate output path.
- **`--list`:** print `TASK path` for each block and exit 0.
- **`--install`:** write each block to its path, creating folders. The file content is the block text with the first line included and exactly one trailing newline. Print `installed <path>`. Exit 0.
- **Default mode:**
  1. Compare every path to its block. Normalise `\r\n` to `\n` and strip trailing whitespace at the end of the file only.
  2. A missing or different file prints `MODIFIED OR MISSING: <path>` and exits 1.
  3. Run `[sys.executable, "-m", "pytest", "-q", "-rs", "-p", "no:cacheprovider", *paths]` and print its last 40 lines.
  4. A non-zero pytest exit means exit 1.
  5. If the pytest summary reports any skipped test and `--allow-skip` is not given, print `SKIPS ARE FAILURES` and exit 1.
  6. Otherwise print `VERIFY OK: <tasks>` and exit 0.

## Locked test
```python
# LOCKED TEST: tests/unit/test_scaffold.py
import importlib
import pathlib
import shutil
import subprocess
import sys
from datetime import date

import yaml

PKGS = ["cattle", "cattle.io", "cattle.quality", "cattle.validate", "cattle.extract", "cattle.perception",
        "cattle.calib", "cattle.identity", "cattle.behavior", "cattle.eval", "cattle.agent",
        "cattle.analytics", "cattle.live", "cattle.tools"]


def test_packages_import():
    for name in PKGS:
        importlib.import_module(name)


def test_constants_match_contract():
    from cattle import constants as C
    assert C.COWS == [f"C{i:02d}" for i in range(1, 11)]
    assert C.CAMS == ["cam01", "cam02", "cam03"]
    assert C.CAM_SUFFIX == {"cam01": "006", "cam02": "017", "cam03": "003"}
    assert C.DAY_DATES == {1: date(2000, 1, 10), 2: date(2000, 1, 11)}
    assert C.BASIS == ("observed", "estimated", "assumed")
    assert [c for c, _ in C.CLASSES] == ["drinking", "feeding", "ruminating_standing", "ruminating_lying",
                                         "grooming", "social", "walking", "standing", "lying", "other"]
    assert C.CODE_TO_LABEL["ruminating_lying"] == "ruminating (lying)"
    assert C.CODE_TO_LABEL["ruminating_standing"] == "ruminating (standing)"
    assert C.CODE_TO_LABEL["social"] == "social interaction"
    assert C.CODE_TO_LABEL["other"] == "other / not visible"
    assert C.LABEL_TO_CODE["other / not visible"] == "other"
    assert C.PRIORITY["drinking"] == 0 and C.PRIORITY["other"] == 9
    assert C.GREY_SAT_MAX == 1.0


def test_configs_parse():
    for name in ["paths", "cameras", "classes", "rules", "agent", "risk"]:
        data = yaml.safe_load(pathlib.Path(f"configs/{name}.yaml").read_text())
        assert isinstance(data, dict) and data, name
    cams = yaml.safe_load(pathlib.Path("configs/cameras.yaml").read_text())["cameras"]
    assert {c: cams[c]["suffix"] for c in cams} == {"cam01": "006", "cam02": "017", "cam03": "003"}
    classes = yaml.safe_load(pathlib.Path("configs/classes.yaml").read_text())["classes"]
    from cattle import constants as C
    assert [(c["code"], c["label"]) for c in classes] == list(C.CLASSES)


def test_paths_env_override(monkeypatch, tmp_path):
    from cattle import config
    monkeypatch.setenv("CATTLE_RAW_ROOT", str(tmp_path / "raw"))
    monkeypatch.setenv("CATTLE_WORK_ROOT", str(tmp_path / "work"))
    p = config.paths()
    assert p.raw_root == tmp_path / "raw" and p.work_root == tmp_path / "work"
    assert isinstance(p.outputs_root, pathlib.Path)


def test_gitignore_protects_event_data_and_secrets():
    gi = pathlib.Path(".gitignore").read_text()
    for pattern in ["tests/fixtures_private/", "*.avi", "*.Avi", "outputs/clips/", "outputs/live/", ".env", ".venv/", "._*"]:
        assert pattern in gi, pattern


def test_makefile_has_all_targets():
    mk = "\n" + pathlib.Path("Makefile").read_text()
    for t in ["setup", "test", "verify", "fixtures", "manifest", "extract", "detect", "track", "link", "identify",
              "label", "l1", "idcheck", "eval", "agent", "features", "risk", "clips", "cards", "viz", "live",
              "slice", "validate-submission", "package"]:
        assert f"\n{t}:" in mk, t


def test_verify_task_script():
    r = subprocess.run([sys.executable, "scripts/verify_task.py", "--help"], capture_output=True, text=True)
    assert r.returncode == 0 and "--install" in r.stdout
    r = subprocess.run([sys.executable, "scripts/verify_task.py", "--list", "T02"], capture_output=True, text=True)
    assert r.returncode == 0 and "tests/unit/test_clock.py" in r.stdout


def test_ffmpeg_available():
    assert shutil.which("ffmpeg") and shutil.which("ffprobe")
```

## Done when
- `python scripts/verify_task.py T00` prints `VERIFY OK`.
- `git log --oneline` shows the scaffold commit.

## Do not
- Implement any feature module beyond `constants.py` and `config.py`.
- Install the ML requirements on the Mac now. They go on the GPU workstation; the Mac's tests must not need them.
