# Glossary

The words of TailorProof and what exactly they mean in **this** app. AI agents read this file before every task (it is loaded through `opencode.jsonc`). Use these exact words in requirements, UI texts, code and tests. If you rename a term, change it here first.

## Terms

| Term | Meaning in this app | Not to be confused with | In code |
| --- | --- | --- | --- |
| Resume | One saved or pasted text of a resume. Always called "resume", never "CV". | Vacancy | `Resume` |
| Vacancy | One saved or pasted job posting. | Application (applying to a job; not in the MVP) | `Vacancy` |
| Term | One entry of the lexicon (a skill with its aliases). It can appear in a vacancy and in a resume. | Requirement | `Term` |
| Alias | Another spelling of the same term (`k8s` for Kubernetes). It means exactly the same thing. | Implements, Paraphrase | `aliases` |
| Implements | A one-way relation: a tool provides a concept (Redux Toolkit implements state management). A resume mention of the tool matches a vacancy that asks for the concept, never the other way round. One hop only. | Alias | `implements` |
| Related | A link between terms that only gives a hint in the UI. It never makes a Matched. | Implements | `related` |
| Paraphrase | The same idea in other words ("HTTP endpoints" for REST). The MVP does not match it, so it is a Gap. | Alias | — |
| Requirement | A term found in the vacancy. It has a priority. | Term, Gap | `Requirement` |
| Priority | `must` or `nice`, taken from the section or the wording around the term in the vacancy. | Confidence (out of scope) | `priority` |
| Quote | An exact substring of the resume that proves a requirement, with its `start` and `end` in the original text. | Sentence, line | `Quote` |
| Matched | A requirement that has a Quote, directly, through an alias or through `implements`. | Verified | `matched` |
| Verified | The check that a Quote really exists in the resume text. | Matched | `verifyQuote()` |
| Gap | A requirement with no Quote, with a reason: `no_quote`, `negated` or `aspirational`. | Requirement | `Gap`, `reason` |
| Not checked | A line of a vacancy's "Requirements" or "Nice to have" section in which the lexicon found no term. Shown as written and never counted in Coverage. | Gap (a term was found, but has no Quote) | `NotChecked`, `notChecked` |
| Coverage | Two numbers: requirements matched out of requirements detected, and must-have matched out of must-have detected. Never one percentage. | Confidence | `coverage` |
| Fluff | A resume sentence that contains a fluff word and has no digit, no technology term and no action verb. | Weak sentence (not used) | `Fluff` |
| Lexicon | The engine's data in `data/lexicon.json`: terms with their relations, markers, fluff words, action verbs, negation patterns. | Seed | `lexicon` |
| Seed | Demo resumes and vacancies in `data/seed.json`. | Lexicon | `seed` |
| Analysis | The derived result for one resume and one vacancy. Never stored. | Proposal | `Analysis` |
| Demo mode / Personal mode | Writes disabled (`ALLOW_WRITES=false`, the default) or enabled (`ALLOW_WRITES=true`). | — | `ALLOW_WRITES` |
| Proposal | An agent's suggested rewrite with its Quotes (planned feature). | Resume edit | `Proposal` |
| Accept | Setting a Proposal's status to `accepted`. It never changes the resume. | Apply | `status` |
| Apply | Changing the resume text. Always done by the user, never by Accept or by an agent. | Accept | — |
