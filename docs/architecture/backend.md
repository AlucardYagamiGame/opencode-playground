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
  seed.ts            loads data/seed.json, resolves samples
  lexicon/
    load.ts          reads data/lexicon.json, validates it, builds the matcher
    validate.ts      the lexicon rules (schema, aliases, relations, cycles, fluff vs terms)
  engine/            pure TypeScript: no Express, no fs, no process.env
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
```

Every module has its test next to it (`*.test.ts`). Files appear when the step that needs them is built, not before.

## Rules

- Relative imports use the `.ts` extension; only erasable TypeScript syntax (no `enum`, `namespace`, parameter properties).
- `createApp({ config, lexicon, samples, store })` builds the app; `store` is `null` without storage. `export const app = createApp(loadDeps())` stays for production. `serveFrontend` and the `import.meta.main` block stay as they are.
- `express.json({ limit: '512kb' })`.
- A route validates input, calls the engine or the store, and sends JSON. It contains no engine logic and no SQL.
- `async` handlers throw `HttpError(status, code, message, field?)`; the error middleware turns it into the error body. Any other error is logged and answered with `500 internal`.
- Ids in the path are checked against the UUID format first; a malformed id is `404 not_found`.
- The `pg` Pool is created once from `DATABASE_URL`. The connection string carries `sslmode=require`.
- The engine reads only its arguments. The lexicon is passed in already validated.

## Store

```ts
interface RecordStore<T, Input, Summary> {
  list(): Promise<Summary[]>;                          // ordered by updatedAt desc
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

All types are in `contract.ts`. Errors: `{ error: { code, message, field? } }`.

| Code | Status | When |
| --- | --- | --- |
| `validation_failed` | 400 | Invalid body; `field` names the input (`resume`, `vacancy`, `excluded`, `title`, `company`, `text`) |
| `storage_disabled` | 403 | Any `/api/resumes` or `/api/vacancies` route without storage |
| `writes_disabled` | 403 | `POST`, `PUT`, `DELETE` with read-only storage |
| `not_found` | 404 | Unknown or malformed id, unknown `/api` route |
| `payload_too_large` | 413 | Body over 512 KB |
| `internal` | 500 | Anything unexpected |

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

interface Sample {
  name: string;
  resume: { title: string; text: string };
  vacancy: { company: string | null; title: string; text: string };
}

type RequirementResult =
  | { status: 'matched'; termId: string; term: string; priority: Priority; quote: Span; via: string | null }
  | { status: 'gap'; termId: string; term: string; priority: Priority; reason: GapReason; context: Span | null }
  | { status: 'excluded'; termId: string; term: string; priority: Priority };

interface NotCheckedLine { text: string; section: Priority }
interface FluffSentence extends Span { fluffWords: string[] }
interface Coverage { matched: number; total: number; mustMatched: number; mustTotal: number }

interface Analysis {
  requirements: RequirementResult[]; // in the order of first appearance in the vacancy
  notChecked: NotCheckedLine[];      // in the order of the vacancy
  fluff: FluffSentence[];            // in the order of the resume
  coverage: Coverage | null;         // null when no requirement is left to count
}

interface ResumeSummary { id: string; title: string; createdAt: string; updatedAt: string }
interface VacancySummary { id: string; company: string | null; title: string; createdAt: string; updatedAt: string }
```

- `excluded` must be an array of strings; unknown term ids are ignored (the vacancy may have changed).
- `via` is the name of the tool for a match through `implements`, otherwise `null`. `context` is set for `negated` and `aspirational`, `null` for `no_quote`.
- `quote`, `context` and fluff spans index the resume text exactly as sent, after trimming. The client shows the trimmed text.
- No field is shaped for one screen only: the coverage line, the captions and the Copy summary are built by the client.

## Tests

- Engine: one test file per module, plus `golden.test.ts` that runs the seed vacancies and compares with "Reference results" in the requirements. Every returned quote passes `verifyQuote`.
- Lexicon: `validate.test.ts` for each rule, and a test that the real `data/lexicon.json` is valid.
- Routes: Supertest on `createApp(...)`. Demo answers and `400` validation always run; the `400` tests pass a store stub that fails the test if any method is called. CRUD tests use `TEST_DATABASE_URL` and are skipped, and reported as skipped, without it.
- Scripts: `test`, `db:migrate` and `db:seed` read `../../.env` like `dev` and `start`. The exact wiring for Vitest is done in Step 6.
