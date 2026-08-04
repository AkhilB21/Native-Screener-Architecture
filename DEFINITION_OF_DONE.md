# Apollo Engine — Definition of Done (DoD)

**Every feature, fix, or build MUST pass every item below before the zip is pushed to the user.**
No exceptions. No "I'll verify later." If any checklist item fails, the build does NOT ship.

---

## MASTER RULE

> **"Do NOT rewrite proven code. Do NOT modify files you didn't change. Do NOT introduce changes outside the scope of the current task. Test cross-file contracts before delivering.**

The root cause of every regression in v4.8 delivery was violating one or more of these three rules.

---

## SECTION A — Pre-Work (Before Writing Any Code)

### A1. Read the Three Governing Documents
- [ ] `PROJECT_INSTRUCTIONS.md` — Read in full. Note version-bump checklist, pre-flight requirements, phase roadmap, file naming convention.
- [ ] `CHANGELOG.md` — Read the latest version section. Understand what was shipped last.
- [ ] `README.md` — Confirm it matches the current version. Flag if stale.

### A2. Identify Scope
- [ ] Write down EXACTLY which files need to change and WHY.
- [ ] Write down which files must NOT be touched.
- [ ] If the task involves `data_repo/`, `eod2_loader.py`, or `sync.py` — document the full import chain and function signatures before starting.

### A3. Confirm Understanding with User
- [ ] For non-trivial tasks, confirm understanding before starting work.
- [ ] State which files will change and which will be left untouched.

---

## SECTION B — Implementation Rules (While Writing Code)

### B1. Never Rewrite Proven Modules
- [ ] If a module (e.g., `data_repo/repo.py`, `eod2_loader.py`) was working in a previous delivery, do NOT rewrite it from scratch.
- [ ] Only make targeted, minimal changes to fix the specific issue.
- [ ] If you must restore a file from a previous zip, also update ALL callers to match.

### B2. Cross-File Contract Integrity
- [ ] Every `import` statement must reference a name that actually exists in the target module.
- [ ] Every function call must match the target function's parameter name, order, and count.
- [ ] Every CLI argument (`.bat` → argparse) must match the `add_argument` dest/flag name.

### B3. Case Sensitivity
- [ ] `_sym_to_eod2()` in `app.py` MUST return `.upper()` (parquet filenames are UPPERCASE).
- [ ] Symbol comparisons must be case-insensitive OR explicitly uppercased.
- [ ] Filenames on disk (`apollo_data/RELIANCE.parquet`) are UPPERCASE — never lowercase.

### B4. No Changes Outside Scope
- [ ] Do not add features, refactors, or "improvements" that were not requested.
- [ ] Do not fix "cosmetic" issues in files unrelated to the task.
- [ ] Do not reformat, reorder, or restyle code you didn't need to touch.

### B5. Preserve Working Paths
- [ ] `DEFAULT_DATA_DIR` in `app.py` must remain `"apollo_data"` (not `"data"`).
- [ ] `eod2_loader.py` must use `from data_repo import load_daily, list_symbols, _is_parquet_repo`.
- [ ] `data_repo/__init__.py` must export: `load_daily`, `save_daily`, `list_symbols`, `_is_parquet_repo`, `_get_repo_dir`.
- [ ] `sync.py` must import from `data_repo.repo` (not from `data_repo` directly) and use `load_daily(repo_dir, symbol)` / `save_daily(repo_dir, symbol, df, source=)`.

---

## SECTION C — Version Bump (9-Point Checklist from PROJECT_INSTRUCTIONS §2b)

Every engine code change requires ALL of these updated:

| # | What | Where | Status |
|---|------|-------|--------|
| 1 | Module header docstring | `apollo_core/scoring.py`, `constants.py`, `trade_engine.py` line 1 | [ ] |
| 2 | App.py file header | `backtest_engine/app.py` line ~10 (`Version: X.Y — ...`) | [ ] |
| 3 | App.py HTML title | `backtest_engine/app.py` ~line 270 (`NSE Backtester vX.Y — <codename>`) | [ ] |
| 4 | Pre-flight constants | `tests/preflight.py` lines 25-26 (`CURRENT_VERSION = "vX.Y"`) | [ ] |
| 5 | Stale version list | `tests/preflight.py` ~line 210 (add old version to `stale_versions`) | [ ] |
| 6 | CHANGELOG.md | Top of file — new `## vX.Y — ...` section | [ ] |
| 7 | README.md | Line 1 — `# Apollo Engine vX.Y — ...` | [ ] |
| 8 | PROJECT_INSTRUCTIONS.md | Change Log table — row with date + summary | [ ] |
| 9 | Zip filename | `/home/z/my-project/download/` — `[ENGINE]_[ddmmyy]_v[N].zip` | [ ] |

---

## SECTION D — Static Verification (Before Zipping)

This is the **most critical section**. It catches the class of bugs that plagued v4.8 delivery.

### D1. Import Chain Verification
- [ ] Grep every `from data_repo` and `import data_repo` across ALL `.py` files.
- [ ] For each import, verify the imported name exists in `data_repo/__init__.py`.
- [ ] For each import, verify the name is actually defined/re-exported in `data_repo/repo.py` (or the correct source module).

### D2. Function Call Verification
- [ ] Grep every call to `load_daily` — verify it uses `(repo_dir, symbol)` parameter order.
- [ ] Grep every call to `save_daily` — verify it uses `(repo_dir, symbol, df, source=)` parameter order.
- [ ] Grep every call to `list_symbols` — verify it uses `(repo_dir)` parameter.
- [ ] Grep every call to `_get_repo_dir` — verify it matches the function signature.

### D3. BAT → Argparse Verification
- [ ] Read every `.bat` file. Extract all `--flag` arguments passed to Python scripts.
- [ ] Read the corresponding Python script's `argparse.ArgumentParser` section.
- [ ] Verify every `.bat` flag has a matching `add_argument` in the Python script (exact flag name).
- [ ] Verify no typos in flag names (e.g., `--source-dir` vs `--eod2-dir`).

### D4. File Existence Verification
- [ ] Every file referenced in `.bat` files must exist in the zip.
- [ ] Every file referenced in `import` statements must exist in the zip.
- [ ] `apollo_data/` folder and its parquet files must be included if the backtester references them.
- [ ] `apollo_universe.json` must be included if the backtester references it.
- [ ] `run_sync_data.bat`, `build_universe.bat`, `build_universe.py` must be included if sync/universe features are advertised.

### D5. JSON Format Compatibility
- [ ] `apollo_universe.json`: Verify `_load_universe_json()` in `app.py` handles the actual JSON format (flat list `[{...}]` vs dict `{"symbols": [{...}]}`).

---

## SECTION E — Dynamic Verification (If Possible)

### E1. Syntax Check
- [ ] Run `python -c "import py_compile; [py_compile.compile(f, doraise=True) for f in [...]]"` on all modified `.py` files.

### E2. Import Check
- [ ] Run `python -c "import backtest_engine.app"` (or equivalent) and confirm no ImportError.

### E3. End-to-End Smoke Test
- [ ] If feasible, run a single-symbol backtest (e.g., `APOLLO`) and confirm it completes without error.
- [ ] Verify parquet loading produces a non-empty DataFrame.

---

## SECTION F — Zip Packaging

### F1. File Naming
- [ ] Zip filename follows `[ENGINE_NAME]_[ddmmyy]_v[N].zip` format.
- [ ] Example: `APOLLO_LIVE_040826_v1.zip`

### F2. Zip Contents
- [ ] All modified files are included.
- [ ] No unintended files are included (e.g., `__pycache__/`, `.pyc`, `.git/`).
- [ ] `apollo_data/` with parquet files is included.
- [ ] `apollo_universe.json` is included.
- [ ] `run_sync_data.bat` and `build_universe.bat` are included.
- [ ] CHANGELOG.md, README.md, PROJECT_INSTRUCTIONS.md are included.

### F3. Clean Old Zips
- [ ] Delete previous zip versions from the SAME day (keep prior-day zips as history).

---

## SECTION G — Communication

### G1. Delivery Message
- [ ] State exactly what was changed (files modified, lines changed).
- [ ] State exactly what was NOT changed and why.
- [ ] Include any manual steps the user needs to take.
- [ ] Include any known limitations or caveats.

### G2. Regression Prevention
- [ ] If a fix required restoring a file from a previous zip, explicitly state: "Restored X from proven v4.7 base. Verified all callers updated."
- [ ] If a `.bat` argument was added/changed, explicitly state the mapping.

---

## SECTION H — Project-Specific Rules (from PROJECT_INSTRUCTIONS.md)

### H1. Bucket Classifier
- [ ] Bucket classifier is REFERENCE-ONLY. Never a gate.
- [ ] `should_skip` was removed in v3.4.1 — never reintroduce.
- [ ] Multiplier has NO effect on scores.

### H2. Gates
- [ ] Since v4.8, gates G1-G4 are informational (never block entry).
- [ ] `GATE-BLOCKED` events must NOT exist in `trade_engine.py` or `signal_monitor.py`.

### H3. Test Stocks
- [ ] The 5 locked test stocks: APOLLO, CYIENTDLM, SYRMA, JYOTICNC, KAYNES.
- [ ] These must always have parquet files in `apollo_data/`.

### H4. User Communication
- [ ] User is a novice. Use plain English. No jargon without definition.
- [ ] Provide step-by-step instructions with exact paths and commands.
- [ ] When in doubt, over-explain rather than under-explain.

---

## SECTION I — Known Gaps to Fix

Issues identified during this review that need to be addressed in the next build:

1. [ ] **README.md is stale** — Still says v4.7. Needs full rewrite for v4.8 (parquet repo, apollo_data/, apollo_universe.json, run_sync_data.bat, build_universe.bat, gates informational, renko wired).
2. [ ] **PROJECT_INSTRUCTIONS.md Change Log** — Missing v4.8 entry. Add row for v4.8 with date and summary.
3. [ ] **Preflight test count** — PROJECT_INSTRUCTIONS.md says 12 tests, but v4.8 CHANGELOG says 20. Update to match.
4. [ ] **CHANGELOG v4.8 says `read_daily(symbol)`** — but the actual function is `load_daily(repo_dir, symbol)`. Fix the CHANGELOG description.

---

## Quick-Reference: Critical File Contracts

| File | Key Exports / Functions | Notes |
|------|------------------------|-------|
| `data_repo/__init__.py` | `load_daily`, `save_daily`, `list_symbols`, `get_manifest`, `scan_parquet_repo` | ALL must be exported. NO `_get_repo_dir` (doesn't exist) |
| `data_repo/repo.py` | `load_daily(repo_dir, symbol)`, `save_daily(repo_dir, symbol, df, source=)`, `list_symbols(repo_dir)` | repo_dir is FIRST param. NO `_get_repo_dir` function exists |
| `data_repo/sync.py` | CLI: `--source-dir` (alias for `--eod2-dir`), imports from `data_repo.repo` | NOT from `data_repo` directly |
| `backtest_engine/eod2_loader.py` | Uses `from data_repo import load_daily as _load_parquet_daily` | Do NOT rewrite |
| `backtest_engine/app.py` | `_sym_to_eod2()` returns `.upper()`, `DEFAULT_DATA_DIR = "apollo_data"` | Do NOT change to lowercase or "data" |
| `run_sync_data.bat` | Passes `--source-dir` flag | Must match sync.py argparse |
| `apollo_universe.json` | Flat list: `[{"sym": "RELIANCE", ...}, ...]` | NOT dict format |
