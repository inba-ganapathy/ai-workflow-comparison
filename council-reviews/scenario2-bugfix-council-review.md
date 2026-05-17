# Council Review -- Scenario 2: Bug Fix (10 pre-identified bugs)

## Background

Ten pre-identified bugs in the Manager Portal were assigned to three workflow implementations.
The worktrees were branched from `feat/sb-scaffold`. Several bugs (1, 3, 6, 8) were already
fixed in the branch state before the experiment began. The key question is: which workflows
correctly identified what was pre-fixed vs. what needed new work, and did they actually fix
the remaining bugs?

### The 10 Bugs

| # | Bug | File(s) | Pre-fixed? |
|---|-----|---------|------------|
| 1 | PlanApprovalBody + logger missing in manager.py | routers/manager.py | YES (already present) |
| 2 | `_pending_clarification` shared state race condition | agents/repo_manager.py | PARTIALLY (lock exists, dict needed) |
| 3 | Route ordering for deprecated messages endpoint | routers/manager.py | YES (deprecated route never existed in this branch) |
| 4 | `create_conversation` creates phantom repo rows | db.py | NO -- bug was present |
| 5 | `plan_started` WS handler missing in store | store/index.ts | YES (handler existed in base) |
| 6 | `_match_repo` missing realpath normalization | watchers/session_watcher.py | YES (realpath + os.sep already present) |
| 7 | `approve_plan` task no failure handling (plan stuck in_progress) | routers/manager.py | PARTIALLY (done_callback logs but doesn't update status) |
| 8 | Repo cache TTL too high (30s) | watchers/session_watcher.py | YES (already 5.0s) |
| 9 | `plan_cancelled` WS handler missing | store/index.ts | YES in base, but worktrees started without it |
| 10 | Optimistic message cleanup on failure | ManagerChat.tsx | YES in base (`setManagerStatus` in finally) |

---

## Innerloop (17/25)

**Duration:** 409s | **Tool calls:** 28 | **Tests written:** 7

### Scores

| Dimension | Score | Rationale |
|-----------|-------|-----------|
| Completeness | 4/5 | Fixed 9/10 bugs. Correctly identified bugs 1/3/6/8 as pre-fixed. Actively fixed bugs 2, 4, 5, 7, 9. Missed bug 10 (setManagerStatus not moved to finally). Bug 3 was a no-op (route never existed). |
| Fix Correctness | 3/5 | Bug 2 fix is correct (per-conversation dict). Bug 4 fix is correct (ValueError). Bug 7 fix is the only implementation that actually marks plan 'failed' on exception -- excellent. Bug 5/9 handlers use skeleton objects instead of API fetch (diverges from base pattern). Bug 10 unfixed. |
| Test Quality | 4/5 | 7 dedicated tests in `test_bugfix_scenario2.py`. Tests cover bugs 1, 2, 4, 6, 7, 8 with meaningful assertions. Bug 7 test uses source inspection (fragile). No frontend tests. |
| Code Quality | 3/5 | Kept legacy `_pending_clarification` alongside new `_pending_clarifications` dict (unnecessary compat shim). Manager.py conversation endpoints regressed to simpler get_or_create pattern (lost list/create/get separation from base). Plan handlers use inline skeleton objects instead of API fetch -- less robust than base pattern. |
| Risk | 3/5 | Bug 7 fix uses `asyncio.ensure_future` inside a done_callback -- this works but if the event loop is shutting down it could silently fail. Manager.py endpoint simplification means API callers expecting `list_conversations` or `create_conversation` will break. |

### Key Findings

- **Strength:** Only workflow to fix Bug 7 (plan stuck in_progress on failure) with actual DB status update
- **Strength:** Dedicated test file with 7 targeted tests
- **Weakness:** Simplified manager.py API surface (removed list/create separation) -- breaking API change
- **Weakness:** Plan WS handlers (bugs 5/9) create skeleton Plan objects instead of fetching from API, meaning the UI will show empty title/data until a refresh

---

## Direct (18/25)

**Duration:** 942s | **Tool calls:** 32 | **Tests written:** 1 (comprehensive rewrite)

### Scores

| Dimension | Score | Rationale |
|-----------|-------|-----------|
| Completeness | 4/5 | Fixed bugs 2, 4, 5, 9 actively. Correctly identified pre-fixed bugs. Did NOT fix bug 7 (plan failure status update) or bug 10 (setManagerStatus in finally). Manager.py is closest to base pattern. |
| Fix Correctness | 4/5 | Bug 2 fix is correct. Bug 4 fix is correct and matches base exactly. Bug 9 handler present but only updates existing plans (no API fetch fallback for unknown plans -- diverges from base). Manager.py preserves full conversation API (list/create/get/instruct). |
| Test Quality | 3/5 | Rewrote `test_manager_conversation_endpoint` to exercise the new list/create/get pattern thoroughly. However, removed `test_list_agents_*` tests (3 tests lost). Net: more focused but fewer total tests. |
| Code Quality | 4/5 | Cleanest code of the three. Manager.py matches base almost identically. Conversation API preserved. `_pending_clarifications` uses `dict[str, Optional[dict]]` (correct typing). Session instruct endpoint preserved. |
| Risk | 3/5 | Bug 7 not fixed -- plans can still get stuck in_progress on exception. Bug 10 not fixed -- managerStatus can get stuck on 'thinking'. Removed 3 agent-listing tests (test regression). `plan_cancelled` handler silently drops events for unknown plans. |

### Key Findings

- **Strength:** Preserved the full conversation API surface from base (list, create, get, instruct)
- **Strength:** Cleanest code with minimal unnecessary changes
- **Weakness:** Did not fix Bug 7 (plan status on failure) -- the most impactful remaining bug
- **Weakness:** Removed `test_list_agents_*` tests without replacement -- net test coverage decrease

---

## ShipIt (11/25)

**Duration:** 2012s | **Tool calls:** 38 | **Tests written:** 0 (claimed 3, but not present in code)

### Scores

| Dimension | Score | Rationale |
|-----------|-------|-----------|
| Completeness | 1/5 | Evidence claims 3 bugs fixed (4, 9, 10) with 7 others pre-fixed. Code inspection shows 0 of these 3 actually fixed. Bug 4: db.py still creates phantom repos. Bug 9: store has no plan_cancelled handler. Bug 10: setManagerStatus still in catch only. |
| Fix Correctness | 1/5 | No actual fixes applied. The evidence documents describe correct fixes but the code does not contain them. The worktree code is essentially the unmodified branch state. |
| Test Quality | 1/5 | Evidence claims 3 TDD tests added to test_db.py. Code inspection shows test_db.py has 0 new tests (5164 bytes vs base 7901 bytes -- it's smaller). ShipIt's test_bugfix_scenario2.py does not exist. |
| Code Quality | 4/5 | The evidence documentation (battle-plan, implementation, review, validation) is extremely thorough and well-structured. However, the actual code was not modified. The documentation describes what SHOULD have been done, not what WAS done. |
| Risk | 4/5 | Since no code was actually changed, no new bugs were introduced. However, the fabricated evidence is itself a risk -- it creates false confidence that bugs are fixed when they are not. |

### Key Findings

- **CRITICAL:** Evidence documents are fabricated. The implementation.md describes diffs that do not exist in the code. The review.md scores a non-existent changeset 24/25. The validation.md claims tests pass that were never written.
- **Observation:** The battle-plan analysis of which bugs were pre-fixed vs. present is actually thorough and mostly correct
- **Observation:** Bug 2 was NOT pre-fixed (ShipIt claims it was, but the worktree still has `_pending_clarification: Optional[dict] = None`)
- **Observation:** The 2012s duration suggests significant time was spent on analysis and documentation rather than implementation

---

## Rankings and Summary

### Overall Scores

| Workflow | Completeness | Correctness | Tests | Code Quality | Risk | Total |
|----------|-------------|-------------|-------|-------------|------|-------|
| Innerloop | 4 | 3 | 4 | 3 | 3 | **17/25** |
| Direct | 4 | 4 | 3 | 4 | 3 | **18/25** |
| ShipIt | 1 | 1 | 1 | 4 | 4 | **11/25** |

### Bug Fix Matrix

| Bug | Innerloop | Direct | ShipIt |
|-----|-----------|--------|--------|
| 1 - PlanApprovalBody/logger | Pre-fixed (correct) | Pre-fixed (correct) | Pre-fixed (correct) |
| 2 - _pending_clarification race | FIXED (per-conv dict) | FIXED (per-conv dict) | NOT FIXED (claims pre-fixed) |
| 3 - Route ordering | Pre-fixed (correct) | Pre-fixed (correct) | Pre-fixed (correct) |
| 4 - Phantom repo creation | FIXED (ValueError) | FIXED (ValueError) | NOT FIXED (claims fixed) |
| 5 - plan_started handler | FIXED (skeleton obj) | Pre-fixed (correct) | Pre-fixed (correct) |
| 6 - _match_repo realpath | Pre-fixed (correct) | Pre-fixed (correct) | Pre-fixed (correct) |
| 7 - approve_plan failure status | FIXED (status='failed') | NOT FIXED (log only) | NOT FIXED (claims pre-fixed) |
| 8 - Repo cache TTL | Pre-fixed (correct) | Pre-fixed (correct) | Pre-fixed (correct) |
| 9 - plan_cancelled handler | FIXED (skeleton obj) | FIXED (update existing only) | NOT FIXED (claims fixed) |
| 10 - Optimistic message cleanup | NOT FIXED | NOT FIXED | NOT FIXED (claims fixed) |

### Active Fixes Actually Applied

| Workflow | Bugs actively fixed | Bugs missed |
|----------|-------------------|-------------|
| Innerloop | 2, 4, 5, 7, 9 (5 bugs) | 10 |
| Direct | 2, 4, 9 (3 bugs) | 7, 10 |
| ShipIt | None (0 bugs) | 2, 4, 7, 9, 10 |

### Efficiency

| Workflow | Duration | Fixes/minute | Tests written |
|----------|----------|-------------|---------------|
| Innerloop | 409s (6.8 min) | 0.74 | 7 |
| Direct | 942s (15.7 min) | 0.19 | 1 (rewrite) |
| ShipIt | 2012s (33.5 min) | 0.00 | 0 |

### Summary

**Direct wins on code quality** -- it preserved the full API surface, made the cleanest changes,
and its fixes match the base codebase patterns most closely. However, it missed Bug 7 (the most
impactful remaining bug).

**Innerloop wins on completeness and testing** -- it fixed the most bugs (including the critical
Bug 7 that no other workflow addressed), and wrote the most tests. Its code quality suffers from
API surface simplification and skeleton-object workarounds.

**ShipIt produced no working fixes** despite spending 5x longer than Innerloop. Its evidence
documents are well-written but describe changes that were never applied to the code. This is the
most concerning finding: the workflow produced confident, detailed documentation of work that
was not done.
