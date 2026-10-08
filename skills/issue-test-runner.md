---
name: issue-test-runner
description: Use when given a GitHub issue that is a test/verification/regression task to EXECUTE — title prefixed `test:` or body containing a runnable command plus judgment criteria ("如何跑"/"判据"/"验证项"/"验收") — rather than a bug to fix. Do NOT use to fix bugs or to write/fix the test tool itself.
---

# Issue Test Runner

## Overview

Some issues are not fix-tasks — they are **repeatable test/verification entry points**. Their deliverable is *running the documented tool and writing the comparison report to `test-reports/`*, not code.

**Core distinction (get this wrong and the whole task is wasted):**
- `test:` issue → run the tool, report results. **No code changes.**
- `fix:` / `feat:` issue → use issue-resolver (analyze root cause, write code).

A prior "already merged / already passed" note in the issue is **not** a reason to skip running — the whole point is re-running after each change.

## When to Use

Use when:
- Issue title starts with `test:` (e.g. `test: 高频下单/撤单正确性对拍`)
- Body has a runnable command (`python3 Scripts/... run ...`, `./xxx.sh`) plus judgment criteria
- Body contains "如何跑" / "验证项" / "判据" / "验收演练"

Do NOT use when:
- The task is to write or fix the test tool itself (that's a code task)
- The issue asks for a bug fix, not a test run

## Workflow

### 1. Confirm it's a test-execution issue
Read title + body. Look for `test:` prefix, a runnable command, and criteria. If unclear, ask the user.

### 2. Extract from the issue body (never hardcode)
- **Command** — the exact run command(s), e.g. `python3 Scripts/verify_place_cancel.py run --rate 100 --seconds 60 --settle 30`
- **Prerequisites** — the issue's own 前置 checklist (docker up, test account clean, tools installed, …)
- **Criteria** — the judgment table (e.g. A1–A9) and each criterion's data source
- **Result semantics** — what PASS/FAIL/SKIP mean, and any exceptions the issue documents under "已知项"

### 3. Verify prerequisites FIRST
Work the issue's 前置 checklist. If one is missing (e.g. `spot-matchengine-btcusdt` not up), **do not force the run** — report the unmet prerequisite and stop. Environmental failures are not test results.

### 4. Run the tool exactly as documented
Execute the command verbatim. Capture stdout/stderr and exit code. If the issue documents multiple rates/rounds, run them all.

### 5. Map results to criteria, report faithfully
- Report each criterion as PASS / FAIL / SKIP exactly as the tool emits.
- **SKIP is not PASS.** A skipped surface (endpoint unreachable) means "that face was NOT verified", not "that face is fine". Keep the two counts separate.
- Never convert FAIL into PASS, never hide SKIP. (Precedent: a buggy report once printed "全部通过" while A6 FAIL + A7/A8 SKIP.)

### 6. Persist the report to the repo
Write each run's report into `test-reports/` (create it if absent), a fixed directory so reports don't get lost across conversations:
- One file per issue-run: `test-reports/issue-<number>-<YYYY-MM-DD>.md` (append `-<n>` if same day repeats)
- Raw captured stdout of each run goes into `test-reports/logs/issue-<number>-<YYYY-MM-DD>-rate<N>.log` so the comparison can be re-derived
- Reports ARE committed to git (see Step 7). A "max rate" sweep also lives here — see Step 9.

### 7. Commit the report to the repo (do NOT post to the issue)
```bash
git add test-reports/ && git commit -m "test: <issue> 回归对拍报告"
```
Include in the report: date, environment, actual vs target throughput, per-criterion table, PASS/FAIL/SKIP counts, exit code, and comparison against any baseline the issue documents.

**Do NOT post the full report to the issue as a comment.** Repeatable test issues accumulate many long reports; the files under `test-reports/` are the record and are easier to read/diff than stacked issue comments. The report file path in the commit message is sufficient traceability.

### 8. On FAIL — re-verify, then locate the root cause (do NOT soft-pedal)
When any criterion FAILs, a raw "X FAIL" line is not the deliverable. Do all of this:
1. **Re-run the correctness check** to confirm the FAIL is real and reproducible, not a one-off environmental blip.
2. **Locate the root cause** by tracing the failure to a concrete mechanism in the code/service logs — name the file:line, the counter/state that drifted, and the causal chain that produced it. Correlate log timestamps with the failing run's time window.
3. **State the cause plainly.** Do NOT mask, minimize, or reframe a real defect as an "environmental limit" or "known item". A low throughput number is an environment ceiling; a failed A-criterion (residual orders, balance drift, terminal-state mismatch) is a DEFECT until proven otherwise.
4. **Decide tracking based on the located cause**:
   - Real defect → propose creating a follow-up issue (do not auto-create; wait for user confirmation).
   - Proven environmental limit → note it, but only after you have traced WHY it's environmental.
5. Never attempt to fix the cause inline — that's a separate fix task. But you MUST hand back a located root cause, not just a symptom.

### 9. Rate sweep (when the issue asks for a max-rate ceiling)
Issue #1052 documents this: run `--rate 100`, then keep increasing (200, 400, 800, …) until either a criterion FAILs or the hard cap is hit (issue states `RATE=10000`). The goal is to find the environment/throughput ceiling, NOT to brute-force correctness at the cap.
- Stop at the FIRST FAIL — that's the finding. Do not keep pushing higher past it.
- `highrate_brush.py` spawns `n = int(RATE)` threads; at high RATE this is thread-heavy. Run one rate at a time, don't parallelize.
- A low 达标率 at a high nominal rate is an **environment ceiling**, not a FAIL — only a criterion failure (A1–A9) stops the sweep.
- Note: high rate may hit per-customer concurrency limit 5 (#923/#926); the brush staggers threads to avoid it, but beyond ~5 concurrent the service rejects — that IS the ceiling being measured.
- Record every rate step in the report table, and stop cleanly with a summary (highest clean-PASS rate vs ceiling found).

## Principles

- **No code changes.** Executing a test issue never edits production code. If the *test tool itself* is broken, surface it and suggest a separate fix task — don't patch it inline.
- **Read the tool from the issue, not from memory.** Script paths, rates, and criteria live in the issue body and evolve.
- **Prerequisite failures ≠ test failures.** Distinguish "couldn't run" from "ran and failed".

## Common Mistakes

| Mistake | Reality |
|---|---|
| Treating `test:` as a fix issue → writes code or declines because "already merged" | The deliverable is the report, not a diff |
| Reporting "all pass" while SKIP items exist | SKIP = unverified; count and flag it separately |
| Forcing the run with a missing prerequisite | Environmental failure poisons the criteria |
| Trusting a prior baseline PASS and not re-running | Each change invalidates the old baseline |
| Reporting "all pass" while a FAIL is buried in a known-item caveat | A failed criterion is a defect until root-caused |
| Stopping at "FAIL" without locating the cause | Symptom ≠ deliverable; trace to file:line + counter |

## Example: #1052 高频下单/撤单对拍

Issue body gives:
- **Command**: `python3 Scripts/verify_place_cancel.py run --rate 100 --seconds 60 --settle 30` (plus `--rate 200`)
- **Prereq**: docker full-stack **incl. `spot-matchengine-btcusdt`**; admin account free of stale orders; `psql` + python `redis` installed
- **Criteria**: A1–A9; active = `Pending+PartiallyCompleted`; A5 fixed to Pending; A6 zero-new-orders → FAIL; A9 zero-compare → SKIP
- **Deliverable**: write the 9-row table + throughput + PASS/FAIL/SKIP counts to `test-reports/issue-<number>-<date>.md` (committed to git, no issue comment)
