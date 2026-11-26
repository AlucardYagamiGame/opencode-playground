# Product Requirements

> **Status:** draft. Sections 1–4 are agreed. Sections 5–7 are a first draft: review them in Step 4.

This file describes **what** TailorProof must do. AI agents read it before planning or implementing anything (it is loaded automatically through `opencode.jsonc`). Technical decisions (storage, hosting, folder structure) live in `docs/architecture.md`. Domain terms are defined in `docs/glossary.md`: use them exactly.

---

## 1. Product overview

**TailorProof** checks a resume against a specific job posting without an LLM and without paid APIs. It shows which requirements of the vacancy the resume covers (each with an exact quote from the resume), which it does not (gaps), and which sentences make an empty claim (fluff).

Problem: when tailoring a resume to a specific posting, it is easy to miss requirements or to leave fluff (often added by AI tools) that cannot be backed up in an interview.

The analysis is deterministic: it is based on a word list (the lexicon), not on a language model. Every result can be explained and tested. The idea is inspired by the open-source project pitches-peaches (evidence with quotes, no fake-precision scores); TailorProof replaces the LLM with a lexicon engine and adds a web UI and a fluff linter.

## 2. Users

A job seeker who tailors one resume to many vacancies. The texts are in English. There is a single user and there are no accounts.

The application runs in two modes, controlled by `ALLOW_WRITES`:

- **Demo mode** (`ALLOW_WRITES=false`, the default): nothing is stored. The user pastes texts or loads a sample and gets an analysis. This is what a public deployment runs.
- **Personal mode** (`ALLOW_WRITES=true`): resumes and vacancies can be saved and analyzed again later.

## 3. User stories

1. As a job seeker, I want to paste a resume and a vacancy and see the analysis at once, so that I get an answer without saving anything.
2. As a job seeker, I want to see the requirements found in the vacancy, split into must-have and nice-to-have, and exclude terms that are not real requirements, so that the result reflects what the posting really asks for.
3. As a job seeker, I want each requirement marked as matched (with an exact quote from my resume) or as a gap with a reason, so that I see what is proven and what is missing.
4. As a job seeker, I want sentences in my resume that make an empty claim highlighted, so that I can replace them with facts.
5. As a job seeker, I want to save resumes and vacancies and analyze a saved pair, so that I can reuse them (personal mode only).

## 4. Scope

### In scope (MVP)

- Analysis of pasted resume and vacancy text (`POST /api/analyze`, nothing stored).
- Requirement extraction by scanning the vacancy for terms from the lexicon, with aliases, one-hop `implements` relations and must/nice priority.
- Matched requirements with an exact quote and its position in the resume; gaps with a reason.
- Coverage shown as two numbers.
- Not checked: lines of the vacancy's requirement sections in which the lexicon found no term, listed as written, so a gap in the lexicon is never silent.
- Fluff highlighting in the resume. Highlights are shown one layer at a time (Evidence or Fluff) in a read-only preview of the analyzed resume.
- A "Copy summary" button that copies a plain-text report to the clipboard.
- Two sample presets (`GET /api/samples`) built from `data/seed.json`.
- Saving, editing and deleting resumes and vacancies when writes are allowed.

### Out of scope

- Accounts and login.
- LLM or embeddings in the application.
- Automatic lexicon updates from the web. The lexicon grows through a reviewed manual process (see section 6).
- Paraphrase or semantic matching, partial matches.
- PDF/DOCX import, fetching a vacancy by URL.
- Company verification and a red-flag check of vacancies.
- Interview questions and confidence ratings.
- Ukrainian texts.
- Agent access and proposals: see [docs/features/agent-proposals.md](features/agent-proposals.md).
- Deployment with real personal data.

### Documented limitations

These are known and will be stated in the README:

- **English only.** Ukrainian would need Unicode-aware tokenization and stem matching.
- **Paraphrase is a gap.** "Designed HTTP endpoints" does not match the term REST. The engine knows only aliases (`k8s` for Kubernetes) and `implements` relations from the lexicon, one hop deep.
- **Negation is a heuristic.** It looks for patterns shortly before a term. It misses a negation placed after the term ("Docker: no experience") and wording that is not in the lexicon. A negation in the vacancy ("no Kubernetes experience required") is not understood: the term is still detected as a requirement.
- **Hedging is not negation.** "Limited exposure to Docker" is Matched.
- **Only skills from the lexicon are checked.** A skill that is not in the lexicon is never detected. The UI says how many skills the lexicon knows. A line of a "Requirements" or "Nice to have" section with no known term is listed as Not checked. This works only when the vacancy has such sections: in a posting without headings the engine cannot tell which lines are requirements.
- **A match proves a mention, not a skill level.** "Familiar with Docker" and "Led the Docker migration of 40 services" are both Matched.
- **Fluff is skipped when a technology is named.** "Passionate about Docker" is not flagged, because it names a technology.

## 5. Acceptance criteria

### Story 1: analyze pasted text

- Given a resume and a vacancy of at least 30 characters each, when I press Analyze, then I see coverage, matched requirements, gaps and fluff.
- Given an empty field or a field shorter than 30 characters, then the API returns `400` with a JSON error naming the field, and the UI shows it next to that field.
- Given a field longer than 20,000 characters, then the API returns `400` with a clear message.
- While the analysis runs I see a loading state. If the API fails I see an error message and can retry.
- "Load sample" offers two presets from `GET /api/samples`: *Strong match* and *Tricky cases*. It fills both fields and stores nothing.
- After Analyze, the resume panel becomes a read-only preview with an "Edit text" button. The text is shown exactly as typed (line breaks preserved) and is never rendered as HTML.
- "Copy summary" copies a plain-text report to the clipboard: the coverage line, the gaps with their reasons, and the fluff sentences.

### Story 2: requirements

- Given a vacancy with the sections "Requirements" and "Nice to have", then terms under them get priority must and nice. A term that appears in both sections is must.
- Terms under "About us" are ignored.
- Given a vacancy that contains no lexicon term, then I see "No known requirements found" and no coverage numbers. If it has requirement sections, their lines are listed as Not checked.
- Every line under "Requirements" or "Nice to have" that contains no lexicon term is listed as Not checked, word for word, in the order of the vacancy. Lines under "About us" are never listed. Not checked lines do not count in either coverage number; their number is shown next to the caption ("· 3 lines not checked"), and omitted when it is 0.
- I can exclude a requirement. Excluded requirements are not shown and do not count in either coverage number. The result is recalculated.

### Story 3: matches and gaps

- Every matched requirement returns `quote` (an exact substring of the resume after normalization of whitespace, quotes and dashes) and `start`/`end` offsets into the original resume text. The UI uses them to highlight the quote. A quote that cannot be located is never returned.
- If a term appears several times in the resume, the first occurrence in the text is used.
- An alias counts as the same term: `k8s` in the resume matches Kubernetes in the vacancy, and `Postgres` matches PostgreSQL.
- A requirement with no quote is a Gap with reason `no_quote`.
- The lexicon relates terms in three ways: `aliases` (the same thing written differently), `implements` (a tool that provides a concept: Redux Toolkit implements state management) and `related`. Only aliases and `implements` can make a Matched; `related` only gives a hint in the UI. Relations are followed one hop, never in chains.
- A vacancy that asks for a concept is matched by a tool in the resume, and the UI says so ("via Redux Toolkit"). The reverse does not hold: a vacancy that asks for Redux is not matched by the words "state management".
- A negation pattern ending at most 3 tokens before the start of the term gives a Gap with reason `negated`. A pattern is a negator, up to 2 filler words, an experience word and a preposition ("no experience with Docker", "no commercial experience with Docker", "no hands-on experience with Docker"), or "never used" / "have not worked with". An aspiration pattern ("currently learning", "plan to learn", "studying") gives reason `aspirational`.
- These stay Matched: "Machine learning with Python", "Deployed Redis-backed services without downtime", "Not only did I ship the service with Terraform".
- Coverage reads `9 of 12 requirements matched · must-have 7 of 8`, with the caption "12 requirements detected in this posting (skills from the lexicon)". There is no single percentage.
- A switch above the resume preview shows one highlight layer at a time. *Evidence*: clicking a requirement highlights its quote in green. *Fluff*: flagged sentences are highlighted in yellow. The two never overlap.

### Story 4: fluff

- A sentence of the resume is flagged when it contains a fluff word and contains no digit, no technology term and no action verb.
- "Led a team of 5 engineers and cut release time by 30%, keeping clean code standards" is not flagged. "Responsible for team communication and clean code" is flagged.
- Flagged sentences are highlighted in the Fluff layer and listed with the reason they were flagged.
- A resume with no fluff shows "No fluff found".
- Fluff is checked in the resume only.

### Story 5: saving (personal mode)

- With `ALLOW_WRITES=true` I can create, edit and delete resumes and vacancies, and analyze a saved pair by id.
- With `ALLOW_WRITES=false` every `POST`, `PUT` and `DELETE` that changes stored data returns `403` with a JSON error, and the UI hides the save controls. `POST /api/analyze` stays allowed.
- An unknown id returns `404`. An empty title returns `400`. A title longer than 100 characters returns `400`.

## 6. Data

- **Resume**: title (required, max 100 characters), text (required, 30–20,000 characters).
- **Vacancy**: company (optional), title (required, max 100 characters), text (required, 30–20,000 characters).
- **Analysis** (requirements, matches, gaps, coverage, fluff) is derived from the two texts and is never stored.

The lexicon is engine configuration, kept in `data/lexicon.json` (not in `seed.json`). It holds:

- terms with their aliases and their relations to other terms, `implements` and `related` (about 120 terms to start),
- section headings and markers for must, nice and ignored sections,
- fluff words,
- action verbs,
- negation and aspiration patterns,
- a `lang` key (`en`).

The MVP lexicon is curated by hand (about 120 terms). Bulk seeding from open datasets (candidates: ESCO skills, Stack Overflow tag synonyms) is planned **after** the MVP, as a script whose output goes through the same review and tests. Their licenses and formats must be checked first, and `implements` relations stay manual.

Rule: a fluff word is never a term. A test checks that the two lists do not overlap.

The lexicon grows through a reviewed manual process, outside the application. Candidate terms, aliases and relations are collected with a search tool, and every alias and `implements` relation carries a source link. A person reviews the change as a diff before it is committed. Lexicon tests check that: the schema is valid, no alias belongs to two terms, relations point to existing terms, `implements` has no cycles, and fluff words and terms do not overlap. After every lexicon change, the results for the two sample presets must stay as expected.

Demo records live in `data/seed.json`: 2 resumes, 8 vacancies and 2 sample presets.

## 7. Open questions

- Priority of terms outside any recognised section. Assumed: nice, unless the sentence says "required" or "must".
- Who maintains the lexicon, and how large should it grow beyond the first 120 terms?
- Fluff is checked in the resume only. Revisit if the same check is wanted for pasted AI text.
- Which open datasets seed the lexicon after the MVP, under which license, and how are their noisy generic terms ("communication") filtered?
