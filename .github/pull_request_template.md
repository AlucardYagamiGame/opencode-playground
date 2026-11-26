<!--
PR title: Conventional Commits, e.g. "docs: add project brief and glossary"
Types: feat, fix, docs, test, refactor, chore, ci
-->

## Summary

<!-- What changes and why, in 1-3 sentences. -->

## Guide step / requirement

<!-- Which step of the course guide or which part of docs/requirements.md this belongs to. -->

## Changes

<!-- Short list of what was added, changed or removed. -->

-

## Verification

<!-- Only commands that were really run, with their real results. For docs-only changes write "docs only, nothing to run". -->

- [ ] `npm run lint`
- [ ] `npm run typecheck`
- [ ] `npm run test`
- [ ] `npm run build`
- [ ] `npm run test:e2e` (if a user flow changed)

## Checklist

- [ ] Terms match `docs/glossary.md`
- [ ] `docs/architecture/` updated (if a decision, endpoint, data model or dependency changed)
- [ ] Tests added for new behaviour (success, error, edge cases)
- [ ] No secrets committed; `.env.example` updated if variables changed

## Notes

<!-- Known limitations, skipped checks, follow-ups. Be honest: say what is incomplete or failing. -->
