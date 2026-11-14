You are a worker on the "BRD Agentic Vision" hackathon repository. Your task ID is {{TASK}}.

Work only from these sources, in this order:
1. Your task card: `plan/tasks/{{TASK}}_*.md`.
2. The sections of `plan/02_contracts.md` that your card lists under "Read".
3. The repository files your card lists under "Read". Do not browse other files.

Procedure (Lane A bootstraps T00 from its card before using this template):
1. Run `python scripts/verify_task.py {{TASK}} --install`.
   - This writes your locked tests.
   - Commit them now: `git add <those test files> && git commit -m "{{TASK}}: locked tests"`.
   - Never edit a locked test. If one looks wrong, stop and report `BLOCKED: test` with the reason.
2. Implement only the files listed under "Owns" in your card.
3. Loop until both commands pass:
   - `python scripts/verify_task.py {{TASK}}` exits 0 with 0 skipped tests.
   - `python -m pytest -q tests/unit tests/contract` stays green.
4. Add your own small tests if useful. Name them after your module and keep each under 5 s.
5. Commit only your owned files: `git add <owned files> && git commit -m "{{TASK}}: <one-line summary>"`.
6. Write `reports/{{TASK}}.md` using `plan/templates/task_report.md`. Keep it under 300 words.

Hard rules:
- **No network access** except your own model API.
- **No package installs.** If you need a package, ask for it under "Requests" in your report.
- **No access to raw video.** Do not touch `raw_root` or `work_root`. Use only `tests/fixtures_private/` and synthetic data.
- **Do not edit** files outside "Owns". That includes `Makefile`, `configs/`, `cattle/constants.py`, `cattle/config.py` and `plan/`. Request such changes in your report instead.
- **Keep output short.** Never print more than 50 lines of command output; use `-q`, `head` and `tail`. Never print whole DataFrames or files.
- **Stop with status `BLOCKED`** if the same failure persists after 3 fix attempts.
- **Stop with status `PARTIAL`** when your card's time box is reached.
- **Follow the contract.** Use exactly the names, columns and signatures in `plan/02_contracts.md` and your card. If the contract is missing something you need, report it; do not invent a variant.
