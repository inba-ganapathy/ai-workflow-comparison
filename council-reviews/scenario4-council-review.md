# SFDC Council Review — Scenario 4: Second Brain Feature
# Reviewer: innerloop:innerloop-code-reviewer subagent
# Date: 2026-05-17

## Innerloop (19/25)
- Schema Quality: 4/5 — notes table, FK to repos, FTS5 porter/unicode61, 2 indexes. No triggers (app-level sync like wiki_pages pattern). Missing created_at DESC index.
- API Completeness: 4/5 — Full CRUD + FTS search + RAG /ask. Separate schemas/notes.py. body:dict anti-pattern loses OpenAPI schema generation. FTS escaping incomplete (only double-quotes).
- Frontend Quality: 4/5 — Split-pane, editor, preview, search toggle, Ask Claude modal. Ctrl+S in title but no onKeyDown handler. useEffect stale-closure risk on activeNote.
- Test Coverage: 4/5 — 29 tests across 2 files (DB + API). FTS delete consistency tested. Missing: FTS after update, no embedding tests.
- Integration: 3/5 — Follows wiki_pages FTS pattern. Lazy imports in handlers inconsistent. fire-and-forget embed task has no runtime error logging.

Critical issues:
- asyncio.create_task(embed_note(...)) swallows runtime errors silently (only catches ImportError)
- body: dict loses FastAPI validation and OpenAPI schema

## Direct (21/25)
- Schema Quality: 5/5 — Production-grade: note_folders (self-ref FK), notes (CHECK constraints, word_count), note_links (backlinks, UNIQUE), notes_fts, note_versions. 6 indexes.
- API Completeness: 5/5 — 16 endpoints: CRUD, FTS, daily notes, folders, tags, backlinks, versions, SSE-streaming AI chat. PATCH for partial updates. _note_to_response helper normalizes snake_case.
- Frontend Quality: 4/5 — NotesShell delegates to 5 subcomponents via react-resizable-panels. Cmd+F keyboard shortcut. Cannot verify subcomponents without reading them.
- Test Coverage: 4/5 — 27 tests. Archive exclusion tested. Missing: FTS after delete, versions endpoint, AI chat.
- Integration: 3/5 — FTS rebuild O(n) on every update. notesApi.ts has 23 functions but ~8 call non-existent endpoints (graph, import, attachments, unlinked-mentions) — dead code.

Critical issues:
- INSERT INTO notes_fts(notes_fts) VALUES('rebuild') on every update = O(n) full rebuild
- notesApi.ts references endpoints not in router (getGraphData, importNotes, uploadAttachment, etc.)
- createdAt: number in TypeScript but backend returns ISO strings — type mismatch

## ShipIt (17/25)
- Schema Quality: 3/5 — FTS triggers in migration (cleanest approach, DB-level consistency). embedding BLOB + embedding_model columns. Only 2 indexes.
- API Completeness: 4/5 — CRUD + FTS + Ask Claude + /embed. Proper Pydantic BaseModel params. Non-streaming Claude call. 503/502 error codes.
- Frontend Quality: 3/5 — NotesShell with NoteListItem, AskClaude, NoteEditor sub-components. Auto-save 1200ms debounce. Zustand store integration. AskClaude uses raw fetch instead of notesApi client — inconsistent.
- Test Coverage: 4/5 — 23 tests. FTS update/delete consistency tested. Missing: tags, /ask, /embed endpoints.
- Integration: 3/5 — Trigger-based FTS inconsistent with rest of codebase (app-level). Phrase-match wrapping f'"{safe_query}"' changes search semantics silently.

Critical issues:
- f'"{safe_query}"' forces phrase-match — "machine learning" only finds exact phrase, not both words
- notes store keyed by id (object) but listing endpoint returns array — shape mismatch
- AskClaude uses raw fetch instead of notesApi

## Rankings
1. Direct — 21/25 — Most complete feature. Production-grade schema. FTS rebuild and dead API client are real issues.
2. Innerloop — 19/25 — Clean focused implementation. Good RAG integration. Minor pattern issues.
3. ShipIt — 17/25 — Best FTS consistency (triggers) but phrase-match bug changes behavior. Minimal feature scope.

## Feature Completeness vs Prompt
| Requirement | Innerloop | Direct | ShipIt |
|-------------|-----------|--------|--------|
| Notes CRUD | Yes (5 endpoints) | Yes (5 + PATCH) | Yes (5 endpoints) |
| FTS search | Yes | Yes | Yes (phrase-match only) |
| Second Brain tab | Yes | Yes | Yes |
| Embedding/semantic | Yes (notes_embedding.py) | Partial (fire-and-forget) | Partial (DB columns, no query path) |
| Ask Claude | Yes (modal→Manager) | Yes (SSE streaming) | Yes (non-streaming) |
| Tests | 29 (2 files) | 27 (1 file) | 23 (1 file) |
| Folders | No | Yes | No |
| Backlinks | No | Yes | No |
| Daily notes | No | Yes | No |
| Versions | No | Yes | No |
| FTS DB triggers | No | No | Yes |
