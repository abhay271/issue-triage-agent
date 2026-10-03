# CLAUDE.md

Shadow-mode GitHub issue triage agent. It predicts labels for issues, learns from maintainer corrections through retrieval memory, and earns autonomy once its precision is high enough. It is evaluated by backtesting on pandas-dev/pandas and astral-sh/ruff.

## Session start
1. Read `docs/implementation-plan.md`. Find the current phase and the first unchecked task. That is the next piece of work.
2. Read `docs/architecture.md` for design context before touching code.

| Doc | Read when |
|---|---|
| `docs/implementation-plan.md` | Every session: current phase, next task, required tests |
| `docs/architecture.md` | Before implementing or changing any component |
| `docs/decisions.md` | Before changing a design choice, to see why it was made. Log new decisions here. |

## Workflow rules
- Work on one logical task at a time. Finish it before starting the next.
- Major features are test-first:
  1. Write the required tests from the plan.
  2. Run them and confirm they fail.
  3. Implement until they pass.
- Never mark a task complete until all its required tests pass.
- After finishing a task or feature:
  - Check it off in `docs/implementation-plan.md`.
  - Note any change from the plan under that phase's **Deviations**.
  - Log design changes in `docs/decisions.md`.
- Never write to GitHub. All API access is read-only.
- Tests never hit the network. Use respx mocks and fixtures.

## Code standards
- No redundant, over-engineered or jargon-heavy code. Prefer the simplest thing that works.
- Clear, descriptive names.
- Comments explain intent or non-obvious reasons, never what the code plainly does. Keep them short.

## Docs standards
All docs are clean, concise and professional. Update them in the same change as the code they describe.

## Maintenance
This file is a living document. Once they exist, add sections for:
- build, test and run commands;
- project structure;
- conventions and gotchas.

Add only what is real. Do not guess ahead.
