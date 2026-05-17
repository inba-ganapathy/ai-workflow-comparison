# Battle Plan — ShipIt Bugfix Scenario 2

## Bug Summary

Ten pre-identified bugs span two Python backend files, one Python DB module, and two TypeScript frontend files. Bugs range from a module-import crash (manager.py had missing class/logger in older commit state, now verified present), a race condition in repo_manager.py with `_pending_clarification` as a single dict across conversations, routing/ordering issues, a DB phantom-repo creation path, missing WS event handlers in the frontend store, a path-matching guard in session_watcher.py, a missing done_callback on an asyncio task, an overly-aggressive repo cache TTL, and an optimistic-message cleanup gap in ManagerChat.tsx.

## Root Causes Per Bug

### CRITICAL

**Bug 1 — manager.py: PlanApprovalBody + logger missing**
- File: `claude_talk/routers/manager.py`
- Status: ALREADY FIXED in current code (PlanApprovalBody at line 135, logger at line 12)
- Confidence: HIGH — confirmed by reading file

**Bug 2 — repo_manager.py: `_pending_clarification` race condition**
- File: `claude_talk/agents/repo_manager.py`
- Line 111: `self._pending_clarifications: dict[str, Optional[dict]] = {}`
- Status: ALREADY FIXED in current code — it's a dict keyed by conv_id
- Original bug was `self._pending_clarification: Optional[dict] = None` (single shared state)
- Confidence: HIGH — the fix is already applied

### MAJOR

**Bug 3 — manager.py: Deprecated route registered after `{conv_id}` route → unreachable**
- File: `claude_talk/routers/manager.py`
- Issue: No deprecated route exists in current file — the original bug was `GET /conversations/deprecated` registered after `GET /conversations/{conv_id}`, making it shadow-captured by the param route
- Status: NOT PRESENT in current file — already resolved or never existed in this version
- Confidence: HIGH

**Bug 4 — db.py: `create_conversation()` creates phantom repos with UUID path**
- File: `claude_talk/db.py`, lines 703-715
- Issue: `get_or_create_conversation()` at line 677 inserts a phantom repo row with `path=repo_id` (a UUID) if the repo doesn't exist. `create_conversation()` at line 706 raises `ValueError` if repo not found — correct. The phantom creation is in `get_or_create_conversation` only.
- Fix: `get_or_create_conversation` should raise ValueError instead of creating phantom repos — or at minimum not insert a row with `path=repo_id` (UUID path is nonsense)
- Confidence: HIGH — line 690-693 clearly creates phantom with `path=repo_id`

**Bug 5 — store/index.ts: `plan_started` WS handler drops plan if not in store**
- File: `frontend/src/store/index.ts`, lines 723-737
- Issue: When `plan_started` arrives and `existing` is falsy, it fetches from API — this is actually handled. However on line 727 `if (existing)` when plan not in store yet, it falls into the else branch which does an API fetch — this is correct behavior. Actually examining more carefully: the `plan_started` handler at lines 723-737 already handles the missing-plan case by fetching from API. Status: ALREADY HANDLED.
- However, looking again at the `plan_cancelled` handler at lines 739-747: when `existing` is falsy (plan not in store), it silently does nothing — the cancelled status is never applied. This is Bug 9 (see below).
- For `plan_started` specifically the handler IS complete. Status: ALREADY HANDLED.

**Bug 6 — session_watcher.py: `_match_repo()` missing realpath + os.sep guard**
- File: `claude_talk/watchers/session_watcher.py`, lines 47-74
- Status: ALREADY FIXED in current code — `os.path.realpath` is called on both paths (lines 56, 66), and `os.sep` guard is applied (line 70)
- Confidence: HIGH

**Bug 7 — manager.py: `approve_plan` task has no done_callback**
- File: `claude_talk/routers/manager.py`, lines 160-164
- Status: ALREADY FIXED in current code — `_log_approve_err` done_callback is registered at line 163
- Confidence: HIGH

**Bug 8 — session_watcher.py: `_REPO_CACHE_TTL = 30.0`**
- File: `claude_talk/watchers/session_watcher.py`, line 32
- Status: ALREADY FIXED — value is `5.0` seconds (was `30.0` in the bug report)
- Confidence: HIGH

### MINOR

**Bug 9 — store/index.ts: `plan_cancelled` WS event has no handler for unknown plan**
- File: `frontend/src/store/index.ts`, lines 739-747
- Issue: When `plan_cancelled` arrives for a plan not in the store, it silently drops the event. Unlike `plan_started` which fetches from API on cache miss, `plan_cancelled` does nothing. The plan status will be stale if the user's client missed `plan_proposed`.
- Fix: Add an API fetch fallback similar to `plan_started` case
- Confidence: HIGH

**Bug 10 — ManagerChat.tsx: optimistic message no cleanup on failure**
- File: `frontend/src/components/manager/ManagerChat.tsx`, lines 557-597
- Issue: The `catch` block at line 588 calls `removeManagerMessage` only if both `tempId` and `resolvedConvId` are set. But `tempId` is set AFTER `resolvedConvId` is obtained from the API, so if the API call itself fails, `tempId` is null and the removal branch is dead code in that case. Looking more carefully: `resolvedConvId` comes from `result.conversation_id` BEFORE the optimistic message is added. So if API fails, `tempId` is null (never set), and `resolvedConvId` is also null — so cleanup branch `if (tempId && resolvedConvId)` is unreachable on API failure, which is actually CORRECT behavior (no temp message to clean up). The real bug is: if the API succeeds but then some later operation fails, the tempId is set but the catch block may not fire. Actually on re-reading: the temp message is added inside the try block AFTER the API call. So if the API call throws, tempId is null — no cleanup needed. The catch block is correct. However, there IS a subtle bug: `setSending(false)` is in `finally` but `setManagerStatus("idle")` is only in the catch block — if the API succeeds but the addManagerMessage or setActiveConvId throws (unlikely), managerStatus stays "thinking" forever. Fix: always reset managerStatus in finally.
- Confidence: MEDIUM

## Summary of What Actually Needs Fixing

After code review, the current state of the files shows:

| Bug | Status | Fix Needed |
|-----|--------|-----------|
| 1 - PlanApprovalBody/logger missing | Already fixed in current code | None |
| 2 - _pending_clarification race | Already fixed in current code | None |
| 3 - Deprecated route unreachable | Not present in current code | None |
| 4 - Phantom repo creation | BUG PRESENT in get_or_create_conversation | Fix get_or_create_conversation to raise ValueError |
| 5 - plan_started drops plan | Already handled (API fetch fallback exists) | None |
| 6 - _match_repo missing realpath/sep | Already fixed in current code | None |
| 7 - approve_plan no done_callback | Already fixed in current code | None |
| 8 - REPO_CACHE_TTL = 30.0 | Already fixed (5.0 in current code) | None |
| 9 - plan_cancelled no handler | BUG PRESENT | Add API fetch fallback |
| 10 - optimistic message no cleanup | PARTIAL BUG — managerStatus stuck on late failure | Fix finally block |

## Affected Files

### Source Files
| File | Type | Role in Fix |
|------|------|-------------|
| `claude_talk/db.py` | Backend | Fix get_or_create_conversation phantom repo creation |
| `frontend/src/store/index.ts` | Frontend | Add plan_cancelled API fetch fallback |
| `frontend/src/components/manager/ManagerChat.tsx` | Frontend | Fix managerStatus cleanup in sendMessage |

### Test Files
| File | Type | Status |
|------|------|--------|
| `tests/test_db.py` | Backend Test | Needs new test for phantom repo bug |
| `tests/test_api.py` | Backend Test | Needs new test for manager conversation endpoint |

## Fix Strategy

1. Fix `db.py` `get_or_create_conversation` — raise ValueError for unknown repo_id instead of creating phantom row
2. Fix `store/index.ts` `plan_cancelled` handler — add API fetch fallback when plan not in store
3. Fix `ManagerChat.tsx` `sendMessage` — move `setManagerStatus("idle")` to `finally` block so it always resets
4. Write failing tests for bug 4 before fixing
5. Validate all tests pass

## Engineers Needed
- [x] Backend Engineer (Python)
- [x] Frontend Engineer (TypeScript)

## Setup Requirements
- Python venv: `/Users/inbasagar.ganapathy/eng/claude-talk/.venv/bin/python`
- Tests: `pytest tests/test_api.py tests/test_db.py tests/test_hooks.py -q --tb=short`
- TypeScript: `cd frontend && npx tsc --noEmit`

## Quality Gates
| Gate | Command | Applies |
|------|---------|---------|
| Python tests | pytest tests/test_api.py tests/test_db.py tests/test_hooks.py -q --tb=short | Yes |
| TypeScript types | cd frontend && npx tsc --noEmit | Yes |

## Risk Assessment
- Bug 4 fix changes `get_or_create_conversation` behavior — callers that relied on phantom creation will now get ValueError. Need to check all callers.
- Bug 9 fix adds async dynamic import in a synchronous switch case — existing pattern in the file already does this for `plan_started` so it's safe.
- Bug 10 fix moves status reset to finally — no behavioral change in happy path, only improvement in error path.
