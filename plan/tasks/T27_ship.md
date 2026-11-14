# T27: README, disclosures, submission validator, package

| Milestone | Priority | Tier | Time box | Depends on |
|---|---|---|---|---|
| M100 | P0 | Haiku | 60 min | all code tasks (can start once T05 and T11 exist; finish last) |

- **Owns:** `README.md`, `scripts/validate_submission.py`, `scripts/package.py`, `docs/`.
- **Read:**
  - `01_project_definition.md` §2 (D1–D16);
  - `02_contracts.md` §3, §8, §11, §13;
  - the brief's submission list, quoted below.
- **Fixtures:** none.

## Goal
Make the submission reproducible and complete. The brief requires:
- code that reproduces all results from the raw video;
- a README listing the exact commands, the model and version used, and any outside data or models;
- disclosure of external resources and AI tools.

The judges score the repository and the brief.

## Specification

### README.md
Must contain these exact headings, in this order:
1. `## Quick start`: setup, then `make test`, then `make slice`.
2. `## Commands`: the full pipeline, `make manifest` → … → `make package`. For each step, give the machine, the expected runtime and the outputs.
3. `## Models and versions`: detector weights with their SHA-256 (from `models/SHA256SUMS`), the embedding model, the agent model and endpoint name (no key), and the Python and ffmpeg versions.
4. `## External data and code`: every package or model, with its licence.
5. `## AI tools used`: what each tool was used for, including Claude models that wrote code and tests and the organizer-approved model API used inside the agent. Say that humans verified the results.
6. `## Hardware and runtime`.
7. `## Data handling`: the event data is not redistributed, how to place the raw data, and private fixtures.
8. `## Outputs`: every submission file and what it means.
9. `## Limitations`: link to the brief.

Every `make <target>` mentioned must exist in the Makefile.

### `scripts/validate_submission.py`
- **Usage:** `validate_submission.py [--outputs outputs] [--slice START END]`.
- Calls `cattle.validate.schemas.validate_submission`, prints the errors, and exits with status 0 or 1.

### `scripts/package.py`
- **Usage:** `package.py --out PATH.zip [--dry-run]`.
- **Includes:** `README.md`, `cattle/`, `app/`, `scripts/`, `configs/`, `tests/` (minus `fixtures_private`), `labels/`, `outputs/` (including `clips/` and `pen_states_lite.sqlite`), `plan/`, `docs/`, `brief.pdf` if present, `requirements*.txt`, `Makefile`, `pyproject.toml`.
- **Does not include** the full state database. The README's `## Commands` explains `make db`, and `make dashboard` with `CATTLE_DB`.
- **Excludes:** `tests/fixtures_private/`, `.env`, `.venv/`, `work/`, `._*`, `__pycache__/`, `*.Avi`/`*.avi`, `models/*.pt`.
- **Dry run:** print one line per file. For each required file that is missing (`outputs/activity.csv`, `outputs/activity_minutes.csv`, `outputs/time_budgets.csv`, `outputs/brd_risk.csv`, `outputs/id_check/index.csv`, `outputs/pen_states_lite.sqlite`, `README.md`), print `MISSING <path>`.

## Locked test
```python
# LOCKED TEST: tests/acceptance/test_readme.py
import pathlib
import re
import subprocess
import sys

README = pathlib.Path("README.md")
HEADINGS = ["## Quick start", "## Commands", "## Models and versions", "## External data and code", "## AI tools used",
            "## Hardware and runtime", "## Data handling", "## Outputs", "## Limitations"]


def test_readme_sections_in_order():
    text = README.read_text()
    positions = [text.find(h) for h in HEADINGS]
    assert all(p >= 0 for p in positions), [h for h, p in zip(HEADINGS, positions) if p < 0]
    assert positions == sorted(positions)


def test_make_targets_in_readme_exist():
    text, mk = README.read_text(), "\n" + pathlib.Path("Makefile").read_text()
    for target in set(re.findall(r"make ([a-z0-9-]+)", text)):
        assert f"\n{target}:" in mk, target


def test_package_dry_run_excludes_private_material(tmp_path):
    r = subprocess.run([sys.executable, "-m", "scripts.package", "--dry-run", "--out", str(tmp_path / "x.zip")],
                       capture_output=True, text=True)
    assert r.returncode == 0, r.stderr[-1000:]
    listed = r.stdout
    for bad in ["tests/fixtures_private", ".env", "/._", ".venv/", "work/"]:
        assert bad not in listed, bad
    assert "README.md" in listed and "outputs/activity.csv" in listed


def test_validate_submission_cli_help():
    assert subprocess.run([sys.executable, "-m", "scripts.validate_submission", "--help"], capture_output=True).returncode == 0
```

## Done when
- `python scripts/verify_task.py T27` passes.
- H08 runs a fresh-clone check: clone into `/tmp/fresh`, `make setup`, copy the fixtures, `make test`, `make slice`.

## Do not
- Paste any API key, personal email or event image into the README.
