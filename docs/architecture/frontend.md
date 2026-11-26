# Frontend (`apps/web`)

Read with `README.md` and `decisions.md`. The screens and their behaviour are in `docs/requirements.md` (stories 1, 2, 6 and 7); the API shapes are in `backend.md`.

## Folders

```
apps/web/src/
  main.tsx              renders App
  App.tsx               loads GET /api/config, owns the Analyze state (useReducer), switches tabs
  api.ts                re-exports the API types, typed fetch functions, ApiError { status, code, field }
  state/
    analyze.ts          the Analyze reducer: texts, loaded record, exclusions, result, layer, selection
  pages/
    AnalyzePage.tsx     the two fields, Load sample, Analyze, the result and the preview
    RecordsPage.tsx     Resumes or Vacancies: list, open, create, edit, delete, Use in Analyze
  components/           small pieces used by the pages (CoverageLine, RequirementList, NotCheckedList,
                        ExcludedList, FluffList, ResumePreview, LayerSwitch, CopySummaryButton, ...)
  lib/
    highlight.ts        text + ranges -> segments
    summary.ts          Analysis + header -> the Copy summary text
    coverage.ts         Coverage -> "N of M requirements matched · must-have X of Y"
```

A component gets its own file when a page would otherwise be hard to read, not in advance.

## Rules

- **Types.** `src/api.ts` re-exports the backend types once: `export type * from '../../api/src/contract.ts'`. Every other file imports them from `api.ts` (for example `import type { Analysis } from '../api.ts'`). Never redefine them.
- **Tabs.** No router. `App.tsx` shows Analyze always, and Resumes and Vacancies only when `config.storage` is true. Create, edit, delete and the Save buttons appear only when `config.allowWrites` is true.
- **State.** The Analyze state lives in `App.tsx`, so changing tabs keeps the pasted texts. The reducer applies the rules of the requirements:
  - editing the resume text clears the result;
  - editing the vacancy text clears the result and the exclusions, and the Copy summary header goes back to "Pasted texts";
  - Load sample replaces both texts (after a confirmation when either field has text), clears the result and the exclusions, and takes the header from the vacancy of the sample;
  - Use in Analyze of a resume replaces only the resume text (after a confirmation when that field has text) and clears the result; the exclusions and the header stay;
  - Use in Analyze of a vacancy replaces only the vacancy text (after a confirmation when that field has text), clears the result and the exclusions, and takes the header from the vacancy;
  - Exclude and Restore send `POST /api/analyze` again with the new `excluded` list.
- **Requests.** Only through `api.ts`, relative `/api/...` URLs, `AbortController` on unmount. An `ApiError` whose `field` is one of the form's fields is shown next to that field; any other error is shown above the form with a Retry button.
- **Highlighting.** `highlight.ts` splits the resume into segments by the ranges of one layer (Evidence: the selected quote or Context; Fluff: all flagged sentences). The preview shows the resume text exactly as typed: the client never trims it, and the offsets of the API point into that same string. It renders the segments as text and `<mark>` elements in a `whitespace-pre-wrap` block. Never `innerHTML` or `dangerouslySetInnerHTML`. Colours: green quote, red Context, yellow fluff, each with a visible label as well as the colour.
- **Copy summary.** `summary.ts` follows the format rules of requirements story 6 (header, coverage line or its empty-state text, sections in a fixed order); the button calls `navigator.clipboard.writeText` and shows "Copied", or an error if the clipboard is not available.
- **Confirmations.** `window.confirm` for replacing text and deleting a record. The title of a new saved record is asked in a small inline form, not `window.prompt`.
- **States.** Every request shows loading, error and empty states. Status changes are announced with `role="status"`.
- **Styling.** Tailwind utility classes, mobile first. No UI kit, no state library, no router, no form library.

## Tests

- `lib/*.test.ts`: plain unit tests (segments at the start, end and edges of the text, empty ranges; the Copy summary of the *Tricky cases* reference result, a header without a company, "Pasted texts", no requirements, all requirements excluded, Not checked and Excluded lines; the coverage line with and without must-have).
- `state/analyze.test.ts`: each clearing rule of the reducer, including that Use in Analyze of a resume keeps the exclusions.
- Pages and components: React Testing Library, queries by role, label or text, `fetch` mocked with `vi.stubGlobal`. Loading, success and error paths for every request.
- E2E (`e2e/`): the Demo flow without storage. `playwright.config.ts` starts its own API and web servers on ports 3101 and 5273 and sets `ALLOW_WRITES=false` and an empty `DATABASE_URL` in `webServer.env`, so a running `npm run dev` is never reused and the E2E run never sees Personal mode. The variables are set there, not inline in an npm script, because `VAR=value command` does not work in the Windows shell that npm uses.
