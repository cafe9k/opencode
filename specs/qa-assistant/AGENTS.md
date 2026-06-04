# QA Assistant AI Execution Constraints

These instructions apply to all work under `specs/qa-assistant/` and to future implementation work for the QA Assistant second-development project.

## Project Intent

- Treat QA Assistant as a second-development project on top of OpenCode, not as a hard fork of OpenCode.
- Prefer isolated documentation, isolated packages, plugins, agents, tools, and thin adapters over changes to official OpenCode core code.
- Keep the project aligned with the upstream `dev` branch and minimize future merge conflicts.

## Documentation Rules

- Keep QA-specific plans, architecture, prompts, schemas, examples, sync notes, and release checklists under `specs/qa-assistant/` unless there is a strong reason to place them elsewhere.
- Do not rewrite top-level OpenCode documentation to carry QA-specific product details.
- If top-level documentation needs to mention this project, add only a short pointer to the QA-specific document.
- Record upstream sync decisions in `specs/qa-assistant/operations/sync-upstream.md` when synchronization work happens.

## Architecture Rules

- Prefer a future isolated package such as `packages/qa-assistant` for implementation.
- Keep QA business logic out of OpenCode core packages.
- If OpenCode integration is necessary, use plugins, agent tools, public APIs, or a thin adapter.
- Do not add QA-specific logic directly to session runner, model execution, file tools, or shared server routes.
- Do not make OpenCode core depend on QA Assistant modules.
- Use a dedicated configuration namespace such as `qa_assistant`.
- Keep storage isolated, either in a dedicated SQLite database or clearly prefixed tables.

## Upstream Sync Rules

- Use `dev` or `origin/dev` as the base for local diffs; do not assume a local `main` ref exists.
- For upstream synchronization, prefer a temporary branch named like `qa/sync-YYYYMMDD`.
- Before broad changes, check whether the same result can be achieved by moving code into isolated QA files.
- Keep patches to official high-churn areas small and centralized.
- High-conflict areas include:
  - `packages/opencode/src/session/`
  - `packages/opencode/src/server/`
  - `packages/opencode/src/plugin/`
  - `packages/desktop/`
  - `packages/app/`
  - top-level `README*.md`

## Implementation Rules

- Follow the root `AGENTS.md` style guide.
- Use Bun APIs when possible.
- Avoid `any`.
- Avoid unnecessary helper extraction.
- Prefer functional array methods over loops when it stays readable.
- Do not introduce broad abstractions before the MVP proves the workflow.
- Keep CLI, agent pipeline, schema validation, export, and desktop UI boundaries explicit.
- Preserve intermediate generation artifacts so failures can be diagnosed by stage.

## Testing and Verification

- Do not run tests from the repository root.
- Run `bun typecheck` from the relevant package directory.
- For document-only changes, run `git diff --check`.
- For future implementation changes, include at least a CLI smoke path that parses a sample requirement, scans a sample repo, validates JSON, and exports `xlsx` or `csv`.

## Safety Constraints

- QA Agent behavior must be read/export only by default.
- Do not allow the QA Agent to modify the target repository.
- Do not fabricate file paths in `codeRefs`.
- Restrict repository reads to the user-selected repo root.
- Restrict requirement document reads to user-selected files.
- Keep generated exports in explicit user-selected or configured output locations.
