# Product Requirements

> **Status:** reviewed draft (Step 4). Change it to "MVP requirements ready" after you read it once more.

This file describes **what** TailorProof must do. AI agents read it before planning or implementing anything (it is loaded automatically through `opencode.jsonc`). Technical decisions (storage, hosting, folder structure) live in `docs/architecture.md`. Domain terms are defined in `docs/glossary.md`: use them exactly.

---

## 1. Product overview

**TailorProof** checks a resume against a specific job posting without an LLM and without paid APIs. It shows which requirements of the vacancy the resume covers (each with an exact quote from the resume), which it does not (gaps), and which sentences make an empty claim (fluff).

Problem: when tailoring a resume to a specific posting, it is easy to miss requirements or to leave fluff (often added by AI tools) that cannot be backed up in an interview.

The analysis is deterministic: it is based on a word list (the lexicon), not on a language model. Every result can be explained and tested. The idea is inspired by the open-source project pitches-peaches (evidence with quotes, no fake-precision scores); TailorProof replaces the LLM with a lexicon engine and adds a web UI and a fluff linter.

The product is honest about what it cannot see: a requirement the lexicon knows is either proven by a quote or reported as a gap with a reason, and a requirement line the lexicon does not know is listed as Not checked (where the posting has sections; see the limitations).

## 2. Users

A job seeker who tailors one resume to many vacancies. The texts are in English. There is a single user and there are no accounts.

The application runs in two modes, controlled by `ALLOW_WRITES`:

- **Demo mode** (`ALLOW_WRITES=false`, the default): nothing is stored. The user pastes texts or loads a sample and gets an analysis. This is what a public deployment runs.
- **Personal mode** (`ALLOW_WRITES=true`): resumes and vacancies can be saved in the database and loaded into the Analyze tab later.

The UI learns the mode, and how many skills the lexicon knows, from `GET /api/config`.

## 3. User stories

1. As a job seeker, I want to paste a resume and a vacancy and see the analysis at once, so that I get an answer without saving anything.
2. As a job seeker, I want to see the requirements found in the vacancy, split into must-have and nice-to-have, see which lines were not checked, and exclude terms that are not real requirements, so that the result reflects what the posting really asks for.
3. As a job seeker, I want each matched requirement shown with an exact quote from my resume, so that I see what is proven.
4. As a job seeker, I want each requirement that is not matched shown as a gap with a reason, so that I see what is missing and why.
5. As a job seeker, I want sentences in my resume that make an empty claim marked, so that I can replace them with facts.
6. As a job seeker, I want to read the result next to my resume with the evidence highlighted, and copy a short report, so that I can edit the resume and share the result.
7. As a job seeker, I want to save resumes and vacancies and load them into the Analyze tab, so that I can reuse them (Personal mode only).

## 4. Scope

### In scope (MVP)

- Analysis of pasted resume and vacancy text (`POST /api/analyze`, nothing stored).
- Requirement extraction by scanning the vacancy for terms from the lexicon, with aliases, one-hop `implements` relations and must/nice priority.
- Matched requirements with an exact quote and its position in the resume; gaps with a reason.
- Coverage shown as two numbers.
- Not checked: lines of the vacancy's must and nice sections in which the lexicon found no term, listed as written, so a gap in the lexicon is never silent.
- Fluff marking in the resume. Highlights are shown one layer at a time (Evidence or Fluff) in a read-only preview of the analyzed resume.
- A "Copy summary" button that copies a plain-text report to the clipboard.
- Two sample presets (`GET /api/samples`) built from `data/seed.json`.
- `GET /api/config` with the mode and the size of the lexicon.
- Saving, editing and deleting resumes and vacancies in Personal mode.

### Out of scope

- Accounts and login.
- LLM or embeddings in the application.
- Automatic lexicon updates from the web. The lexicon grows through a reviewed manual process (see section 6).
- Paraphrase or semantic matching, partial matches.
- Hints about related skills (`related` links between terms).
- PDF/DOCX import, fetching a vacancy by URL.
- Company verification and a red-flag check of vacancies.
- Interview questions and confidence ratings.
- Ukrainian texts.
- Analysis of a saved pair by id (the UI loads saved texts into the Analyze tab instead).
- Agent access and proposals: see [docs/features/agent-proposals.md](features/agent-proposals.md).
- A public deployment that holds personal data.

### Documented limitations

These are known and will be stated in the README:

- **English only.** Ukrainian would need Unicode-aware tokenization and stem matching.
- **Paraphrase is a gap.** "Designed HTTP endpoints" does not match the term REST. The engine knows only aliases (`k8s` for Kubernetes) and `implements` relations from the lexicon, one hop deep.
- **Negation is a heuristic.** It looks for patterns shortly before a term, in the same sentence. It misses a negation placed after the term ("Docker: no experience") and wording that is not in the lexicon. A negation in the vacancy ("no Kubernetes experience required") is not understood: the term is still detected as a requirement.
- **Hedging is not negation.** "Limited exposure to Docker" is Matched.
- **Only skills from the lexicon are checked.** A skill that is not in the lexicon is never detected. The UI says how many skills the lexicon knows. A line of a must or nice section with no known term is listed as Not checked. A line that contains one known and one unknown skill ("Kafka and Go") is not listed, because the known skill is found.
- **Not checked needs sections.** In a posting without recognised headings the engine cannot tell which lines are requirements.
- **Alternatives are counted separately.** "Postgres or MySQL" gives two requirements, not one choice.
- **Years of experience are not checked.** "4+ years of React" is Matched when React is mentioned.
- **The vacancy title is not analyzed**, only its text.
- **A match proves a mention, not a skill level.** "Familiar with Docker" and "Led the Docker migration of 40 services" are both Matched.
- **Fluff is skipped when a technology is named.** "Passionate about Docker" is not flagged, because it names a technology.
- **Terms that are ordinary English words** (Go, React, REST, Express, ...) are matched case-sensitively. "react to incidents" is not a mention, but a lowercase "react" in a skills list is missed, and "Go" at the start of a sentence can still be wrong.

## 5. Acceptance criteria

The precise engine rules are in "Engine rules" below. The criteria of the stories refer to them.

### Story 1: analyze pasted text

- Given a resume and a vacancy of at least 30 characters each (after trimming), when I press Analyze, then I see coverage, requirements, gaps, Not checked lines and fluff.
- Given an empty, whitespace-only or shorter than 30 characters field, then the API returns `400` with a JSON error naming the field, and the UI shows it next to that field.
- Given a field longer than 20,000 characters (after trimming), then the API returns `400` naming the field, and the UI shows it next to that field.
- While the analysis runs I see a loading state. If the API fails I see an error message and can retry.
- "Load sample" offers two presets from `GET /api/samples`: *Strong match* and *Tricky cases*. It fills both fields, clears the previous result and stores nothing. If a field already has text, I am asked to confirm before it is replaced.
- The page shows how many skills the lexicon knows.

### Story 2: requirements

- Priority comes from the sections of the vacancy ("Engine rules", sections). A term that appears several times in the vacancy gets the highest priority found: must beats nice. Terms under "About us" are ignored.
- Given a vacancy that contains no lexicon term, then I see "No known requirements found" and no coverage numbers. If it has must or nice sections, their lines are listed as Not checked.
- Every line of a must or nice section that contains no lexicon term is listed as Not checked, word for word, in the order of the vacancy. Navigation text such as "Show less" or "Apply" is never listed. Not checked lines do not count in either coverage number. Their number is shown next to the caption ("· 3 lines not checked") and omitted when it is 0.
- Outside recognised sections, priority comes from the wording of the sentence. "Kubernetes is required" is must, "AWS knowledge is a plus" is nice, a term with no marker is nice.
- I can exclude a requirement. It moves to a collapsed "Excluded" list with a Restore button, and does not count in either coverage number. The API recalculates the result. Exclusions are kept while the vacancy text is unchanged and are cleared when the vacancy text changes or a sample is loaded. If no requirement is left I see "No requirements left" and no coverage numbers.

### Story 3: matches and quotes

- Every Matched requirement has a `quote` and its `start`/`end` offsets into the original resume text, and `quote === resume.slice(start, end)`. A quote is the whole sentence or list line that contains the mention. A quote that cannot be located is never returned. Every Quote the engine returns passes `verifyQuote`.
- If a term is mentioned several times, the first clean mention is used ("Engine rules", choosing the mention). A direct mention (the term or an alias) is preferred over one through `implements`.
- An alias counts as the same term: `k8s` in the resume matches Kubernetes in the vacancy, and `Postgres` matches PostgreSQL.
- A vacancy that asks for a concept is matched by a tool in the resume, and the UI says so ("via Redux Toolkit"). The reverse does not hold: a vacancy that asks for Redux Toolkit is not matched by the words "state management". Relations are followed one hop, never in chains.
- Matching is by whole tokens: `Java` is not found in `JavaScript`, `React` is not counted in `React Native`, and `Redis-backed` matches Redis.
- These stay Matched: "Machine learning with Python", "Deployed Redis-backed services without downtime", "Not only did I ship the service with Terraform".

### Story 4: gaps and reasons

- A requirement that is not Matched is a Gap with one of the reasons `no_quote`, `negated` or `aspirational`.
- `no_quote`: the resume has no mention of the term. `negated`: the only mentions are negated ("no experience with Docker", "no commercial experience with Docker", "no hands-on experience with Docker", "never used Docker", "have not worked with Docker"). `aspirational`: the only mentions are aspirational ("currently learning Docker", "plan to learn Docker", "studying Docker").
- "No experience with Docker or Kubernetes" makes both terms `negated`.
- A negated or aspirational Gap shows its Context: the sentence that caused the reason, with its own `start` and `end`. A Context is not proof and is not a Quote.
- If the resume also has a clean mention of the term elsewhere (for example in the SKILLS line), the requirement is Matched.

### Story 5: fluff

- A sentence of the resume is flagged when it contains a fluff word and contains no digit, no technology term and no action verb.
- "Led a team of 5 engineers and cut release time by 30%, keeping clean code standards" is not flagged. "Responsible for team communication and clean code" is flagged.
- Each flagged sentence is listed with its reason in the form `fluff words: "passionate", "team player"; no number, technology term or action verb` (all fluff words found in the sentence).
- A resume with no fluff shows "No fluff found". Fluff is checked in the resume only.

### Story 6: reading the result

- After Analyze, the resume panel becomes a read-only preview with an "Edit text" button. The text is shown exactly as typed (line breaks preserved) and is never rendered as HTML. Editing the text clears the previous result.
- A switch above the preview shows one highlight layer at a time. *Evidence*: clicking a Matched requirement highlights its quote in green, and clicking a Gap that has a Context highlights the Context in red. *Fluff*: flagged sentences are highlighted in yellow.
- "Copy summary" copies a plain-text report in this form, and the button shows "Copied" afterwards:

  ```text
  TailorProof: Backend Engineer (Python), Initech
  3 of 7 requirements matched · must-have 2 of 5
  Gaps:
  - Docker (must): aspirational — "I am currently learning Docker."
  - Kubernetes (must): negated — "I have no commercial experience with Kubernetes."
  - REST (must): no_quote
  - PostgreSQL (nice): no_quote
  Fluff:
  - "Passionate team player with strong communication skills and a clean code mindset." (fluff words: "passionate", "team player"; no number, technology term or action verb)
  - "Dynamic, results-driven professional who thrives in fast-paced environments." (fluff words: "dynamic", "results-driven"; no number, technology term or action verb)
  - "Responsible for team communication and clean code." (fluff words: "responsible for"; no number, technology term or action verb)
  ```

  This is the report for the *Tricky cases* preset. The fluff words in it only illustrate the format: the real words come from the lexicon. Sections with no entries are left out; a *Not checked:* section (one line per Not checked line) and an *Excluded:* section (one line per excluded term) appear when they have entries. The first line uses the company and title when they are known and "Pasted texts" otherwise.

### Story 7: saving (Personal mode)

- With `ALLOW_WRITES=true` the Resumes and Vacancies tabs appear. Each lists its saved items, newest first, and offers create, edit, delete (after confirmation) and "Use in Analyze", which loads the saved text into the Analyze tab. In the Analyze tab, "Save resume" asks for a title and "Save vacancy" asks for a title and an optional company; both save the current text.
- With `ALLOW_WRITES=false` every request to `/api/resumes` and `/api/vacancies` returns `403` with a JSON error, and the UI hides the tabs and the save buttons. `POST /api/analyze`, `GET /api/samples` and `GET /api/config` stay allowed.
- An update replaces the whole record. An unknown id returns `404`. A title that is empty or whitespace-only returns `400`. A title or a company longer than 100 characters returns `400`.

### Engine rules

These rules make the criteria testable. They are the contract of the lexicon engine.

**Characters and sentences.** Matching ignores case and treats curly quotes, non-breaking spaces and en/em dashes as their plain equivalents. This replaces characters one for one and never changes the length of the text, so offsets always point into the original. One function splits a text into sentences and is used for both quotes and fluff:

- every line that starts with a list marker (`-`, `*`, `•`, `–`, `1.`, `1)`) starts a new sentence, and the marker is not part of it;
- a sentence also ends after `.`, `!` or `?` that is followed by whitespace, so `Node.js` and `3.5` never end a sentence;
- a line break ends a sentence, unless the next line starts with a lowercase letter (a soft wrap such as "reducing⏎deploy time");
- a blank line always ends a sentence;
- a sentence span has no leading marker or whitespace and no trailing whitespace.

**Matching a term.** A mention is a term or one of its aliases found as whole tokens: it must not be directly preceded or followed by a letter or a digit. Characters that are part of the term itself (`.`, `+`, `#`, `-`) count as part of it, so `Node.js`, `C++` and `C#` work. At each position the longest match wins, so `React Native` is one mention of React Native and not of React. A term that is also an ordinary English word (Go, REST, React, Express, Spring, Swift, Rust) is marked `caseSensitive` in the lexicon and must match with the exact case, so "react to incidents" and "the rest" are not mentions. Role names such as Full-Stack, Frontend, Backend or the "DevOps" in "DevOps Engineer" name a job, not a skill: they are not terms, so the title line of a vacancy adds no requirement.

**Choosing the mention.** For one requirement, collect the mentions in the resume: direct ones (the term or an alias) and ones through `implements` (a tool that implements the term). Classify each by its sentence as clean, negated or aspirational. Then:

1. a clean direct mention exists: Matched, with the first one as the quote;
2. otherwise a clean mention through `implements` exists: Matched, with the first one, and the UI says "via <tool>";
3. otherwise any negated or aspirational mention exists: Gap with the reason of the first one, direct mentions before mentions through `implements`;
4. otherwise: Gap with reason `no_quote`.

**Verifying a quote.** `verifyQuote(resume, quote)` returns `{ start, end }` of the first occurrence of `quote` in `resume`, or `null`. It compares with the same one-for-one normalization as matching and treats every whitespace character as a space. The Quotes of the engine always pass it (a test checks this), and the planned Agent proposals use the same function for quotes written by an agent.

**Sections and priority.** A heading is a line that, without a list marker, equals a heading from the lexicon (ignoring case and a trailing colon), or starts with such a heading followed by a colon ("Nice to have: Docker"). Headings belong to four groups: must ("Requirements", "Required", "Must have", "Qualifications"), nice ("Nice to have", "Bonus", "Preferred"), ignore ("About us", "About the company", "Benefits", "What we offer") and neutral ("About the role", "About the job", "What you will do", "Responsibilities"). Rules:

- A heading starts a section that lasts until the next heading or the end of the text. Text before the first heading is neutral.
- Content on the heading line after a colon belongs to the section ("Nice to have: Docker" puts Docker into nice).
- Every non-empty line of a must or nice section is one requirement line. The terms in it get the priority of the section. A line with no term is Not checked.
- Lines of the lexicon's navigation list ("Show more", "Show less", "Apply", "Save", "Sign in to see who you know") are skipped everywhere.
- Lines of an ignore section are skipped.
- In neutral text, the text is split into sentences and each term gets must if its sentence contains a must marker word (required, must, essential, mandatory), otherwise nice (a nice marker such as "plus" or "bonus" confirms it).
- A term found in several places (several sections, or a section and neutral text) gets the highest priority: must beats nice.

**Negation and aspiration.** The lexicon holds patterns. A pattern must end in the same sentence at most 3 tokens before the start of the term (tokens are words, so `Docker or Kubernetes` counts as 2 tokens after "with"). A negation pattern is a negator, up to 2 filler words, an experience word and a preposition ("no experience with", "no commercial experience with"), or "never used", or "have not worked with". Contractions are written with or without the apostrophe ("haven't", "haven’t"). "Without" counts only together with an experience word, so "without downtime" does not negate anything. An aspiration pattern is "currently learning", "plan to learn", "studying" and similar.

**Fluff.** A sentence is flagged when it contains a fluff word (a single word or a phrase, whole tokens, ignoring case) and contains no digit, no lexicon term (of any kind, not only the requirements of this vacancy) and no action verb. Fluff words are never terms.

**Coverage.** Requirements are the detected terms that are not excluded. The line reads `N of M requirements matched · must-have X of Y`, with the caption "M requirements detected in this posting (skills from the lexicon)". If there are no must-have requirements the must-have part is omitted. There is no single percentage.

### Reference results of the sample presets

The lexicon and the engine must give these results. They are the first tests (golden tests). A lexicon change that alters them is reviewed together with this table.

*Strong match* (resume "Full-Stack Engineer (strong match)", vacancy Acme Cloud, no Not checked lines, no fluff):

| Requirement | Priority | Result |
| --- | --- | --- |
| React, TypeScript, Node.js, REST, PostgreSQL, Docker | must | Matched |
| Kubernetes | must | Matched through the alias `k8s` |
| Redis | must | Gap `no_quote` |
| AWS | nice | Matched |
| State management | nice | Matched via Redux Toolkit |
| GraphQL, Terraform | nice | Gap `no_quote` |

Coverage: `9 of 12 requirements matched · must-have 7 of 8`.

*Tricky cases* (resume "Backend Developer (tricky cases: fluff, negation, paraphrase)", vacancy Initech, no Not checked lines):

| Requirement | Priority | Result |
| --- | --- | --- |
| Python, Redis | must | Matched |
| Terraform | nice | Matched |
| Docker | must | Gap `aspirational` ("I am currently learning Docker.") |
| Kubernetes | must | Gap `negated` ("I have no commercial experience with Kubernetes.") |
| REST | must | Gap `no_quote` (the resume only says "HTTP endpoints") |
| PostgreSQL | nice | Gap `no_quote` |

Coverage: `3 of 7 requirements matched · must-have 2 of 5`. Fluff: 3 sentences ("Passionate team player …", "Dynamic, results-driven professional …", "Responsible for team communication and clean code.").

Other seed vacancies the tests also cover: Globex shows "No known requirements found" and the three lines of its Requirements section as Not checked; Wayne has no headings, so Kubernetes is must and Terraform, GitHub Actions, Docker, Python and AWS are nice; Stark Digital's "Show more", "Show less", "Apply" and "Save" lines are skipped and "Nice to have: Docker" is read as a nice requirement.

## 6. Data

- **Resume**: `id`, title (required, max 100 characters), text (required, 30–20,000 characters after trimming), `createdAt`, `updatedAt`.
- **Vacancy**: `id`, company (optional, max 100 characters), title (required, max 100 characters), text (required, 30–20,000 characters after trimming), `createdAt`, `updatedAt`.
- **Analysis** (requirements, matches, gaps, Not checked lines, coverage, fluff) is derived from the two texts and is never stored.
- Texts are trimmed before validation and stored trimmed, so the length rules and the database constraints always agree.

The lexicon is engine configuration, kept in `data/lexicon.json` (not in `seed.json`). It holds:

- terms with their aliases, their `implements` relations to other terms and a `caseSensitive` flag (60 to 120 terms; it must cover every term used by the seed vacancies),
- section headings (must, nice, ignore, neutral) and must/nice marker words for sentences,
- navigation lines to skip,
- fluff words,
- action verbs,
- negation and aspiration patterns,
- a `lang` key (`en`).

Rule: a fluff word is never a term. A test checks that the two lists do not overlap.

The MVP lexicon is curated by hand. Bulk seeding from open datasets (candidates: ESCO skills, Stack Overflow tag synonyms) is planned **after** the MVP, as a script whose output goes through the same review and tests. Their licenses and formats must be checked first.

The lexicon grows through a reviewed manual process, outside the application. Candidate terms and relations are collected with a search tool, and every `implements` relation carries a source link. A person reviews the change as a diff before it is committed. Lexicon tests check that: the schema is valid, no alias belongs to two terms, relations point to existing terms, `implements` has no cycles, and fluff words and terms do not overlap. After every lexicon change, the results for the two sample presets must stay as in "Reference results".

Demo records live in `data/seed.json`: 2 resumes, 8 vacancies and 2 sample presets.

## 7. Open questions

Backlog, not blockers of the MVP:

- Which open datasets seed the lexicon after the MVP, under which license, and how are their noisy generic terms ("communication") filtered?
- Who maintains the lexicon, and how large should it grow?
- Fluff is checked in the resume only. Revisit if the same check is wanted for pasted AI text.
