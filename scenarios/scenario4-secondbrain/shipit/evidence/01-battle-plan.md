# Battle Plan — Second Brain Tab (claude-talk)

## Feature Summary
Add a "Second Brain" tab to claude-talk — a personal knowledge base with note creation/editing, FTS5 full-text search, note embedding via the existing RAG pipeline, and an "Ask Claude" inline Q&A button.

## Architecture Overview

### Backend

**DB Migration**: `db/migrations/0008_second_brain.sql`
- `notes` table: id (TEXT PK), title, content, tags (JSON array), source_url, embedding (BLOB), embedding_model, created_at, updated_at, metadata (JSON)
- `notes_fts` virtual table: FTS5 over title+content+tags with porter tokenizer (matches wiki_pages_fts pattern from 0006)
- Triggers: `notes_ai` (after insert) and `notes_au` (after update) to keep FTS in sync
- Index: `idx_notes_updated` on updated_at DESC

**CRUD helpers** in `claude_talk/db.py`:
- `create_note(note: dict) -> str` — INSERT with UUID, returns id
- `get_notes(limit, offset, tag_filter) -> list[dict]`
- `get_note(note_id) -> dict | None`
- `update_note(note_id, updates) -> None`
- `delete_note(note_id) -> None`
- `search_notes_fts(query, limit) -> list[dict]` — BM25 FTS5 search

**FastAPI Router**: `claude_talk/routers/notes.py`
- `GET /api/notes` — list (paginated, tag filter)
- `POST /api/notes` — create
- `GET /api/notes/{id}` — fetch single
- `PUT /api/notes/{id}` — update
- `DELETE /api/notes/{id}` — delete
- `GET /api/notes/search?q=...` — FTS5 search
- `POST /api/notes/{id}/embed` — trigger embedding via `claude_talk.retrieval.Retriever`

**Embedding pipeline**: On note save (POST/PUT), call `Retriever.ingest(f"note:{id}", content)` asynchronously. Embeddings stored in `rag_chunks` (existing table) with `flavor="note"` and `source_type="note"`.

**Router registration** in `claude_talk/main.py`: add try/import block matching the pattern used for `knowledge_router`, `innerloop_router`, etc.

### Frontend

**Types** in `frontend/src/types.ts`:
```ts
export interface Note {
  id: string;
  title: string;
  content: string;
  tags: string[];
  source_url?: string;
  created_at: string;
  updated_at: string;
  metadata: Record<string, unknown>;
}
export interface NoteSearchResult {
  note: Note;
  snippet: string;
  score: number;
}
```

**API helpers** in `frontend/src/lib/notesApi.ts`:
- `getNotes(params)`, `createNote(data)`, `getNote(id)`, `updateNote(id, data)`, `deleteNote(id)`, `searchNotes(q)`, `embedNote(id)`

**Store slice** in `frontend/src/store/index.ts`:
```ts
interface NotesSlice {
  notes: Record<string, Note>;
  notesSearchResults: NoteSearchResult[];
  setNotes: (notes: Note[]) => void;
  upsertNote: (note: Note) => void;
  removeNote: (id: string) => void;
  setNoteSearchResults: (results: NoteSearchResult[]) => void;
}
```

**Nav Tab** in `frontend/src/App.tsx`:
- Import `BookOpen` from lucide-react
- Extend `NavTab` union: `| "second-brain"`
- Add to `NAV` array: `{ id: "second-brain", Icon: BookOpen, label: "Second Brain" }`
- Add to `TAB_LABELS`
- Add case in `main` switch

**NotesShell** at `frontend/src/components/notes/NotesShell.tsx`:
- Left panel: note list (sorted by updated_at) + search input using FTS endpoint
- Right panel: editor (textarea) with title, tags, content fields
- "Ask Claude" button: sends note content + user question to `POST /api/manager/ask` (or inline via a simple `fetch` to the notes router `POST /api/notes/{id}/ask`)
- Uses CSS custom properties matching the existing design language (var(--surface), var(--accent), etc.)

## Affected Files

### New Files
| File | Type | Role |
|------|------|------|
| `db/migrations/0008_second_brain.sql` | Migration | notes + notes_fts tables |
| `claude_talk/routers/notes.py` | Backend | FastAPI CRUD + search + embed |
| `frontend/src/components/notes/NotesShell.tsx` | Frontend | Full Second Brain UI |
| `frontend/src/lib/notesApi.ts` | Frontend | API client helpers |
| `tests/test_notes.py` | Test | DB + router tests |
| `frontend/src/__tests__/NotesShell.test.tsx` | Test | Component tests |

### Modified Files
| File | Type | Change |
|------|------|--------|
| `claude_talk/db.py` | Backend | Add notes CRUD helpers |
| `claude_talk/main.py` | Backend | Register notes router |
| `frontend/src/App.tsx` | Frontend | Add Second Brain nav tab |
| `frontend/src/types.ts` | Frontend | Add Note, NoteSearchResult types |
| `frontend/src/store/index.ts` | Frontend | Add NotesSlice |
| `frontend/src/lib/api.ts` | Frontend | Optional: could use notesApi.ts instead |

## Quality Gates
| Gate | Command | Applies |
|------|---------|---------|
| Python tests | `.venv/bin/pytest tests/ -q --tb=short` | Yes |
| TypeScript types | `cd frontend && npx tsc --noEmit` | Yes |
| Frontend lint | `cd frontend && npm run lint` | Yes |

## Risk Assessment
- FTS5 trigger maintenance: deletes must also clean notes_fts via `notes_ad` trigger
- Embedding is async fire-and-forget — errors must not crash the create/update flow
- "Ask Claude" feature needs an API key; graceful degradation if key missing
- The `rag_chunks` table already exists from migration 0006 — no schema conflict
