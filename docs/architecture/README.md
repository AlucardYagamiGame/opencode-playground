# Architecture

This folder records **how** TailorProof is built and **why**. What the product does is in `docs/requirements.md`, the words it uses are in `docs/glossary.md`.

| File | Contents | Who reads it |
| --- | --- | --- |
| `README.md` (this file) | Parts of the system, modes, cross-cutting rules | Every agent, before every change |
| `decisions.md` | The decision log, newest first | Every agent, before every change |
| `data-model.md` | Tables, Neon branches, lexicon and seed formats | Backend work, lexicon work |
| `backend.md` | `apps/api` layout, API contract, engine modules, tests | Backend work |
| `frontend.md` | `apps/web` layout, state, highlighting, tests | Frontend work |

Claude Code reads these files on demand (see `CLAUDE.md`). OpenCode loads this file and `decisions.md` into every session through `opencode.jsonc`. Keep this file short.

## Parts of the system

| Part | Location | Technology |
| --- | --- | --- |
| Frontend | `apps/web` | React 19, TypeScript, Vite, Tailwind CSS 4 |
| Backend and engine | `apps/api` | Node.js 22, Express 5, TypeScript, `pg` |
| Lexicon | `data/lexicon.json` | JSON, read once at startup |
| Seed | `data/seed.json` | JSON: demo records and the two sample presets |
| Database | Neon PostgreSQL | Branches `main`, `dev`, `test`, `demo` |
| Unit tests | `apps/*/src/**/*.test.ts(x)` | Vitest, React Testing Library, Supertest |
| End-to-end tests | `e2e/` | Playwright |

```
Browser ──► Vite dev server (:5173) ──/api/*──► Express (:3001) ──► engine (pure TypeScript)
                                                     │
                                                     └──► Store ──► Neon PostgreSQL (optional)
```

- The frontend uses relative URLs (`fetch('/api/...')`). In development the Vite proxy forwards `/api/*` to Express, so there is one origin and no CORS. Ports: `API_PORT`, `WEB_PORT`.
- In production (`npm run build`, then `npm start`) Express serves the built frontend from `apps/web/dist` and the API on one port (`PORT`, then `API_PORT`, then 3001). The whole application is one Node.js web service.

## Modes

Two environment variables decide what the application may do:

| `DATABASE_URL` | `ALLOW_WRITES` | State | Used for |
| --- | --- | --- | --- |
| not set | not `true` | Demo, no storage | Local quick start, E2E tests |
| set | not `true` | Demo, read-only storage | The public demo (Neon `demo` branch, `SELECT`-only role) |
| set | `true` | Personal mode | The user's own computer (Neon `main` or `dev`) |
| not set | `true` | Refuses to start | — |

- An empty value counts as not set. Writes are on only when `ALLOW_WRITES` is exactly the string `true`.
- `POST /api/analyze`, `GET /api/samples`, `GET /api/config` and `GET /api/version` work in every state.
- Without storage every route under `/api/resumes` and `/api/vacancies` returns `403 storage_disabled`. With read-only storage the `GET` routes work and the others return `403 writes_disabled`.
- `GET /api/config` returns `{ allowWrites, storage, lexiconTerms }`; the UI shows the tabs and the save controls from it.

## Cross-cutting rules

- **Errors.** Every error response is `{ "error": { "code", "message", "field"? } }` with the codes listed in `backend.md`. A `500` never shows internal details.
- **Validation.** Hand-written, in `apps/api/src/validation.ts`, before any engine or store call. Texts are trimmed first; lengths count Unicode code points.
- **Configuration.** Only from environment variables, parsed once in `apps/api/src/config.ts`. The list and fake values are in `.env.example`. The API scripts read `.env` with Node's `--env-file-if-exists`; there is no `dotenv`.
- **Secrets.** Connection strings live in `.env` (ignored by git) or in the hosting dashboard, never in the repository or in a chat.
- **Fail fast.** The server refuses to start when the lexicon is invalid or `ALLOW_WRITES=true` has no `DATABASE_URL`.
- **Safe text.** User text is never rendered as HTML.
- **Dependencies.** The MVP adds only `pg` and `@types/pg`. Any other dependency needs a new entry in `decisions.md`.
- **Planned feature.** `docs/features/agent-proposals.md` is not built in the MVP, but the API keeps stable JSON shapes for it and `verifyQuote` stays a separate, tested function.

## API endpoints

| Method | Path | Description | Response |
| --- | --- | --- | --- |
| GET | `/api/version` | API name and version | `200 { name, version }` |

Planned endpoints and their shapes are in `backend.md`. Move a row here when the endpoint is implemented.

## How to document a decision

Add an entry to `decisions.md` for every significant decision: new endpoints, data model, storage, dependencies, folder structure. Then update the file of this folder that the decision touches.
