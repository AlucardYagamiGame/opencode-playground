# Feature: Agent proposals

> **Status:** planned — do not implement as part of the MVP

## Idea

An external AI agent (for example Claude, through an MCP adapter) reads the requirements and gaps of an analysis and suggests a rewrite of a weak resume sentence. The application contains no LLM and no LLM key: the agent runs outside, in the user's own tool.

The application stays in control. Every suggestion is a **Proposal** that carries the exact quotes it relies on. The API verifies each quote against the resume text. A Proposal with an invented quote is rejected.

## User stories

1. As a job seeker, I want each agent rewrite to appear with the exact quotes it relies on, so that I can check it against my own facts.
2. As a job seeker, I want to accept or reject a rewrite, so that my resume stays under my control.

## Acceptance criteria

- A quote that is not found in the resume: the API returns `422`, nothing is saved, and the offending quote is named in the error body.
- All quotes verified: the Proposal is saved as `pending` and shown next to the original sentence.
- Accept: the status becomes `accepted` and the user can copy the text. The resume text is **unchanged** (Accept does not mean Apply).
- Reject: the status becomes `rejected`.
- With `ALLOW_WRITES=false`, creating a Proposal and changing its status return `403`.
- The resume is edited after the Proposal was created: the Proposal stays, and the UI marks it `outdated`.
- Unknown id returns `404`.

## What it touches

- **Data:** new entity `Proposal` (`originalQuote`, `rewritten`, `quotes[]`, `status`: `pending | accepted | rejected`, link to a Resume).
- **API:** create, list and change status of Proposals. It reuses the quote verification function of the MVP (`verifyQuote`).
- **Agent adapter:** a small MCP server over the REST API. Tools: `list_requirements_and_gaps`, `verify_quote`, `submit_proposal`. One new dependency, `@modelcontextprotocol/sdk`, to be justified in `docs/architecture/decisions.md`.
- **UI:** an "Agent proposals" panel with the original sentence, the rewrite, the verified quotes, and Accept / Reject buttons.

## Constraints

- The application checks that quotes exist. It does **not** judge whether the rewrite is good or follows from the quotes.
- No LLM and no LLM key in the application or the repository.
- The agent connects only to a local API (no remote exposure, no tokens in this version).

## Decisions the MVP must keep open

- The analysis API returns JSON with stable shapes: the same structures serve the browser and the agent.
- Quote verification is a separate, tested function, so the Proposal endpoint can call it.
