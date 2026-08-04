# APOLLO — Project Instructions

Single source of truth for all engine deliveries. Read this BEFORE every zip is built.

---

## 1. File Naming Convention (added 2026-07-27)

**Format:** `[ENGINE_NAME]_[ddmmyy]_v[N].zip`

| Component | Rule | Example |
|---|---|---|
| `ENGINE_NAME` | Engine identifier | `Apollo_Backtest`, `Apollo_Live` |
| `ddmmyy` | Day, month, year — 2 digits each | `270726` (27 July 2026) |
| `vN` | Auto-incremented per iteration, restarting at v1 each new day | `v1`, `v2`, `v3` |

**Examples:**
- `Apollo_Backtest_270726_v1.zip` — first backtest-engine delivery on 27 July 2026
- `Apollo_Backtest_270726_v2.zip` — second iteration same day (bugfix)
- `Apollo_Live_280726_v1.zip` — first live-engine delivery on 28 July 2026

**Triggered by:** Any request that changes engine code.

**Additional rules:**
- Old zips from prior days are kept (history). Old zips from the SAME day are deleted when a newer version is delivered (no clutter).
- Individual changed files (e.g., just `app.py`) follow the same pattern: `app_270726_v2.py`
- Always include the versioned zip + individual changed files in `/home/z/my-project/download/`

---

## 2. Title-Version Sync (from Conversation for Next Version.txt L2526)

The landing-page title line in `backtest_engine/app.py` must always read:

> `Multi-timeframe RSI Recovery Scoring Engine — NSE Backtester vX.Y.Z — <codename>`

The version in the title MUST match the zip's version. No stale version strings allowed.

There is an HTML comment in `app.py` (around line 272) reminding of this rule. The pre-flight Test 12 enforces it programmatically.

---

## 2b. Version-Bump Checklist (mandatory since v4.3)

Every time engine code changes (new feature, bugfix, signal change), ALL of the following must be updated before zipping:

| # | What | Where | Example for v4.3 |
|---|------|-------|-------------------|
| 1 | Module header docstring | `apollo_core/scoring.py`, `constants.py`, `trade_engine.py` line 1 | `"""... (v4.3)"""` |
| 2 | App.py file header | `backtest_engine/app.py` line 10 | `Version: 4.3 — ...` |
| 3 | App.py HTML title | `backtest_engine/app.py` ~line 270 | `NSE Backtester v4.3 — Renko Hard Gate` |
| 4 | Pre-flight constants | `tests/preflight.py` lines 25-26 | `CURRENT_VERSION = "v4.3"` |
| 5 | Stale version list | `tests/preflight.py` ~line 210 | Add old version to `stale_versions` |
| 6 | CHANGELOG.md | Top of file | New `## Apollo Backtest vX.Y — ...` section |
| 7 | README.md | Line 1 | `# Apollo Engine vX.Y — ...` |
| 8 | PROJECT_INSTRUCTIONS.md | Change Log table | Row with date + summary |
| 9 | Zip filename | `/home/z/my-project/download/` | `Apollo_Live_300726_v1.zip` |

---

## 3. Pre-Flight Test Suite (mandatory since v3.4.2)

No zip gets built until `tests/preflight.py` passes ALL tests on the freshly-extracted zip (not just my working copy).

Current test count: **12 tests**:
1. Syntax check every `.py` file
2. `__future__` import ordering
3. Import every module
4. `app.py` title contains current version
5. `run_apollo.bat` path correctness
6. Real backtest on CYIENTDLM (end-to-end)
7. Backtest determinism (run twice, compare)
8. Bucket classifier does NOT skip
9. All required files present
10. `requirements.txt` valid
11. Universe file paths resolve correctly
12. Version string consistency across files

The test script ships inside the zip — user can run `python tests\preflight.py` to verify on their end.

---

## 4. Bucket Classifier is REFERENCE-ONLY (directive from user)

The bucket classifier is for **display/classification only**, NEVER a gate:
- Every stock reaches the Apollo engine, irrespective of bucket class
- Multiplier has NO effect on stock scores
- Scores come out raw as they should
- The `should_skip` return value was removed in v3.4.1 — never reintroduce it

Guard comment exists in `apollo_core/trade_engine.py` above `total = raw_total`. Do NOT change that line without explicit user approval.

---

## 5. User Skill Level — Novice

The user is not a developer and not tech-savvy. When explaining things:
- Use plain English, no jargon without definition
- Provide step-by-step instructions with exact paths and commands
- Anticipate failure modes (e.g., multiple Python processes, stale cache)
- Include screenshots/diagnostics the user can run themselves
- When in doubt, over-explain rather than under-explain

---

## 6. Phase Roadmap

| Phase | Status | Description |
|---|---|---|
| Phase 1 — Refactor | ✅ Complete (v3.4) | Code restructured into `apollo_core/` + `backtest_engine/` |
| Phase 2 Step 0 — Bucket ungated | ✅ Complete (v3.4.1, v3.4.2) | Gate removed, path bugfixes, pre-flight suite |
| Phase 2 Step 1 — Split `extract_trades()` | ⏳ Pending | `detect_exit_triggers()` (pure) + `execute_exit()` |
| Phase 2 Step 2 — Live engine core | ⏳ Pending | `live_engine/data_replay.py`, `signal_monitor.py`, `state_store.py` |
| Phase 2 Step 3 — Live dashboard | ⏳ Pending | `live_engine/dashboard.py`, `run_live.py` |
| Phase 2 Step 4 — Alert delivery | ⏳ Pending | `live_engine/alert_manager.py` (file log + Telegram) |
| Phase 2 Step 5 — Documentation | ⏳ Pending | `LEARN.md`, `RUNBOOK.md` |
| Phase 3 — VPS + Kite + Telegram | ⏳ Pending | Cloud deployment |
| Phase 4 — Paper trade | ⏳ Pending | Live data, no real orders |
| Phase 5 — Live trade | ⏳ Pending | Real orders |

---

## 7. Test Stocks (locked by user)

The 5 stocks for live simulation testing:
1. APOLLO
2. CYIENTDLM
3. SYRMA
4. JYOTICNC
5. KAYNES

---

## 8. File Paths

| Path | Purpose |
|---|---|
| `/home/z/my-project/download/` | User-facing deliverables (zips, individual files) |
| `/home/z/my-project/scripts/` | Generation scripts (persisted, not delivered) |
| `/home/z/my-project/worklog.md` | Shared multi-agent work log (append-only) |
| `/home/z/my-project/PROJECT_INSTRUCTIONS.md` | THIS FILE — read before every delivery |

---

## 9. Communication Style

- Confirm understanding before each major step
- Provide thorough novice-friendly guidance (like in "Conversation for Next Version.txt")
- Incremental delivery with verification gates
- When bugs are found: fix → bump version → run pre-flight → re-deliver
- Never ship without running the pre-flight suite

---

## Change Log to This File

| Date | Change |
|---|---|
| 2026-07-27 | Initial creation. Captured: file naming convention, title-version sync, pre-flight suite, bucket-ungated rule, novice user, phase roadmap, test stocks, paths, comms style. |
| 2026-07-30 | v4.3: Added version-bump discipline — every engine code change must update version in file headers, app.py title, pre-flight CURRENT_VERSION, CHANGELOG.md, README.md, PROJECT_INSTRUCTIONS.md, and zip filename. RENKO_HARD_GATE=5 added. classify_score() now accepts r_points for Renko Hard Gate. New action "RENKO GATED". |
| 2026-08-01 | v4.7: Added Pool E (Equity Analytics, 21 pts) — E1 52W-Low proximity, E2 Stochastic oversold turn, E3 Relative Volume surge, E4 VPT accumulation, E5 50-SMA recovery zone. Added entry gates G1 PE (etmoney), G2 liquidity floor, G3 52W-low proximity, G4 52W-high distance — enforced at entry (GATE-BLOCKED). New modules gates.py + fundamentals.py. Created tests/preflight.py. |
