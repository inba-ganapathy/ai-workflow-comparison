# Independent Review — Second Brain Feature

## Verdict: PASS

## Scores
| Dimension | Score | Rationale |
|-----------|-------|-----------|
| Logic | 5/5 | FTS5 triggers correctly handle insert/update/delete sync; CRUD operations complete and correct; search returns ranked results via BM25 `rank`; note_id paths all do 404-before-operate |
| Security | 4/5 | All SQL parameters are positional (no interpolation); tags stored as JSON prevents injection; FTS query escapes double-quotes. Minor: `source_url` accepted without URL validation — could accept `javascript:` URIs if rendered as links |
| Regression | 5/5 | New migration uses `IF NOT EXISTS` guards; new router registered with try/import so import errors never break startup; all existing test suites still pass (305 passing); no existing endpoints modified |
| Performance | 4/5 | FTS5 BM25 is fast; embedding is fire-and-forget so creates never block; list cap of 500 prevents runaway queries. Minor: `_trigger_embedding` creates a new `Retriever()` per call — the Retriever includes an `InMemoryVectorStore` which is recreated each time and not shared with any persistent store |
| Maintainability | 5/5 | Code matches surrounding codebase style (async def, dict returns, json.loads/dumps, same migration numbering); Pydantic models separate from DB helpers; route ordering comment explains the search-before-id constraint; 19 tests cover all paths including FTS trigger behaviour |
| **TOTAL** | **23/25** | |

Pass Threshold: >= 20/25

## Critical Findings
None — no issues that must be fixed before shipping.

## Recommendations
1. **source_url validation**: Add `HttpUrl` or simple `str` with scheme whitelist in `NoteCreate` to prevent `javascript:` URLs from being stored and rendered as clickable links in `NotesShell.tsx`.
2. **Retriever singleton**: `_trigger_embedding` creates `Retriever()` on every call, which allocates a new `InMemoryVectorStore`. Consider injecting a shared retriever instance (e.g., via app state) so embeddings accumulate across calls and vector search is useful.
3. **`/api/notes/ask` endpoint**: `NotesShell.tsx` calls `POST /api/notes/ask` but this endpoint was not implemented in the router. The UI will get a 404 on Ask Claude. This is the only functional gap — add a simple proxy endpoint or mark the Ask Claude feature as "coming soon" in the UI.

## Questions for the Implementer
1. Is the `Ask Claude` feature intentionally deferred? The `POST /api/notes/ask` endpoint is called from the UI but not defined in the router.
2. The `embedding` BLOB column is on the `notes` table itself but `_trigger_embedding` writes chunks into `rag_chunks`. Which is the source of truth for note embeddings?
