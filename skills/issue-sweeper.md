---
name: issue-sweeper
description: |
  Use when operating as a background autonomous agent (via /loop) that needs to monitor a GitHub repository for open issues, prioritize them by dependency order, and process them one at a time. Routes each issue to either the fix pipeline (issue-resolver → pr-reviewer → pr-resolver → pr-merge) for code tasks, or the test branch (issue-test-runner → close) for repeatable test-execution tasks whose deliverable is a report in test-reports/, not a PR. Use when the user asks to "sweep issues", "process all open issues", "monitor and fix issues", or "run the issue sweeper". Also use when you are in an autonomous loop and need to decide what work to do next on a repo with open issues.
compatibility:
  - gh (GitHub CLI) authenticated
  - git
  - issue-resolver skill
  - issue-test-runner skill
  - pr-reviewer skill
  - pr-resolver skill
  - pr-merge skill
  - graphify (knowledge graph CLI, installed via pip/uv)
---

# Issue Sweeper

Orchestration skill that monitors a GitHub repo's open issues, prioritizes by dependency order, and processes them one at a time through the full pipeline. Driven by `/loop` autonomous mode. Never parallel — one issue at a time, from open to closed, before starting the next.

## Core Principle

**Serial end-to-end pipeline with stage timeouts.** Each issue goes through all 4 stages before the next issue starts. No shortcuts, no parallel processing, no skipping stages. Every stage has a 10-minute hard timeout — no stage waits indefinitely.

**HARD GATE: One issue at a time. Period.** At any point, exactly ONE issue is in the pipeline. You do NOT start Issue N+1 until Issue N is fully CLOSED and verified. "Parallel waves," "batching," "running independent issues alongside" — these are all parallelism by another name. They are all forbidden.

**Violating the letter of the rules is violating the spirit of the rules.** There is no distinction between "spirit" and "letter" — the rules ARE the process. An autonomous agent following the letter IS following the spirit.

**Violating the serial principle is the most common failure.** When you see multiple open issues, the impulse to parallelize is strong — resist it. The pipeline is serial by design because:
1. Parallel worktrees collide and corrupt each other's state
2. Context from one issue informs the next — processing #3 may reveal patterns that change how you approach #4
3. "Independent" issues are an assumption, not a fact. Changes interact in unexpected ways
4. Safety and correctness > throughput speed
5. You are ONE agent with ONE worktree. You literally cannot parallelize — you would create conflicting worktrees that corrupt the repo

### Definition of "Serial"

**Serial means:** Issue N is picked → Stage 1 complete → Stage 2 complete → Stage 2.5 if needed → Stage 3 complete → Verified CLOSED → THEN pick Issue N+1.

**Serial does NOT mean:**
- "I'll start #4 while waiting for #3's CI" — that's parallelism
- "I'll run the trivial ones alongside the main one" — that's parallelism
- "Wave 1 in parallel, then Wave 2" — that's parallelism with a different name
- "Let me kick off #4's issue-resolver while #3 is in pr-reviewer" — that's pipeline parallelism

**If you catch yourself thinking any of the above:** STOP. You are about to violate the HARD GATE. Finish the current issue first.

## When to Use

- Autonomous `/loop` mode — deciding what to do next on a repo with open issues
- User says "sweep all issues", "process open issues", "run issue sweeper"
- User says "monitor issues and fix them"

## When NOT to Use

- One-off issue fix (use issue-resolver directly)
- PR review without context of an issue (use pr-reviewer directly)
- Urgent hotfix that needs to bypass normal pipeline

## Process Flow

```
issue-sweeper invoked
│
├─ CHECK: gh auth status → fail? stop, notify user
│
├─ RECOVERY CHECK: state file exists? → idempotent recovery (see §Idempotent Recovery)
│
├─ PR CHECK: any open PRs authored by you?
│   ├─ gh pr list --state open --author @me --repo <owner/repo>
│   ├─ If PRs exist → route each through PR REVIEW PIPELINE (skip issue-resolver)
│   │   ├─ For each open PR:
│   │   │   ├─ Check if review already submitted (gh pr view --json reviews)
│   │   │   ├─ If no review or previous review was CHANGES_REQUESTED → STAGE 2: pr-reviewer
│   │   │   │   ├─ APPROVED/COMMENTED → STAGE 3: pr-merge
│   │   │   │   └─ CHANGES_REQUESTED → STAGE 2.5: pr-resolver → pr-reviewer → pr-merge
│   │   │   └─ If already APPROVED → STAGE 3: pr-merge directly
│   │   ├─ After PR merged → VERIFY CLOSURE on linked issue
│   │   └─ ALL PRs processed → THEN continue to SCAN
│   └─ No PRs → continue to SCAN
│
├─ SCAN: gh issue list --state open
│   ├─ Filter: skip if assignee is not you AND not empty
│   ├─ Filter: skip if issue is in failed_issues (already tried 3+ times)
│   ├─ Filter: skip if issue is blocked (depends on failed issue)
│   └─ RESULT: list of candidate issues
│
├─ EMPTY? → phase = IDLE
│   └─ ScheduleWakeup(1200-1800s) for monitoring. Stop.
│
├─ BUILD PRIORITY QUEUE
│   ├─ Parse explicit dependencies from each issue body
│   │   Patterns: "Depends on #N", "Blocked by #N",
│   │            "阻塞于 #N", "依赖 #N", "需要先完成 #N"
│   ├─ Build dependency DAG (adjacency list)
│   ├─ Detect cycles (DFS)
│   │   └─ Cycles found → pick oldest issue, break cycle, record warning
│   ├─ Topological sort (Kahn's algorithm)
│   │   - Most-depended-on first
│   │   - Same tier → created_at ascending (oldest first)
│   └─ RESULT: ordered queue
│
├─ PICK NEXT: dequeue head → current_issue
│   ├─ Assign to self: gh issue edit N --add-assignee @me
│   ├─ retry_count = 0
│   └─ Write state file: in_progress = {number, stage: "classify", started_at: now}
│
├─ CLASSIFY: is this a test-execution issue? (see §Test-Issue Routing)
│   ├─ YES → TEST BRANCH (report to test-reports/, no PR)
│   │   ├─ STAGE T1: issue-test-runner ⏱ [10 min timeout]
│   │   │   ├─ Run the documented tool, write report to test-reports/
│   │   │   ├─ Report committed to git, NO issue comment
│   │   │   └─ Extract PASS/FAIL/SKIP from report
│   │   ├─ STAGE T2: closure
│   │   │   ├─ All PASS → close issue (reason "completed")
│   │   │   ├─ Any FAIL → do NOT close; locate root cause, propose follow-up fix issue
│   │   │   └─ SKIP only (unmet prerequisite) → report, leave open
│   │   └─ CONTINUE to VERIFY CLOSURE
│   └─ NO → FIX BRANCH (original pipeline, unchanged)
│
├─ SYNC: git checkout main && git pull origin main
│   ├─ Ensures latest code before every issue fix
│   ├─ FAILED (conflict, network) → retry once, then skip issue
│   └─ Stash any lingering local changes first: git stash && ... && git stash pop
│
├─ STAGE 1: issue-resolver ⏱ [10 min timeout]
│   ├─ Invoke: Skill("issue-resolver") with issue URL in context
│   ├─ STALL CHECK every 2 minutes: is the skill still active?
│   │   ├─ Skill returned output → continue
│   │   ├─ No output, no progress for 10 min → TIMEOUT
│   │   └─ Timeout → record failure, add to failed_issues, skip to next
│   ├─ Extract PR URL from skill output
│   ├─ FAILED → retry (max 3) or skip
│   ├─ OK → current_pr = { number, url }
│   │   └─ Write state file: in_progress.stage = "pr-reviewer"
│   └─ graphify . --update  (incrementally update knowledge graph)
│
├─ STAGE 2: pr-reviewer ⏱ [10 min timeout]
│   ├─ Invoke: Skill("pr-reviewer") with PR URL
│   ├─ STALL CHECK every 2 minutes
│   ├─ Review verdict:
│   │   ├─ APPROVED or COMMENTED → continue to STAGE 3
│   │   ├─ CHANGES_REQUESTED → go to STAGE 2.5
│   │   └─ FAILED or TIMEOUT → retry (max 3) or skip
│   └─ OK → continue
│
├─ STAGE 2.5: pr-resolver (only if CHANGES_REQUESTED) ⏱ [10 min timeout]
│   ├─ Invoke: Skill("pr-resolver") with PR URL
│   ├─ STALL CHECK every 2 minutes
│   ├─ Check fix-cycle counter
│   │   └─ fix-cycle >= 3 → STOP, record failure, skip issue
│   ├─ After fix → invoke pr-reviewer again (confirm fixes)
│   │   └─ Loop at most 3 times (prevent infinite review-fix cycle)
│   │   └─ Each pr-reviewer re-invocation also has 10 min timeout
│   ├─ OK → continue to STAGE 3
│   └─ graphify . --update  (incrementally update knowledge graph)
│
├─ STAGE 3: pr-merge ⏱ [10 min timeout]
│   ├─ Invoke: Skill("pr-merge") with PR URL
│   ├─ STALL CHECK every 2 minutes
│   ├─ FAILED or TIMEOUT → retry (max 3) or skip
│   └─ OK → merged
│
├─ VERIFY CLOSURE
│   ├─ gh issue view N --json state
│   ├─ state == "CLOSED" → completed_issues += { number, title }
│   ├─ state != "CLOSED" → wait 120s, re-check
│   │   └─ Still not closed → gh issue close N --reason "completed"
│   ├─ current_issue = null, current_pr = null
│   └─ Clear state file
│
└─ LOOP: queue empty? → IDLE phase. Else → PICK NEXT.
```

## Priority Algorithm

### Explicit Dependency Parsing

From each issue body, scan for these patterns (case-insensitive):

```
Depends on #N
Depends on: #N
Blocked by #N
Blocked by: #N
阻塞于 #N
阻塞于：#N
依赖 #N
依赖：#N
需要先完成 #N
需要先完成：#N
```

Extract `N` as integer. Issue `N` must be processed before this issue.

### DAG Construction

```python
# Build adjacency list: depends_on[issue] = [list of issues it depends on]
depends_on = {}
for issue in candidates:
    deps = parse_dependencies(issue.body)
    depends_on[issue.number] = deps

# Build reverse: depended_by[issue] = [list of issues that depend on it]
depended_by = {}
for issue, deps in depends_on.items():
    for dep in deps:
        if dep not in depended_by:
            depended_by[dep] = []
        depended_by[dep].append(issue)
```

### Cycle Detection

Run DFS on the DAG. If a cycle is found:
1. Pick the issue with the earliest `created_at` in the cycle
2. Break dependency from that issue to break the cycle
3. Record warning: `cycle_detected: [cycle members], broken_at: #N`

### Topological Sort (Kahn's Algorithm)

```python
queue = []  # priority: (depended_count desc, created_at asc)
in_degree = compute_in_degree(candidates, depends_on)

# Start with issues that have no dependencies (in_degree == 0)
ready = [i for i in candidates if in_degree[i.number] == 0]

# Sort by: most depended_on first, then oldest first
ready.sort(key=lambda i: (-depended_by.get(i.number, []).__len__(), i.created_at))
queue = ready

# Process
result = []
while queue:
    issue = queue.pop(0)
    result.append(issue)
    for dependent in depended_by.get(issue.number, []):
        in_degree[dependent] -= 1
        if in_degree[dependent] == 0:
            queue.append(dependent)
            queue.sort(key=lambda i: (-depended_by.get(i.number, []).__len__(), i.created_at))

return result
```

### Output

Present the sorted queue to the user before processing begins:

```markdown
## Issue Sweeper — 优先级队列

处理顺序：
  1. #3 "设计数据库 Schema" — 被 #1, #4, #8 依赖，创建 Sep 20
  2. #1 "实现用户注册 API"   — 被 #2 依赖，创建 Sep 19
  3. #2 "实现登录页面"       — 依赖 #1，创建 Sep 20
  4. #4 "优化查询性能"       — 依赖 #3，创建 Sep 22
  5. #8 "新增退款功能"       — 依赖 #3，创建 Sep 23

预计处理 5 个 issues，逐个处理。
```

## Test-Issue Routing

The CLASSIFY gate decides whether an issue is a **test-execution** issue (report deliverable, no PR) or a **fix/feat** issue (code deliverable, full PR pipeline). Route BEFORE `SYNC`/STAGE 1 so a test issue never enters issue-resolver.

**Route to the TEST branch when:**
- Issue title starts with `test:` (case-insensitive), OR
- Body has a runnable command PLUS judgment criteria ("如何跑" / "验证项" / "判据" / "验收"), AND the task is to *run an existing tool*, not to *write* the test code.

**Route to the FIX branch when:**
- Title is `fix:` / `feat:` / `docs:` etc., OR
- The deliverable is writing code — including **writing** the test itself (e.g. "编写集成测试", "add D1 integration tests"). Writing a test file IS a code task and produces a PR; only *running an existing repeatable test tool* is a test-execution task.

**Ambiguity rule:** if it is unclear whether the issue writes a test vs. runs an existing tool, the default is the FIX branch — the report-only test branch must not accidentally swallow a code task. If still uncertain after reading title+body, surface the question to the user rather than guessing.

**State:** record the classification in the state file: `in_progress.branch = "test" | "fix"`.

## Stage Details

### PR CHECK: Open PRs Before Scanning Issues

**Triggered after** RECOVERY CHECK and before SCAN. The sweeper must NOT start new issue work if there are open PRs that haven't been reviewed/merged yet. This prevents the "orphan PR" problem — PRs created by a previous issue-resolver run that never got reviewed because the loop restarted.

**Check for open PRs:**

```bash
gh pr list --state open --author @me --repo <owner/repo> --json number,title,headRefName,createdAt
```

**If PRs exist, process each one SERIALLY:**

1. **Check if the PR is already reviewed:**
   ```bash
   gh pr view <number> --repo <owner/repo> --json reviews,state
   ```

2. **Route based on review status:**

   | Review Status | Action |
   |---------------|--------|
   | No reviews submitted | → STAGE 2: pr-reviewer |
   | Latest review is `CHANGES_REQUESTED` | → STAGE 2.5: pr-resolver → pr-reviewer → pr-merge |
   | Latest review is `APPROVED` or `COMMENTED` | → STAGE 3: pr-merge directly |
   | PR already merged (state=MERGED) | → VERIFY CLOSURE on the linked issue, then skip |

3. **After PR merged:** Verify the linked issue is closed (extract issue number from PR body's `Closes #N` or `Fixes #N`). Then update the sweeper state file: add the issue to `completed` if closed, remove from queue.

4. **ALL open PRs processed → THEN continue to SCAN for new issues.**

**Why this comes before SCAN:**
- A PR represents work already done (issue-resolver completed). Starting a new issue while its PR is unmerged would create parallel worktrees — violating the serial principle.
- The PR itself proves an issue was already being worked on. The sweeper's responsibility is to finish what was started before starting anything new.
- This is the most common recovery scenario: issue-resolver finishes → PR created → loop restarts before pr-reviewer runs. Without this check, the sweeper would scan issues, find the same one still open, and create a **second** PR on the same issue.

**State tracking:** During PR pipeline processing, write to the state file with `in_progress.pr_url` and `in_progress.stage` as you would for a normal pipeline run.

### STAGE T1: issue-test-runner (test branch only)

**Triggered when** CLASSIFY routes an issue to the test branch (see §Test-Issue Routing). This stage replaces the whole fix pipeline (issue-resolver → pr-reviewer → pr-merge) — a test issue produces a report, not a PR, so none of the review/merge stages apply.

**Invoke with:** `Skill("issue-test-runner")` — include the issue URL in the current message.

**What to watch for:**
- The report lands in `test-reports/` and is committed to git. There is NO issue comment — long repeatable-test reports stay out of the issue thread (see issue-test-runner Step 7).
- Extract PASS/FAIL/SKIP counts from the report to drive STAGE T2 closure.
- A FAIL is not a closure — issue-test-runner hands back a located root cause; carry that into STAGE T2.

**After issue-test-runner completes** → do NOT run graphify. No code changed; the knowledge graph reflects code, and the report is documentation.

### STAGE T2: closure decision (test branch only)

Decide based on the report's PASS/FAIL/SKIP counts:

| Outcome | Action |
|---------|--------|
| All criteria PASS (SKIP counts separately, is NOT PASS) | Close issue: `gh issue close <N> --reason "completed"` |
| Any FAIL | Do NOT close. Locate root cause (issue-test-runner hands it back). Propose a follow-up fix issue — do NOT auto-create; wait for user confirmation. Leave the test issue open. |
| SKIP only (unmet prerequisite) | Do NOT close. Report the unmet prerequisite. It is an environmental failure, not a test result. |

After closure decision, clear the state file and continue the loop (queue empty → IDLE; else PICK NEXT).

### STAGE 1: issue-resolver

**Before invoking:** Always sync from main:

```bash
git checkout main && git pull origin main
```

This ensures every issue starts from the latest merged code. If stash has lingering changes from a prior stage that didn't clean up properly, `git stash` first and `git stash pop` after pull. If pull fails (network, conflict), retry once; if still fails, skip the issue.

**Invoke with:** `Skill("issue-resolver")` — include the issue URL in the current message so the skill can parse it.

**What to watch for:**
- The skill will ask for user confirmation before modifying code and before committing. In autonomous mode, auto-confirm if:
  - Changes are **low risk**: single file, <50 lines, no financial/auth/permission logic
  - Otherwise: record the issue in `pending_confirmation` list, skip this issue, continue to next
- After the skill completes, extract the PR number and URL from its output
- If the skill reports an error or cannot proceed, increment retry_count

**After issue-resolver completes successfully** → run graphify incremental update:

```bash
# Update the knowledge graph with code changes from the fix
# --update only re-extracts changed files, no full rebuild
graphify . --update
```

This keeps the project knowledge graph in sync with each code change. The graph reflects the latest codebase state after every issue fix. If `graphify-out/graph.json` does not exist yet (first run), run `graphify .` to build the initial graph.

**Auto-confirmation judgment:**

| Risk Level | Criteria | Action |
|------------|----------|--------|
| Low | 1 file, <50 lines, no money/auth/perm keywords | Auto-confirm |
| Medium | 2-3 files, or 50-150 lines, or touches business logic | Auto-confirm but note for review |
| High | >3 files, or >150 lines, or touches payment/security/permission | Record pending, skip |

### STAGE 2: pr-reviewer

**Invoke with:** `Skill("pr-reviewer")` — pass the PR URL from STAGE 1 output.

**What to watch for:**
- Review verdict is in the skill output
- If `CHANGES_REQUESTED`: proceed to STAGE 2.5
- If `APPROVED` or `COMMENTED` only: proceed to STAGE 3
- If review fails to submit (network error): retry (max 3)

### STAGE 2.5: pr-resolver

**Invoke with:** `Skill("pr-resolver")` — pass the PR URL.

**What to watch for:**
- After pr-resolver completes, invoke pr-reviewer again to confirm fixes
- Track review-fix cycle count: if you reach 3 review→fix→review loops on the same issue, STOP and record failure
- This mirrors pr-resolver's own `fix-cycle` counter — respect it

**After pr-resolver completes successfully** → run graphify incremental update:

```bash
# Update the knowledge graph with review-fix code changes
graphify . --update
```

Review feedback fixes often touch multiple files. Keeping the graph updated ensures the next issue's resolver has accurate codebase context.

### STAGE 3: pr-merge

**Invoke with:** `Skill("pr-merge")` — pass the PR URL.

**What to watch for:**
- CI failures: pr-merge will report which checks failed. If CI is flaky (timeout, runner died), retry. If real failure, skip.
- Conflict resolution: pr-merge auto-resolves if possible
- Merge success → proceed to VERIFY CLOSURE

### VERIFY CLOSURE

After merge, confirm the issue is closed:

```bash
gh issue view <number> --repo <repo> --json state --jq '.state'
```

- `CLOSED` → done, move to next issue
- Not `CLOSED` → wait 60s, re-check. If still not closed after 2 attempts:
  ```bash
  gh issue close <number> --repo <repo> --reason "completed"
  ```

## Stage Timeout Rules

Every skill invocation has a **10-minute hard timeout**. The sweeper does NOT wait indefinitely.

### Deadline Tracking

Each stage runs under a deadline. The deadline is absolute wall-clock time, not CPU time:

```
deadline = now + 10 minutes
```

### Stall Detection

Every **2 minutes** during a stage, the sweeper evaluates progress:

| Signal | Verdict |
|--------|---------|
| Skill returned output (text, JSON, tool results) | Running — continue |
| In-progress tool calls visible (Agent running, sub-agent dispatched) | Running — continue |
| No output for 2+ minutes, no active tool calls | STALL SUSPECTED |
| No output for 4+ minutes | STALL ALMOST CERTAIN |
| Deadline exceeded (10 min) | TIMEOUT |

### Timeout Procedure

When a stage times out:

1. **Record the failure** immediately — write to `.claude/sweeper_state.json` to prevent loss on context compaction
2. **Clear the in-progress marker** — unassign the issue if possible
3. **Add to failed_issues** — same format as other failures
4. **Mark dependent issues as blocked**
5. **Continue to next issue** — do NOT stop the sweeper

```
[FAILED] #N "title" — stage: issue-resolver, reason: TIMEOUT (10 min exceeded), retries: 0
```

### Stage-Specific Timeout Handling

| Stage | Timeout Behavior |
|-------|-----------------|
| issue-test-runner (T1) | Leave issue open, skip. Test issue stays open for next sweep round. |
| closure decision (T2) | N/A — instant; no timeout applies. |
| issue-resolver | Unassign issue, skip. If < 3 retries, issue stays open for next sweep round. |
| pr-reviewer | Skip to STAGE 3 (merge without review feedback). Record warning. |
| pr-resolver | Skip to STAGE 3 (merge current state). Record warning. |
| pr-merge | Retry once (CI may be slow). If still times out, skip. |

### Timeout Rescue Pattern

If a stage experiences a stall (4+ min no output) BUT the deadline has not expired:

1. **Ping the skill**: Use SendMessage or check the sub-agent's status
2. **If skill is alive**: wait, the 10-min deadline still applies
3. **If skill appears dead/crashed**: treat as timeout immediately

## Idempotent Recovery

The sweeper MUST survive context compaction, session interruptions, and loop restarts.

### State File

Always persist current state to `.claude/sweeper_state.json` in the project root:

```json
{
  "repo": "deng/klickl-escrow",
  "sweeper_round": "2026-09-23T02:00:00Z",
  "in_progress": {
    "number": 2,
    "title": "Shared Zod validators",
    "stage": "pr-reviewer",
    "started_at": "2026-09-23T02:15:00Z",
    "pr_url": "https://github.com/deng/klickl-escrow/pull/3",
    "retry_count": 0
  },
  "completed": [
    {"number": 1, "title": "shared types package"}
  ],
  "failed": [],
  "blocked": [],
  "queue": [3, 4, 5]
}
```

### Write State on Every Transition

State MUST be written to disk at these checkpoints:
- After picking an issue (before invoking issue-resolver)
- After issue-resolver completes (before invoking pr-reviewer)
- After pr-reviewer completes (before merging or pr-resolver)
- After pr-resolver completes
- After pr-merge completes
- On any failure or timeout

```bash
# Write state atomically
cat > .claude/sweeper_state.json.tmp << 'JSON'
{...}
JSON
mv .claude/sweeper_state.json.tmp .claude/sweeper_state.json
```

### Recovery on Restart

When issue-sweeper is invoked and `.claude/sweeper_state.json` exists:

1. **Load state file**
2. **Check `in_progress`** — if not null, you have an interrupted issue
3. **Determine recovery action:**

| in_progress.stage | Recovery Action |
|-------------------|-----------------|
| `classify` | Re-classify the issue (see §Test-Issue Routing), then resume at the correct branch. |
| `issue-test-runner` | Check if the report was committed to `test-reports/`. If yes → resume at STAGE T2 closure. If no → re-invoke issue-test-runner. |
| `closure-decision` | Re-apply the PASS/FAIL/SKIP table from the committed report and close/leave-open accordingly. |
| `issue-resolver` | Check if PR was already created (`gh pr list --head ...`). If yes → resume at pr-reviewer. If no → restart issue-resolver for this issue. |
| `pr-reviewer` | Check PR status. If review submitted → resume at next stage. If not → re-invoke pr-reviewer. |
| `pr-resolver` | Check if fix commits pushed. If yes → re-invoke pr-reviewer. If no → re-invoke pr-resolver. |
| `pr-merge` | Check merge status. If merged → proceed to VERIFY CLOSURE. If not → re-invoke pr-merge. |

4. **Deadline on recovery:** The recovered stage gets a fresh 10-minute timeout from recovery time. The interrupted run's elapsed time is discarded.
5. **If the interrupted issue can't be recovered** (e.g. branch deleted, PR closed): record failure, clear in_progress, pick next.

### No State File = Fresh Start

If `.claude/sweeper_state.json` does not exist, treat it as a fresh invocation. Rebuild the queue from GitHub issues. This handles first-run and manual state reset.

### State File Location

```
.claude/sweeper_state.json   # project root, gitignored (do NOT commit)
```

This file is machine-readable state, not user-facing. It survives context compaction because it's on disk, not in conversation memory.

## Updated Auto-Confirmation (Autonomous Mode)

In autonomous `/loop` mode, auto-confirmation thresholds are BROADER than the interactive mode matrix:

| Risk Level | Interactive Mode | Autonomous Mode |
|------------|-----------------|-----------------|
| Low | Auto-confirm | Auto-confirm |
| Medium | Auto-confirm (note) | Auto-confirm (note) |
| High (no money/auth/perm) | Record pending, skip | Auto-confirm with warning |
| High (money/auth/perm touched) | Record pending, skip | Record pending, skip |

**Principle:** In autonomous mode, the only hard stop is payment/security/permission changes. Everything else is auto-confirmed — the pr-reviewer stage is the safety net for Medium/High non-financial changes.

If a skill asks for confirmation mid-stage (e.g. "Proceed with this approach?") and the change is NOT money/auth/perm:
1. Auto-confirm immediately
2. Note in the state file: `"auto_confirmed": true, "risk": "medium"`
3. Continue

This prevents the #1 cause of sweeper stalls: waiting for human input that will never arrive in a `/loop` session.

### Retry Counts

| Failure Point | Max Retries | Action on Exhaustion |
|--------------|-------------|---------------------|
| issue-test-runner (run failure) | 1 | Leave issue open, record failure. Re-verify prereq before retry. |
| issue-test-runner (TIMEOUT) | 1 | Leave issue open, record failure. Issue stays open. |
| issue-resolver (analysis failure) | 3 | Skip, record failure |
| issue-resolver (build failure) | 3 | Skip, record failure |
| pr-reviewer (fetch failure) | 3 | Skip, record failure |
| pr-resolver (fix-cycle >= 3) | 0 | Skip immediately |
| pr-merge (CI failure) | 3 | Skip, record failure |
| pr-merge (conflict resolution failure) | 2 | Skip, record failure |
| Issue not closed after merge | 1 | Force close |

### Failure Record

When an issue is skipped, record:

```
[FAILED] #N "title" — failed at: <stage>, reason: <why>, retries: <count>
```

Present all failures in a summary at the end of each sweep round.

### Continue After Failure

**Critical:** When one issue fails, move to the NEXT issue in the queue. Do NOT stop the sweeper. A single issue failure must not block the pipeline.

Exception: If the failed issue is a dependency for remaining issues, those dependent issues become blocked. Mark them as blocked and note in the summary.

## State Tracking

Issue-sweeper tracks state through conversation context. On each invocation, scan current open issues and compare with what was already processed:

```
sweeper_round:
  start_time: 2026-09-23T10:00:00Z
  repo: owner/repo
  completed: [{number, title}]
  failed: [{number, title, failed_stage, reason, retries}]
  blocked: [{number, title, blocked_by}]  # issues whose dependency failed
  in_progress: {number, title} or null
  queue: [{number, title, dependencies}]  # remaining
```

When `/loop` re-invokes you and you reload this skill, first scan open issues:
1. Cross-reference open issues with `completed` and `failed` from prior rounds
2. Rebuild queue including new issues that appeared since last scan
3. Re-run priority algorithm (new issues may have new dependencies)
4. Continue processing from the queue head

## Monitoring Mode (IDLE Phase)

When the queue is empty (all issues processed or failed):

```
✅ Issue Sweeper — 本轮完成

成功关闭: 3 issues (#3, #1, #2)
跳过: 1 issue (#7 — pr-merge CI 持续失败)
阻塞: 1 issue (#8 — 依赖 #7)
剩余: 0

进入监控模式，每 20-30 分钟扫描新 issues...
```

Then call `ScheduleWakeup`:
- `delaySeconds`: 1200-1800 (20-30 min monitoring interval)
- `reason`: "监控新 issues",
- `prompt`: `<<autonomous-loop-dynamic>>`
- `noop`: `false` (this tick did work)

## Retry & Failure Strategy

### Retry Counts

| Failure Point | Max Retries | Action on Exhaustion |
|--------------|-------------|---------------------|
| issue-test-runner (run failure) | 1 | Leave issue open, record failure. Re-verify prereq before retry. |
| issue-test-runner (TIMEOUT) | 1 | Leave issue open, record failure. Issue stays open. |
| issue-resolver (analysis failure) | 3 | Skip, record failure |
| issue-resolver (build failure) | 3 | Skip, record failure |
| issue-resolver (TIMEOUT) | 1 | Skip, record failure. Issue stays open. |
| pr-reviewer (fetch failure) | 3 | Skip, record failure |
| pr-reviewer (TIMEOUT) | 0 | Skip to STAGE 3. Record warning. |
| pr-resolver (fix-cycle >= 3) | 0 | Skip immediately |
| pr-resolver (TIMEOUT) | 0 | Skip to STAGE 3. Record warning. |
| pr-merge (CI failure) | 3 | Skip, record failure |
| pr-merge (conflict resolution failure) | 2 | Skip, record failure |
| pr-merge (TIMEOUT) | 1 | Retry once. If still times out, skip. |
| Issue not closed after merge | 1 | Force close |

### Failure Record

When an issue is skipped, record:

```
[FAILED] #N "title" — failed at: <stage>, reason: <why>, retries: <count>
```

Write the failure to `.claude/sweeper_state.json` immediately to persist across context compaction.

Present all failures in a summary at the end of each sweep round.

### Continue After Failure

**Critical:** When one issue fails, move to the NEXT issue in the queue. Do NOT stop the sweeper. A single issue failure must not block the pipeline.

Exception: If the failed issue is a dependency for remaining issues, those dependent issues become blocked. Mark them as blocked and note in the summary.

## Red Flags — STOP Processing

These thoughts mean you're about to violate the sweeper's rules:

| Thought | Reality |
|---------|---------|
| "I can process #3 and #4 in parallel" | Serial only. Parallel worktrees collide. |
| "These issues are independent, no conflict risk" | "Independent" is an assumption, not a fact. Changes interact in unexpected ways. |
| "Let me do Wave 1 (parallel) then Wave 2" | Calling parallelism "Waves" is still parallelism. Serial only. One at a time. |
| "Processing one at a time wastes time sitting idle" | Safety + correctness > throughput. An incorrect merge costs far more time. |
| "#5 has no changes needed, I'll merge directly" | Every issue goes through all 4 stages. |
| "It's a test: issue, but I'll push it through issue-resolver" | Test-execution issues go to the test branch (issue-test-runner → close). No PR exists for them; routing to issue-resolver fabricates an empty PR. |
| "The test passed, but it's a repeatable entry point, keep it open" | PASS → auto-close (reason completed). Open repeatable tests accumulate as backlog; the report in test-reports/ is the record. |
| "This test issue FAILed, but I'll close it anyway to clear backlog" | FAIL → do NOT close. Locate the root cause and propose a follow-up fix issue. Closing hides a real defect. |
| "This is just a typo/documentation, skip pr-reviewer" | Every PR goes through review. "Trivial" changes still get reviewed — even typos can be wrong fixes. |
| "The README isn't production code, 规则零 doesn't apply" | 规则零 is process discipline. Every merge — code OR docs — goes through the pipeline. No exemptions. |
| "Running review on a one-line change is pure ceremony" | Review is a safety net, not ceremony. The cost of skipping it once is training yourself to skip it always. |
| "The review-fix loop went to 4 but this time it's really fixed" | 3 loops max. Record failure. |
| "CI is probably flaky, let me merge anyway" | CI must pass. If flaky, retry. Never skip CI. |
| "I'll process the simplest issue first to get momentum" | Priority order only. Dependency DAG determines order. |
| "The user won't mind if I batch the confirmations" | Ask once per issue. Auto-confirm only low-risk changes. |
| "I see 10 issues, I'll use a workflow to fan them out" | Serial only. One at a time. |
| "Parallel with #1: #3, #4, #5" | That's 4 issues in parallel. NO. One at a time. |
| "I can run #4 and #5's resolve steps in parallel" | One issue at a time through ALL 4 stages. Process #4 fully, THEN #5. |
| "The trivial ones can run alongside the main one" | "Trivial" ≠ parallel exemption. Serial means serial. |
| "While #3 waits for CI, I'll start #4's issue-resolver" | Pipeline parallelism. One issue in the pipeline. Finish #3, then #4. |
| "This is a quick win that reduces open-issue count" | Throughput vanity. Correctness > speed. Stay serial. |
| "The skill hasn't responded in 8 minutes, I'll wait longer" | No. 10 min hard deadline. Time out and move on. |
| "It's probably almost done, another 5 minutes won't hurt" | Sunk cost. Hit the deadline and fail forward. The issue stays open for the next round. |
| "I lost my state from last round, I'll guess where I was" | No guessing. Check `.claude/sweeper_state.json`. If missing, rebuild from GitHub. |
| "The sub-agent asked for confirmation but it's medium risk" | In autonomous mode, auto-confirm everything except money/auth/perm. Don't stall. |

## Important Principles

1. **Serial always.** One issue at a time. No parallel processing, ever.
2. **Route before acting.** Classify each issue as test-execution vs. fix/feat BEFORE entering any stage. A test issue goes to the test branch; never fabricate a PR for a report.
3. **Full pipeline always.** Every issue goes through its branch's stages. No shortcuts.
3. **Dependency order always.** Process in topologically sorted order. Most-depended-on first.
4. **Fail forward.** One issue failure does not stop the sweeper. Record and continue.
5. **Time-box every stage.** Every skill invocation has a 10-minute hard deadline. No indefinite waits.
6. **Ownership before action.** Assign the issue to yourself before starting work.
7. **Verify closure.** Don't assume merging the PR closes the issue. Check and confirm.
8. **State on disk.** Write sweeper_state.json at every transition. Survive context compaction.
9. **Recover gracefully.** On restart, detect interrupted issues and resume from the correct stage.
10. **Autonomous but careful.** Auto-confirm everything except money/auth/perm changes. Don't wait for human input.

## Quick Reference

### Commands

```bash
# List open issues
gh issue list --repo <owner/repo> --state open --limit 50 --json number,title,body,createdAt,assignees,labels

# Assign issue to self
gh issue edit <number> --repo <owner/repo> --add-assignee @me

# Check issue state
gh issue view <number> --repo <owner/repo> --json state --jq '.state'

# Force close issue
gh issue close <number> --repo <owner/repo> --reason "completed"

# Parse dependencies from body
grep -oiE '(depends on|blocked by|阻塞于|依赖|需要先完成)\s*#?\s*[0-9]+' <<< "$BODY"
```

### Auto-confirmation Decision Matrix

```
Single file?      Less than 50 lines?     No money/auth/perm keywords?
     YES  ────────────  YES  ──────────────  YES  → AUTO-CONFIRM
     YES  ────────────  NO   ──────────────  ANY  → AUTO-CONFIRM (note)
     NO   ────────────  ANY  ──────────────  YES  → RECORD PENDING, SKIP
     ANY  ────────────  ANY  ──────────────  NO   → RECORD PENDING, SKIP
```

## Common Mistakes

### Processing issues in parallel
**Wrong:** "I'll kick off #3 and #4 at the same time since they're independent."
**Wrong:** "Wave 1 (parallel): #1, #3, #4, #5. Wave 2 (after #1): #2."
**Wrong:** "The trivial ones (#3, #4) can run alongside the main chain (#1→#2)."
**Wrong:** "While #3 waits for CI, I'll start #4's issue-resolver."
**Right:** Pick #1. Process through ALL 4 stages until CLOSED. Then pick #2. Then #3. Then #4. Then #5. One at a time, serial, no exceptions.

### Skipping stages
**Wrong:** "This is a one-line fix, no need for code review."
**Wrong:** "It's just a README typo. Skip pr-reviewer and pr-resolver, just merge."
**Wrong:** "The README isn't production code, 规则零 only applies to production code."
**Right:** Every PR goes through pr-reviewer. Even one-liners and documentation. Review is a safety net, not ceremony.

### Ignoring dependency order
**Wrong:** "I'll pick the easiest issue first."
**Right:** Follow the topological sort order. Dependencies first, dependents later.

### Not verifying closure
**Wrong:** "PR merged, issue must be closed. Moving on."
**Right:** `gh issue view N --json state` and confirm `CLOSED`.

### Stopping on first failure
**Wrong:** "#7 failed at pr-merge. I'll stop and wait for the user."
**Right:** Record #7 as failed. If #8 depends on #7, mark #8 as blocked. Continue with other unblocked issues.