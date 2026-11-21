# Project memory

## Product intent & audience

- Governance/decision-tracking project; user reviews structured decision registers before final verdicts.
- Decisions are organized into rounds (e.g., "round3") and tracks (e.g., "d10a"), suggesting an iterative multi-round governance process.

## Brand personality & voice

- Analytical, structured, deliberative tone; user expects methodical one-at-a-time review.
- Explanations should cover both financial and technical consequences before any verdict.

## Visual design & UX direction

- No UI inferred yet; current interaction is conversational/document-review based.

## Decisions, constraints & rationale

- Current workspace is minimal (`.gitignore`, `package-lock.json`, `README.md`); not a git repository and has no `docs/governance/` folder.
- User references commit `e4704b6d` — version history/snapshots matter to the user even though git isn't initialized here.
- File-naming convention inferred: `docs/governance/d{track}-round{n}-{descriptor}.md`.

## Open questions & assumptions

- Where does the decision register actually live? It may be in an external repo or prior Bolt session not mounted here.
- User expects 17 decision statements to review; source document not yet available in workspace.
- Awaiting user's choice: restore via Version History, paste contents, or switch workspace.