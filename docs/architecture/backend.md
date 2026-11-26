# Backend (`apps/api`)

Read with `README.md`, `decisions.md` and `data-model.md`. The engine rules themselves are in `docs/requirements.md` ("Engine rules"); this file says where the code for them lives.

## Folders

```
apps/api/src/
  server.ts          createApp(deps), export const app, serveFrontend, startup (import.meta.main)
  config.ts          reads the environment once: allowWrites, databaseUrl, port
  contract.ts        request and response types only, shared with apps/web
  errors.ts          HttpError and the one error middleware
  validation.ts      hand-written validators, length in code points
  seed.ts            loads data/seed.json, resolves samples (a sample without its seed record stops the start)
  lexicon/
    load.ts          reads data/lexicon.json, validates it, calls compileLexicon()
    validate.ts      the lexicon rules (schema, aliases, relations, cycles, fluff vs terms)
  engine/            pure TypeScript: no Express, no fs, no process.env
    lexicon.ts       the LexiconData type (the JSON), the Lexicon type (compiled) and compileLexicon()
    normalize.ts     one-for-one character normalization
    sentences.ts     the one sentence splitter (quotes and fluff)
    mentions.ts      whole-token matching, longest match, caseSensitive
    sections.ts      headings, sections, navigation lines, priority
    negation.ts      negation and aspiration of a mention
    requirements.ts  detect requirements in the vacancy, Not checked lines
    matching.ts      choose the mention: Matched or Gap with reason and Context
    fluff.ts         flagged sentences and their fluff words
    coverage.ts      the coverage numbers
    verify-quote.ts  verifyQuote(resume, quote)
    analyze.ts       analyze(lexicon, resume, vacancy, excluded): Analysis
  routes/
    meta.ts          GET /api/config, GET /api/samples
    analyze.ts       POST /api/analyze
    resumes.ts       /api/resumes
    vacancies.ts     /api/vacancies
  store/
    store.ts         the Store interface
    postgres.ts      the only implementation (pg Pool)
    schema.sql       tables (see data-model.md)
    migrate.ts       npm run db:migrate (DATABASE_URL)
    seed-demo.ts     npm run db:seed (DEMO_DATABASE_URL)
  test/
    db.ts            the address of the database tests (TEST_DATABASE_URL only)
    global-setup.ts  applies schema.sql once before the database tests
```

Every module has its test next to it (`*.test.ts`). Files appear when the step that needs them is built, not before. `tsconfig.build.json` excludes `src/test/` as well as the test files.

## Rules

- Relative imports use the `.ts` extension; only erasable TypeScript syntax (no `enum`, `namespace`, parameter properties).
- `createApp({ config, lexicon, samples, store })` builds the app; `store` is `null` without storage. `export const app = createApp(loadDeps())` stays for production. `serveFrontend` and the `import.meta.main` block stay as they are.
- `express.json({ limit: '512kb' })`.
- A route validates input, calls the engine or the store, and sends JSON. It contains no engine logic and no SQL.
- `async` handlers throw `HttpError(status, code, message, field?)`; the error middleware turns it into the error body. It also maps the two errors of the JSON body parser: a malformed body (`type: 'entity.parse.failed'`) is `400 validation_failed` without `field`, and a body over the limit (`type: 'entity.too.large'`) is `413 payload_too_large`. Any other error is logged and answered with `500 internal`.
- Ids in the path are checked against the UUID format first; a malformed id is `404 not_found`.
- The `pg` Pool is created once from `DATABASE_URL`. The connection string carries `sslmode=require`.
- The engine reads only its arguments. `lexicon/load.ts` passes it a lexicon that is already validated and compiled by `engine/lexicon.ts`, so the golden tests can compile the real lexicon without the server.

## Scripts

| Script | Where | Reads `.env` | Runs |
| --- | --- | --- | --- |
| `dev`, `start` | `apps/api/package.json` | yes | `src/server.ts` (watch), `dist/server.js` |
| `db:migrate`, `db:seed` | `apps/api/package.json`, forwarded by the root `package.json` with `-w apps/api` | yes | `src/store/migrate.ts`, `src/store/seed-demo.ts` with Node directly: the build copies only TypeScript, so `schema.sql` is not in `dist` |
| `test` | `apps/api/package.json` | **no** | `vitest run` |

`.env` is read with Node's `--env-file-if-exists=../../.env`. A variable that is already in the environment wins over the file, even when its value is empty.

## Store

```ts
interface RecordStore<T, Input, Summary> {
  list(): Promise<Summary[]>;                          // ordered by updatedAt desc, then title
  get(id: string): Promise<T | null>;
  create(input: Input): Promise<T>;
  update(id: string, input: Input): Promise<T | null>; // replaces the whole record
  delete(id: string): Promise<boolean>;
}

interface Store {
  resumes: RecordStore<Resume, ResumeInput, ResumeSummary>;
  vacancies: RecordStore<Vacancy, VacancyInput, VacancySummary>;
}
```

The routes check the mode before calling the store (`403` first, then `400`, then the store).

## API contract

All types are in `contract.ts`, named as in `docs/glossary.md`. Errors: `{ error: { code, message, field? } }`.

| Code | Status | When |
| --- | --- | --- |
| `validation_failed` | 400 | Invalid body; `field` names the input (`resume`, `vacancy`, `excluded`, `title`, `company`, `text`). A body that is not valid JSON has no `field`. |
| `storage_disabled` | 403 | Any `/api/resumes` or `/api/vacancies` route without storage |
| `writes_disabled` | 403 | `POST`, `PUT`, `DELETE` with read-only storage |
| `not_found` | 404 | Unknown or malformed id, unknown `/api` route |
| `payload_too_large` | 413 | Body over 512 KB |
| `internal` | 500 | Anything unexpected |

`field` is a plain string. The planned Agent proposals may set it to a path such as `quotes[1]`, so the client compares it with its own field names and shows any other value above the form.

| Method | Path | Request | Response |
| --- | --- | --- | --- |
| GET | `/api/version` | — | `200 { name, version }` |
| GET | `/api/config` | — | `200 { allowWrites, storage, lexiconTerms }` |
| GET | `/api/samples` | — | `200 Sample[]` |
| POST | `/api/analyze` | `{ resume, vacancy, excluded? }` | `200 Analysis` |
| GET | `/api/resumes` | — | `200 ResumeSummary[]` |
| GET | `/api/resumes/:id` | — | `200 Resume` |
| POST | `/api/resumes` | `{ title, text }` | `201 Resume` |
| PUT | `/api/resumes/:id` | `{ title, text }` | `200 Resume` |
| DELETE | `/api/resumes/:id` | — | `204` |
| GET, POST, PUT, DELETE | `/api/vacancies[/:id]` | `{ company?, title, text }` | Same as resumes, with `Vacancy` and `VacancySummary` |

```ts
type Priority = 'must' | 'nice';
type GapReason = 'no_quote' | 'negated' | 'aspirational';
interface Span { text: string; start: number; end: number } // text === source.slice(start, end)
type Quote = Span;

interface Sample {
  name: string;
  resume: { title: string; text: string };
  vacancy: { company: string | null; title: string; text: string };
}

interface RequirementBase { termId: string; term: string; priority: Priority }
interface MatchedRequirement extends RequirementBase { status: 'matched'; quote: Quote; via: string | null }
interface Gap extends RequirementBase { status: 'gap'; reason: GapReason; context: Span | null }
interface ExcludedRequirement extends RequirementBase { status: 'excluded' }
type Requirement = MatchedRequirement | Gap | ExcludedRequirement;

interface NotChecked { text: string; section: Priority }
interface Fluff extends Span { fluffWords: string[] }
interface Coverage { matched: number; total: number; mustMatched: number; mustTotal: number }

interface Analysis {
  requirements: Requirement[]; // in the order of first appearance in the vacancy
  notChecked: NotChecked[];    // in the order of the vacancy
  fluff: Fluff[];              // in the order of the resume
  coverage: Coverage | null;   // null when no requirement is left to count
}

interface ResumeSummary { id: string; title: string; createdAt: string; updatedAt: string }
interface VacancySummary { id: string; company: string | null; title: string; createdAt: string; updatedAt: string }
```

- `excluded` must be an array of strings; unknown term ids are ignored (the vacancy may have changed).
- `via` is the name of the tool for a match through `implements`, otherwise `null`. `context` is set for `negated` and `aspirational`, `null` for `no_quote`.
- The length rules use the trimmed texts, but `POST /api/analyze` analyzes the texts exactly as sent. `quote`, `context` and fluff spans index the resume string of the request, so the client highlights its own text without trimming it.
- `fluffWords` lists each fluff word of the sentence once, written as in the lexicon, in the order of its first appearance in the sentence.
- `verifyQuote` returns `null` for an empty or whitespace-only quote.
- A vacancy `company` that is missing, empty or whitespace-only is stored and returned as `null`.
- No field is shaped for one screen only: the coverage line, the captions and the Copy summary are built by the client.

## Tests

- Engine: one test file per module, plus `golden.test.ts` that compiles the real lexicon, runs the seed vacancies and compares with "Reference results" in the requirements. Every returned quote passes `verifyQuote`.
- Lexicon: `validate.test.ts` for each rule, and a test that the real `data/lexicon.json` is valid.
- Routes: Supertest on `createApp(testDeps)`. Only the existing `/api/version` test imports `app`. Demo answers and `400` validation always run; the `400` tests pass a store stub that fails the test if any method is called.
- The `test` script never reads `.env`, so the test process does not see `DATABASE_URL` or `ALLOW_WRITES`, and the imported `app` is always Demo mode without storage.
- Database tests take their address only from `src/test/db.ts`: `TEST_DATABASE_URL` from the environment (CI), otherwise from the `.env` file parsed with `util.parseEnv`, taking only that one variable. They are skipped, and reported as skipped, without it.
- `apps/api/vitest.config.ts` runs `src/test/global-setup.ts`, which applies `schema.sql` once when `TEST_DATABASE_URL` is set. Vitest runs test files in parallel, and two `create table if not exists` at the same time can fail, so no test file applies the schema itself. Each database test file cleans only the table it tests.
