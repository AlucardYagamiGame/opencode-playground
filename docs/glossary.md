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
| Paraphrase | The same idea in other words ("HTTP endpoints" for REST). The MVP does not match it, so it is a Gap. | Alias | — |
| Mention | One place in the resume where a term, one of its aliases, or a tool that implements it appears as whole tokens. A mention is clean when its sentence has no negation or aspiration pattern shortly before the term. | Quote | `Mention` |
| Sentence | A piece of the resume or vacancy text split by one fixed rule (list lines, `.!?` followed by whitespace, line breaks that are not soft wraps). Quotes and fluff both work on sentences. | Line | `Sentence` |
| Section | A part of a vacancy that starts at a heading and lasts until the next heading. Its kind (must, nice, ignore or neutral) is the kind of its heading. | Heading | `Section` |
| Heading | A line of a vacancy that matches a heading of the lexicon ("Requirements", "Nice to have: ..."). Headings belong to four groups: must, nice, ignore and neutral. | Section | `heading` |
| Marker word | A word in a sentence that sets a priority outside sections: must markers ("required", "must") and nice markers ("plus", "bonus"). | Heading | `markers` |
| Navigation line | A line of a pasted web page that is not part of the posting ("Show less", "Apply", "Save"). It is skipped everywhere. | Not checked | `navigationLines` |
| Requirement | A term found in a vacancy, in a must or nice section or in neutral text. It has a priority. Terms in ignored sections are not requirements. | Term, Gap | `Requirement` |
| Priority | `must` or `nice`, taken from the section or from the marker words of the sentence. | Confidence (out of scope) | `priority` |
| Quote | The whole sentence of the resume that contains the mention proving a requirement, with its `start` and `end` in the original text. `quote === resume.slice(start, end)`. Only a Matched requirement has one. | Mention, Line | `Quote` |
| Matched | A requirement with a Quote, found directly, through an alias or through `implements`. | Verified | `matched` |
| Verified | The check that a quote really exists in the resume text: `verifyQuote(resume, quote)` returns `{ start, end }` or `null`. | Matched | `verifyQuote()` |
| Gap | A requirement that is not Matched, with a reason: `no_quote`, `negated` or `aspirational`. | Not checked | `Gap`, `reason` |
| Context | The sentence that caused the reason of a negated or aspirational Gap, with its own `start` and `end`. It is shown to explain the Gap and is not proof. | Quote | `context` |
| Negation | A pattern shortly before a term that says the person does not have the skill ("no experience with Docker"). It gives the reason `negated`. | Hedging (not negation) | `negated` |
| Aspiration | A pattern that says the person is only learning the skill ("currently learning Docker"). It gives the reason `aspirational`. | Negation | `aspirational` |
| Not checked | A line of a vacancy's must or nice section in which the lexicon found no term. Shown as written and never counted in Coverage. | Gap (a term was found, but has no Quote) | `NotChecked`, `notChecked` |
| Exclude | The user's action that removes a requirement from the result and from Coverage. An excluded requirement can be restored. | Delete | `excluded` |
| Coverage | Two numbers: requirements matched out of requirements detected, and must-have matched out of must-have detected. Never one percentage. | Confidence | `coverage` |
| Fluff | A resume sentence that contains a fluff word and has no digit, no technology term and no action verb. | Weak sentence (not used) | `Fluff` |
| Fluff word | A word or phrase from the lexicon that signals an empty claim ("passionate", "team player"). Never a term. | Fluff | `fluffWords` |
| Action verb | A verb from the lexicon that shows a concrete action ("built", "led", "migrated"). A sentence with one is not Fluff. | — | `actionVerbs` |
| Lexicon | The engine's data in `data/lexicon.json`: terms with their relations, section headings, marker words, navigation lines, fluff words, action verbs, negation and aspiration patterns. | Seed | `lexicon` |
| Seed | Demo resumes and vacancies in `data/seed.json`. | Lexicon | `seed` |
| Sample | One of two presets (a resume and a vacancy from the seed) that "Load sample" puts into the Analyze fields. | Seed | `Sample` |
| Reference results | The expected analysis of the two samples, written in `docs/requirements.md`. They are the first tests (golden tests). | Seed | `golden` |
| Highlight layer | One of the two views of the resume preview: *Evidence* (quotes, green) or *Fluff* (flagged sentences, yellow). Only one is shown at a time. | — | `layer` |
| Analysis | The derived result for one resume and one vacancy. Never stored. | Proposal | `Analysis` |
| Demo mode / Personal mode | Writes disabled (`ALLOW_WRITES=false`, the default) or enabled (`ALLOW_WRITES=true`). | — | `ALLOW_WRITES` |
| Proposal | An agent's suggested rewrite with its Quotes (planned feature). | Resume edit | `Proposal` |
| Accept | Setting a Proposal's status to `accepted`. It never changes the resume. | Apply | `status` |
| Apply | Changing the resume text. Always done by the user, never by Accept or by an agent. | Accept | — |
