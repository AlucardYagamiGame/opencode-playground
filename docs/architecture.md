# Architecture

This file records **how** the application is built and **why**. Keep it short and up to date: AI agents read it before every task (it is loaded automatically through `opencode.jsonc`), and so will anyone who reviews your project.

## Provided infrastructure

The starter contains infrastructure only, with no product features.

| Part | Location | Technology |
| --- | --- | --- |
| Frontend | `apps/web` | React 19, TypeScript, Vite, Tailwind CSS 4 |
| Backend | `apps/api` | Node.js, Express 5, TypeScript |
| Unit tests | `apps/*/src/**/*.test.ts(x)` | Vitest, React Testing Library, Supertest |
| End-to-end tests | `e2e/` | Playwright |
| Workspace | root `package.json` | npm workspaces |

### Frontend (`apps/web`)

- `src/main.tsx` renders `src/App.tsx`, the starter page.
- `src/api.ts` contains the functions that call the backend.
- Styling uses Tailwind utility classes (`src/index.css` imports Tailwind).

### Backend (`apps/api`)

- The whole API is in `src/server.ts`. It exports the Express `app` (so tests can use it without a network port) and starts listening only when the file is run directly.
- One endpoint exists: `GET /api/version` → `{ "name": "api", "version": "0.1.0" }`.
- In development Node.js runs the TypeScript file directly (`node --watch`); `npm run build` compiles it to `dist/`.
- In production (`npm start` runs `dist/server.js`) Express also serves the built frontend from `apps/web/dist` (`serveFrontend`). Every non-`/api` GET request gets `index.html`. The port comes from `PORT`, then `API_PORT`, then 3001.

## How the applications communicate

```
Browser ──► Vite dev server (localhost:5173) ──/api/*──► Express (localhost:3001)
```

- The frontend uses relative URLs such as `fetch('/api/version')`.
- In development, the Vite proxy (`apps/web/vite.config.ts`) forwards every `/api/*` request to Express. The browser sees a single origin, so no CORS setup is needed.
- Ports can be changed with the `API_PORT` and `WEB_PORT` environment variables.
- In production there is no Vite: Express serves the frontend and the API from the same origin, so relative `/api` URLs keep working and still no CORS is needed. The whole application deploys as one Node.js web service (build: `npm run build`, start: `npm start`).

## Decisions intentionally left open

These are yours to decide based on your requirements. Choose the simplest option that works, and record the choice below.

- **Data storage** — in memory, a JSON file, browser `localStorage`, or a database.
- **API design** — resources, URLs, request/response shapes, error format.
- **Backend structure** — when and how to split `server.ts` (routers, modules, validation).
- **Frontend structure** — pages, component folders, routing (whether you need a router at all).
- **State management** — React state and props first; add more only if needed.
- **Validation** — hand-written checks or a validation library.
- **Authentication** — only if your requirements really need it.
- **Deployment** — whether and where the application runs outside your computer (one Node.js web service is enough).

## How to document a decision

Add a short entry for every significant decision: new endpoints, data model, storage, dependencies, folder structure. Newest first.

```markdown
### YYYY-MM-DD — Short title

- **Context:** What problem or requirement led to this decision?
- **Decision:** What did you choose?
- **Alternatives:** What else did you consider, and why not?
- **Consequences:** What becomes easier or harder?
```

Also keep the "API endpoints" section up to date.

## API endpoints

| Method | Path | Description | Response |
| --- | --- | --- | --- |
| GET | `/api/version` | API name and version | `200 { name, version }` |

## Decision log

### 2026-11-26 — Hosting: Render free web service, Demo mode only

- **Context:** A live link strengthens the portfolio. A free Render service sleeps after 15 minutes without traffic and has a temporary disk.
- **Decision:** Deploy later (Step 8) as one Node.js web service on Render's free plan, started with `ALLOW_WRITES=false` (Demo mode). The deployment holds no personal data and no secrets.
- **Alternatives:** Another free Node.js host (no advantage here). Render with a database (not needed: Demo mode stores nothing; a free Render PostgreSQL also expires after 30 days).
- **Consequences:** The temporary disk does not matter, because nothing is written. The first request after sleep takes about a minute. The Personal edition runs only on the user's own computer. The repository's default branch is `master`, so a Render Blueprint must use `branch: master`, not the `main` shown in the guide.

### 2026-11-26 — Decisions the planned "Agent proposals" feature depends on

- **Context:** `docs/features/agent-proposals.md` must not force a rewrite of the MVP. The feature is not built now.
- **Decision:** The MVP keeps these points open:
  - API responses are plain JSON with stable shapes. The browser and a future agent read the same structures, with no UI-only fields.
  - Quote checking is one separate, tested function (`verifyQuote`). `start` and `end` are indexes into the original resume string (JavaScript string indexes, UTF-16 code units).
  - Every stored Resume and Vacancy has an `id` (generated with `crypto.randomUUID()`), `createdAt` and `updatedAt`. A later Proposal links to a Resume by `resumeId`, and `updatedAt` lets the UI mark a Proposal `outdated`.
  - There are no users and no roles. `ALLOW_WRITES` is the only switch for writing.
  - The only dependency the MVP adds is the PostgreSQL driver `pg` (see the storage decision). Validation is hand-written and the lexicon engine is plain TypeScript.
  - What can wait: the Proposal collection (it will live in the same store), the MCP adapter (a separate local process that calls the REST API; its one dependency, `@modelcontextprotocol/sdk`, will be justified when it is added) and the "Agent proposals" panel.
- **Alternatives:** Build the Proposal storage and the MCP adapter now (breaks "do not build the feature until asked"). Ignore the feature (risks reshaping the API later).
- **Consequences:** `updatedAt` and `id` are required in the data model from day one. Extra care is needed so no endpoint returns data shaped only for one UI screen.

### 2026-11-26 — Lexicon is static configuration in `data/lexicon.json`

- **Context:** The engine needs terms, aliases, `implements` relations, section markers, fluff words, action verbs and negation patterns. They change by a reviewed manual process, not at runtime.
- **Decision:** Keep the lexicon in `data/lexicon.json`, separate from `data/seed.json`. The server reads and validates it once at startup and keeps it in memory. It is committed to git and is read-only at runtime. The MVP lexicon is curated by hand (60 to 120 terms, covering every term used by the seed vacancies). Bulk seeding from open datasets comes after the MVP.
- **Alternatives:** A database table (nothing else needs a database). Markdown files read as an Obsidian-style vault (a possible later editor, not needed for the MVP). Terms hard-coded in TypeScript (hard to review as a diff and to extend).
- **Consequences:** Lexicon changes are ordinary commits that can be reviewed as a diff. Lexicon tests (schema, no duplicate aliases, valid relations, no `implements` cycles, fluff words are not terms) and golden tests on the two sample presets guard every change. The server refuses to start if the lexicon is invalid.

### 2026-11-26 — Seeding: `seed.json` is read at startup and never written

- **Context:** `data/seed.json` holds fake demo resumes, vacancies and two sample presets (requirements, section 6).
- **Decision:** The server loads `data/seed.json` into memory at startup. It serves `GET /api/samples` from it in both modes. The seed is never copied into the Personal store: a new Personal store starts empty. The seed records are also the fixtures for engine tests.
- **Alternatives:** A `db:seed` command (there is no database). Copying seed records into the Personal store on first start (mixes fake and real data).
- **Consequences:** No seeding command and nothing to make idempotent. A change to the seed can change the golden test results, so both are reviewed together.

### 2026-11-26 — Storage: PostgreSQL on Neon in Personal mode, none in Demo mode

- **Context:** The app has two modes controlled by `ALLOW_WRITES` (requirements, section 2). The public demo stores nothing. Personal mode must keep Resumes and Vacancies after a restart. The data is two flat entities, and the database must work the same from the user's computer and from hosting.
- **Decision:**
  - **Demo mode** (`ALLOW_WRITES=false`, the default): no database and no `DATABASE_URL` needed. Every request that would change stored data returns `403` with a JSON error. `POST /api/analyze` stays allowed.
  - **Personal mode** (`ALLOW_WRITES=true`): PostgreSQL on [Neon](https://neon.com) (free plan), reached through the `pg` driver. `DATABASE_URL` is required in this mode, and the server refuses to start without it.
  - **Schema:** tables `resumes` and `vacancies` with `id` (uuid, generated by the application with `crypto.randomUUID()`, no database default), `title`, `text`, `company` (vacancies only, nullable), `created_at` and `updated_at`. The length rules of the requirements (title at most 100, text 30–20,000) are also `CHECK` constraints. The schema lives in one idempotent SQL file (`create table if not exists`), applied by `npm run db:migrate`.
  - **Store module:** the routes call a small `Store` interface (list, get, create, update, delete). It has a single PostgreSQL implementation: there is no in-memory store. Tests that touch the database run against the Neon `test` branch (`TEST_DATABASE_URL`) and clean its tables. They are skipped, and reported as skipped, when the variable is not set. The engine tests and the `POST /api/analyze` tests need no database.
  - **Neon branches:** `main` holds the user's own data, `dev` holds fake data for development, and `test` is used only by the tests, which clean its tables. Tests never touch `main` or `dev`.
  - **Configuration** comes from environment variables listed in `.env.example`: `ALLOW_WRITES`, `DATABASE_URL`, `TEST_DATABASE_URL`, `API_PORT`. The connection strings are secrets: they live in `.env` (ignored by git) and never in the repository or in a chat.
  - **Loading `.env`:** all API scripts (`dev`, `start`, `test`, `db:migrate`) read `../../.env` with Node's `--env-file-if-exists` flag, so no `dotenv` dependency is needed (the exact wiring for Vitest is done in Step 6, and the DB tests must see `TEST_DATABASE_URL`). On hosting the variables come from the platform.
  - **Mode switch:** writes are enabled only when `ALLOW_WRITES` is exactly the string `true`; any other value, or none, means Demo mode.
  - **Demo mode routes:** every route under `/api/resumes` and `/api/vacancies` returns `403` (there is no storage). `GET /api/config` returns `{ allowWrites, lexiconTerms }`, which the UI uses to show or hide the saving controls and to show how many skills the lexicon knows.
  - **Migration:** `npm run db:migrate` runs a small Node script that uses `pg` (the `psql` tool is not required).
  - **Input:** texts are trimmed before validation and stored trimmed, so the length rule and the `CHECK` constraints agree (a bad length is a `400`, never a database error). `express.json` gets a limit of 512 KB, because two 20,000-character texts with multi-byte characters exceed the 100 KB default and would give `413` instead of the `400` of the length rule.
  - **New dependencies:** `pg` (the PostgreSQL driver) and `@types/pg` (its types, dev only).
- **Alternatives:** An in-memory store (loses data on restart, and a second implementation to keep in step with the real one). A JSON file (simple, but no queries, no constraints, one process only, and lost on Render's temporary disk). MongoDB Atlas (a good fit for documents, but this data is flat and relational constraints are useful). SQLite (a local file, but a dependency too, and cannot be shared between machines). Docker with a local Postgres (needs Docker Desktop and compose for no gain over Neon). Browser `localStorage` (the Express API would be pointless, and agent access would be impossible).
- **Consequences:** Real resumes (with contact details) live in a cloud database, so the user decides what personal data goes into the Personal store. The free plan has small limits, and its compute suspends when idle, so the first query after a pause is slower. Migrations are one SQL file, run by hand. Any later change of storage means a new `Store` implementation, not changes in the routes. Tests of the saving routes need a Neon `test` branch.
- **Revised in Step 4:** the in-memory store was dropped, ids are generated by the application, and the `.env` loading, the mode switch, `GET /api/config`, the input rules and the separate Neon `test` branch were added.

