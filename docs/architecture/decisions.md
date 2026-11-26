# Decision log

Every significant decision: endpoints, data model, storage, dependencies, folder structure, tooling. Newest first. An entry is not rewritten later: when a decision changes, add a new entry and a short "Revised in Step N" line to the old one.

```markdown
### YYYY-MM-DD — Short title

- **Context:** What problem or requirement led to this decision?
- **Decision:** What did you choose?
- **Alternatives:** What else did you consider, and why not?
- **Consequences:** What becomes easier or harder?
```

### 2026-11-26 — Tests and E2E never read the personal `.env`

- **Context:** The architecture review of Step 5 found that a `test` script that reads `.env` puts the unit tests into the user's own mode. `server.ts` builds `app` from the environment when it is imported, so with `ALLOW_WRITES=true` the tests would run against the Neon `main` branch, or fail at import when `DATABASE_URL` is missing. Playwright reuses a running `npm run dev`, which reads `.env` as well.
- **Decision:**
  - The `test` script runs `vitest run` without `.env`. The tests see only the shell environment, so the imported `app` is always Demo mode without storage. New route tests call `createApp(testDeps)`.
  - The database tests take their address from one helper, `src/test/db.ts`: `TEST_DATABASE_URL` from the environment (CI), otherwise from the `.env` file parsed with `util.parseEnv`, taking only that variable. Nothing in the tests reads `DATABASE_URL`.
  - E2E: `playwright.config.ts` starts its own servers on ports 3101 (API) and 5273 (web) and sets `ALLOW_WRITES=false` and an empty `DATABASE_URL` in `webServer.env`. A variable from the environment wins over `--env-file-if-exists`, even an empty one (checked on Windows), so no extra script is needed. The other ports keep `reuseExistingServer` from picking up a running `npm run dev`.
  - `.env.example` leaves the connection strings empty and shows fake examples in comments, so a copied file starts in Demo mode without storage instead of pointing at a host that does not exist.
  - `API_PORT` and `WEB_PORT` are shell variables only and are not in `.env.example`: Vite and Playwright read only the shell, so a port in `.env` would move the API and not the proxy.
- **Alternatives:** Keep loading `.env` and override variables in the Vitest config (`DATABASE_URL` would still reach the test process). A separate `.env.test` file (one more secret file). A separate `dev:e2e` script (works, but the environment override already does the job). Variables inline in npm scripts (`VAR=value command` does not work in the Windows shell that npm uses).
- **Consequences:** A test can reach the user's data only by reading `DATABASE_URL` on purpose. E2E runs even while `npm run dev` is open.

### 2026-11-26 — Neon roles and branches for the demo

- **Context:** The demo decision relies on the database refusing writes. A role created in the Neon Console, CLI or API becomes a member of `neon_superuser`, which belongs to `pg_write_all_data` and can write every table (Neon docs, "Manage roles"). A `demo_reader` created on the Roles page would make that guarantee false. A Neon branch also starts as a copy of its parent's data.
- **Decision:**
  - `demo_reader` is created with SQL in the SQL Editor of the `demo` branch and gets only `connect` on the database, `usage` on the schema and `select` on the two tables. `connect` is granted explicitly, as in Neon's read-only recipe, so the role does not depend on the database's default access. A role created with SQL is not a member of `neon_superuser`. Its password is generated (Neon requires at least 60 bits of entropy) from URL-safe characters only, so it fits a connection string without escaping. After creating it, a `select` as `demo_reader` must work and an `insert` must fail with "permission denied"; this is checked once by hand. The SQL is in `data-model.md`.
  - The branches `dev`, `test` and `demo` are created from `main` while `main` is still empty, so no copy of personal data ends up in them.
- **Alternatives:** `neon roles create --access read_only` (one more tool to install for one command). A role from the Console (can write).
- **Consequences:** Setting up the demo takes one SQL snippet instead of a few clicks, and the read-only guarantee holds.

### 2026-11-26 — Smaller fixes from the architecture review

- **Context:** The `architect` review of Step 5 found points a developer would have to guess in Step 6.
- **Decision:**
  - **Offsets.** The length rules use the trimmed texts, but `POST /api/analyze` analyzes the texts exactly as sent, so `start` and `end` point into the string the client shows.
  - **Contract names follow the glossary:** `Requirement` (`MatchedRequirement | Gap | ExcludedRequirement`), `NotChecked`, `Fluff`, `Quote`. `Span` is the shared technical shape of a Quote, a Context and a Fluff sentence, and is in the glossary.
  - **Lexicon types.** `engine/lexicon.ts` holds the lexicon types and the pure `compileLexicon()`; `lexicon/load.ts` only reads, validates and compiles.
  - **Web types.** `apps/web/src/api.ts` re-exports `contract.ts` once with `export type *`; other files import from `api.ts`, so the path to the API is written once.
  - **Errors.** A malformed JSON body is `400 validation_failed` without `field`; a body over the limit is `413 payload_too_large`. `field` stays a plain string, because Agent proposals may later send a path such as `quotes[1]`.
  - **Ordering.** Lists are ordered by `updated_at desc, title`, because `db:seed` inserts in one transaction and every seed record gets the same `now()`.
  - **Database scripts** live in `apps/api/package.json` (forwarded from the root with `-w apps/api`) and run `src/store/*.ts` directly, because the build does not copy `schema.sql` to `dist`. The database tests apply the schema once in a Vitest `globalSetup`, and each test file cleans only its own table, because Vitest runs files in parallel and two `create table if not exists` at the same time can fail.
  - **Use in Analyze.** Loading a resume keeps the exclusions; loading a vacancy clears them, as the requirements say.
  - **Agents ask before `npm run db:*`** in both tools (`opencode.jsonc` and `.claude/settings.json`), because `db:seed` replaces the contents of the `demo` branch.
  - **Build order.** The demo-only parts (the `demo` branch, `demo_reader`, `db:seed`, the secret on Render) come last and are the first to cut if time runs out. The read-only checks in the routes and the UI are built together with CRUD, because adding them later costs more.
- **Alternatives:** Leave these points to Step 6 (each would be decided ad hoc in code, and some only after a bug).
- **Consequences:** `backend.md`, `frontend.md`, `data-model.md`, the requirements (Copy summary format, quote rules) and the glossary were updated in the same change.

### 2026-11-26 — Tests and the build order of Step 6

- **Context:** The engine is the riskiest part, and the reference results in the requirements are the best tests the project has. The database is a cloud service that tests cannot always reach.
- **Decision:**
  - Engine: unit tests per module, plus golden tests that run the two sample presets and the other seed vacancies and compare with "Reference results" in the requirements. Lexicon tests check the rules of requirements section 6.
  - Routes: Supertest against `createApp(deps)`. `analyze`, `samples`, `config` and the Demo mode answers (`403`) always run. The `400` validation tests use a store stub that fails the test if it is called, which proves validation happens before storage (the stub stores nothing, so it is not an in-memory store). CRUD tests run against the Neon `test` branch and are skipped, and reported as skipped, without `TEST_DATABASE_URL`.
  - Web: React Testing Library for pages and components, plain unit tests for `lib/highlight.ts`, `lib/summary.ts` and `lib/coverage.ts`.
  - E2E: Playwright in Demo mode without storage (the API is started with `ALLOW_WRITES=false` and an empty `DATABASE_URL`), so it needs no database and no secret: load a sample, analyze, check the coverage line, switch layers, copy the summary.
  - Build order: lexicon and engine with golden tests, then `GET /api/config`, `GET /api/samples` and `POST /api/analyze`, then the Analyze screen, then the store and the CRUD routes, then the Resumes and Vacancies tabs.
- **Alternatives:** UI first (the engine rules would be discovered late and the UI reworked). Database first (blocks the public demo on the least risky part).
- **Consequences:** The demo works end to end before any database code exists. Saved-record routes are tested only where a `test` branch is configured.
- **Revised in Step 5 (review):** E2E runs on its own ports with the environment set in `playwright.config.ts`, the database tests read only `TEST_DATABASE_URL`, and the demo-only parts come last in the build order. See "Tests and E2E never read the personal `.env`" and "Smaller fixes from the architecture review".

### 2026-11-26 — Frontend: React state and Tailwind, no extra libraries

- **Context:** One screen with several states (Analyze) and two simple list screens. The requirements ask for safe text rendering and exact highlights.
- **Decision:**
  - No router: `App.tsx` switches tabs (Analyze, Resumes, Vacancies) with React state. Nothing needs a deep link, and `serveFrontend` already returns `index.html` for any path, so a router can be added later without server changes.
  - State: `useReducer` in `App.tsx` owns the Analyze state (both texts, the loaded sample or record, exclusions, the result, the layer, the selected requirement), so switching tabs never loses pasted text. The reducer encodes the clearing rules of the requirements.
  - Data: typed `fetch` functions in `src/api.ts` that throw an `ApiError` with `status`, `code` and `field`. No data-fetching library.
  - Forms: controlled inputs, errors placed by the `field` of the API error. No form library.
  - Highlights: `lib/highlight.ts` turns the text and the offset ranges into segments, rendered as React text and `<mark>` elements inside a `whitespace-pre-wrap` block. Never `innerHTML` or `dangerouslySetInnerHTML`.
  - Styling: Tailwind utility classes only, no UI kit.
- **Alternatives:** react-router (no URLs to share), Redux or Zustand (one reducer is enough), TanStack Query (no caching or background refresh needed), React Hook Form (four fields), a component library (more to learn than to gain, and a portfolio benefits from plain code).
- **Consequences:** No new frontend dependency. The reducer is the one place to test the clearing rules. Details in `frontend.md`.

### 2026-11-26 — Lexicon file format

- **Context:** Requirements section 6 lists what the lexicon holds; the engine and the lexicon tests need one exact format.
- **Decision:** `data/lexicon.json` follows the format in `data-model.md`. Every term has a stable `id` (a lowercase slug, used in `excluded` and in tests), a display `name`, optional `aliases`, `caseSensitive` and `implements` (each with a `source` link). Headings, marker words, navigation lines, fluff words, action verbs, negation and aspiration patterns are separate lists. A hand-written validator checks it at startup; the server refuses to start if it is invalid.
- **Alternatives:** A JSON Schema file with a validator library (a dependency for one file). YAML (needs a parser dependency).
- **Consequences:** The format is reviewed as a diff. Renaming a term `id` breaks exclusions of open pages only, which is acceptable because nothing stores them.

### 2026-11-26 — Data model details

- **Context:** The schema of the storage decision left some rules open: how length is counted, who updates `updated_at`, how lists are ordered.
- **Decision:** Lengths count Unicode code points (`[...text].length` in TypeScript), which is what PostgreSQL `char_length` counts, so the application check and the `CHECK` constraint always agree. JavaScript `.length` counts UTF-16 units and would disagree on emoji. The application sets `updated_at` in every `UPDATE` (no trigger). Lists are ordered by `updated_at` descending ("newest first" means the record touched last). Columns are snake_case; the store maps them to camelCase. The full schema is in `data-model.md`.
- **Alternatives:** UTF-16 length (an emoji-heavy text could pass the API and fail the database with a `500`). A database trigger for `updated_at` (logic hidden from the TypeScript code).
- **Consequences:** One helper counts length for every field. Moving to another database keeps the same rule.
- **Revised in Step 5 (review):** lists are ordered by `updated_at desc, title`. See "Smaller fixes from the architecture review".

### 2026-11-26 — API contract and error format

- **Context:** The browser and the planned agent read the same JSON. The UI must place validation errors next to fields.
- **Decision:**
  - Every error is `{ "error": { "code": string, "message": string, "field"?: string } }`. Codes: `validation_failed` (400), `storage_disabled` (403), `writes_disabled` (403), `not_found` (404), `payload_too_large` (413), `internal` (500). A `500` never includes internal details.
  - `POST /api/analyze` takes `{ resume, vacancy, excluded? }` (`excluded` is a list of term ids) and keeps nothing between requests: the client sends the exclusions every time.
  - Lists of saved records return summaries without `text`; `GET /:id` returns the whole record. `DELETE` returns `204`. A malformed id is a `404`, not a database error.
  - The request and response types live in `apps/api/src/contract.ts` (types only). The web app imports them with `import type`, which is erased at build time, so there is one source of truth and no runtime link between the apps.
  - The endpoints and shapes are listed in `backend.md`.
- **Alternatives:** A `packages/contract` workspace (cleaner boundary, more configuration for two apps). Types copied into the web app (they drift apart). Error bodies without a `code` (the UI would parse messages).
- **Consequences:** `contract.ts` must not import Node or Express types. A change to it shows up in both apps' type checks.
- **Revised in Step 5 (review):** the types are named as in the glossary, the web app imports them through one re-export in `src/api.ts`, and a malformed JSON body is `400` without `field`. See "Smaller fixes from the architecture review".

### 2026-11-26 — Backend layers: a pure engine behind thin routes

- **Context:** The engine is deterministic and has the most rules. Routes need different dependencies in tests (Demo, Demo with storage, Personal).
- **Decision:**
  - `apps/api/src/engine/` is plain TypeScript with no Express, no file system and no environment access. It takes the lexicon and the two texts and returns an `Analysis`.
  - Routes are thin: validate, call the engine or the store, send JSON. Errors go to one error middleware.
  - `createApp(deps)` builds the Express app from `{ config, lexicon, samples, store }`. `export const app = createApp(loadDeps())` stays for production and the existing tests; new tests call `createApp` with their own dependencies.
  - The folder layout is in `backend.md`.
- **Alternatives:** Keep everything in `server.ts` (unreadable with seven endpoint groups and an engine). A separate `packages/engine` workspace (the web app does not need the engine, because exclusions are recalculated by the API).
- **Consequences:** Golden tests run in milliseconds without HTTP. The Agent proposals feature can call `verifyQuote` from the engine directly.

### 2026-11-26 — Demo mode reads seed records from a Neon `demo` branch

- **Context:** Without a database the public demo looks like a front end and one function, not a full-stack application. Letting visitors save would make the owner hold strangers' personal data (resumes with phone numbers), show it to other visitors, and invite spam. A second database (MongoDB Atlas) was considered to try a document store.
- **Decision:**
  - Storage is on when `DATABASE_URL` is set (an empty value counts as not set); writes are on when `ALLOW_WRITES` is exactly `true`. This gives three states: no storage (local quick start and E2E), storage read-only (the public demo), storage with writes (Personal mode). `ALLOW_WRITES=true` without `DATABASE_URL` refuses to start.
  - The public demo on Render reads a Neon branch `demo` that holds only the seed resumes and vacancies. It connects with a role that has only `SELECT`, so even a wrong `ALLOW_WRITES` cannot write: the database refuses.
  - `npm run db:seed` applies the schema and replaces the contents of the `demo` branch with the records of `data/seed.json`. It reads only `DEMO_DATABASE_URL`, never `DATABASE_URL`, so it cannot touch the Personal data by mistake.
  - `GET /api/samples` still comes from `data/seed.json`, so "Load sample" works in every state.
- **Alternatives:** No database in the demo (simplest, but the reason for this entry). Public writes (personal data of strangers, GDPR duties, spam). MongoDB Atlas for the demo (a second `Store` implementation, a second driver and test setup for the same entities, the same privacy question; worth trying later as a separate feature, see the backlog in the requirements).
- **Consequences:** Neon has four branches: `main` (own data), `dev`, `test`, `demo`. One more variable, `DEMO_DATABASE_URL`, and one more script. The first demo request after idle is slower, because both Render and Neon wake up. Revises parts of the storage, seeding and hosting entries below.
- **Revised in Step 5 (review):** `demo_reader` is created with SQL, not in the Neon Console, and the branches are created while `main` is empty. See "Neon roles and branches for the demo".

### 2026-11-26 — Architecture docs are a folder

- **Context:** One `architecture.md` grows with every decision, and every agent would load all of it.
- **Decision:** `docs/architecture/` holds `README.md` (overview and cross-cutting rules), `data-model.md`, `backend.md`, `frontend.md` and this `decisions.md`. The engine rules stay in `docs/requirements.md`, because they describe what the product does; `backend.md` describes only how the engine code is organized.
- **Alternatives:** One file (simple, but long and loaded whole). Moving the engine rules here (mixes "what" and "how", and the requirements reviewers would lose them).
- **Consequences:** Every reference to the old file was updated (AGENTS.md, CLAUDE.md, `opencode.jsonc`, `.opencode/`, the PR template). Each agent reads the part it needs.

### 2026-11-26 — Agent tooling: Claude Code and OpenCode side by side

- **Context:** The design work happens in Claude Code. OpenCode with DeepSeek stays for small, fully specified tasks and to show the mentors of the course that the OpenCode workflow was learned. Claude Code reads `CLAUDE.md`, not `AGENTS.md` and not the `instructions` of `opencode.jsonc`.
- **Decision:**
  - A committed `CLAUDE.md` imports `AGENTS.md` and `docs/glossary.md`, so both tools follow the same rules. The requirements and the architecture docs are read on demand, by role, because an import is loaded into every turn of every session and subagent.
  - `opencode.jsonc` loads `docs/requirements.md`, `docs/architecture/README.md`, `docs/architecture/decisions.md` and `docs/glossary.md`, as the guide suggests. The OpenCode agents and commands point to the new files.
  - The `architect` agent is a Claude Code subagent (`.claude/agents/architect.md`) with only Read, Grep and Glob. It checks a plan or a set of changed files against `docs/architecture/` and returns findings and options. The main session writes decisions after the user approves them.
- **Alternatives:** An OpenCode `architect` that edits only `docs/architecture/`, as in the guide (the design work is not done in OpenCode). A Claude Code `architect` with Edit and Write limited by a hook (Claude Code restricts a subagent by its tool list, not by path; the hook adds configuration to debug). Dropping OpenCode (loses the cheap workhorse and the course evidence).
- **Consequences:** When a doc path changes, both `CLAUDE.md`/`AGENTS.md` and `opencode.jsonc`/`.opencode/` are updated together. The read-only architect cannot drift the docs, and only one editor writes each file.
- **Revised in Step 5 (review):** both tools ask before `npm run db:*` (`opencode.jsonc` and a committed `.claude/settings.json`). See "Smaller fixes from the architecture review".


### 2026-11-26 — Hosting: Render free web service, Demo mode only

- **Context:** A live link strengthens the portfolio. A free Render service sleeps after 15 minutes without traffic and has a temporary disk.
- **Decision:** Deploy later (Step 8) as one Node.js web service on Render's free plan, started with `ALLOW_WRITES=false` (Demo mode). The deployment holds no personal data and no secrets.
- **Alternatives:** Another free Node.js host (no advantage here). Render with a database (not needed: Demo mode stores nothing; a free Render PostgreSQL also expires after 30 days).
- **Consequences:** The temporary disk does not matter, because nothing is written. The first request after sleep takes about a minute. The Personal edition runs only on the user's own computer. The repository's default branch is `master`, so a Render Blueprint must use `branch: master`, not the `main` shown in the guide.
- **Revised in Step 5:** the deployment also reads the seed records from the Neon `demo` branch through a role that can only `SELECT`. Its `DATABASE_URL` is a secret set in the Render dashboard. See "Demo mode reads seed records from a Neon `demo` branch".

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
- **Revised in Step 5:** `npm run db:seed` now copies the seed resumes and vacancies into the Neon `demo` branch (never into the Personal store). `GET /api/samples` still reads `data/seed.json`.

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
- **Revised in Step 5:** Demo mode can now have read-only storage (the Neon `demo` branch), so in Demo mode the `GET` routes of `/api/resumes` and `/api/vacancies` work when `DATABASE_URL` is set, and only writes return `403`. `GET /api/config` also returns `storage`. Lengths count Unicode code points.
- **Revised in Step 5 (review):** the `test` script no longer reads `.env` (the database tests read only `TEST_DATABASE_URL` through a helper), `API_PORT` left `.env.example`, and the connection strings in it are empty. See "Tests and E2E never read the personal `.env`".

