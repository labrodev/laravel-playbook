# Workflow

This document defines the standard workflow for implementing tasks in Labrodev Laravel projects.

The goal is consistent execution: predictable changes, minimal architectural drift, and reliable delivery quality.

If any step conflicts with architectural rules, naming conventions, or boundaries, propose the closest compliant alternative instead of breaking the rules.

---

## 0) Mandatory pre-checks

Before touching code:

1. Read `../index.md` (playbook root) and follow the documentation map and precedence rules.
2. If deeper context is required, consult `docs/index.md` and the relevant chapters under `docs/`.
3. Treat `docs/13-review-checklist.md` as acceptance criteria.

No code should be written before these steps are mentally completed.

---

## 1) Task interpretation

For every task, explicitly define:

- a short summary of the desired outcome (1–3 sentences)
- assumptions (only if unavoidable)
- impacted domains/modules (`App/Layer` and/or `Core/Domain/{Domain}`)
- data impact (database changes: yes/no)
- risk level (low / medium / high)

Avoid overplanning. Keep it practical and explicit.

---

## 2) Implementation plan (roadmap)

Prepare a step-by-step roadmap before writing code.

The roadmap must include:
- files and folders to be created or modified (exact paths)
- new classes to be added (exact names)
- whether a database migration is required
- whether factories or seeders must be updated
- whether automated tests are in scope (only when the task or this plan explicitly requires them)
- notable edge cases or constraints

The roadmap must respect project structure, naming rules, and boundaries.

---

## 3) Branching rules

Create a new branch before starting implementation.

Branch naming format:

YYYY-MM-DD_HH-MM-SS_short-description

Example:
- `2026-01-19_21-01-01_update_order`

Rules:
- use an action-oriented description
- use lowercase and underscores
- do not use spaces
- keep the description short but explicit

---

## 4) Database changes

If the task requires adding or modifying persisted data:

1. Create a migration.
2. Update the affected model(s):
    - add or update casts
    - add or update `$visible`
    - add or update relationships if needed
    - ensure the model declares `#[CollectedBy(...)]` for the domain Collection
3. Update relevant Data validation rules (store/update only).
4. Update or add tests **only when the task or implementation plan explicitly asks for tests**.

If applicable, also update:
- Factory (when the model is used in tests or common setup)
- Seeders (when the application relies on seeded defaults or reference data)

Rules:
- migrations are mandatory for schema changes
- models do not use `$fillable`
- no mass assignment
- attributes are assigned row by row in Actions or Orchestrators
- factories and seeders create data only and must not encode workflows

---

## 5) Write-side changes (store/update)

Write-side changes must follow the canonical pattern:

- Layer controller (`App/Layer/...`) handles I/O only
- Data class (`Core/Domain/{Domain}/Data`) defines validation and casting
- Action (`Core/Domain/{Domain}/Actions`) exposes `__invoke(...)` as its single public entry point
- Action assigns model attributes explicitly
- Orchestrator is used only for multi-step or cross-action workflows (Orchestrators use `execute(...)` as their workflow entry point)

Controller rules:
- inject Actions as method arguments
- do not resolve Actions manually
- do not introduce Request classes
- do not pass raw arrays into Core

---

## 6) Read-side changes

Read-side logic belongs exclusively to the delivery layer (`App/Layer`):

- IndexQueries for listing, filtering, pagination
- ViewModels for presentation shaping
- Exports for file or external format generation

Rules:
- read-side does not use Data classes
- read-side does not use validation
- read-side must not mutate domain state

---

## 7) Code style and comments

Rules:
- do not add custom comments by default
- add a comment only for genuinely non-obvious or complex logic
- prefer clarity through naming, structure, and small methods
- follow all naming conventions strictly (singular, entity-first Actions, single public `__invoke()` on Actions)

Code should explain itself through structure.

---

## 8) Testing

**Do not add, expand, or run automated tests unless the task or implementation plan explicitly asks for tests.** When tests are in scope, they must reflect the architecture:

- business behavior is exercised through Actions and Orchestrators (invoke Actions as callables in tests)
- Rules and Services are tested as unit tests when non-trivial
- Jobs are tested as wrappers (delegation and configuration only)
- HTTP / Layer tests verify wiring only
- use factories for setup, not for behavior

If behavior changes and tests are in scope:
- add new tests or update existing ones accordingly

---

## 9) Self-review checklist

Before finalizing the change:

1. Re-read `docs/13-review-checklist.md` and ensure all **Must** items pass.
2. Verify architecture boundaries are respected.
3. Verify naming conventions are respected.
4. Verify no Request classes were introduced.
5. Verify no `$fillable` or mass assignment exists.

---

## 10) Tooling checks

Run the following tools for the changed code:

- phpstan
- pint (for modified files)
- rector (for modified files)

Fix all issues before opening a PR.

---

## 11) Change overview

Before creating a PR, prepare a concise internal overview:

- what was changed
- why it was changed
- which domains/modules were affected
- whether database changes were introduced

This overview becomes the PR summary.

---

## 12) Pull request creation

Create a PR targeting the branch from which the task originated.

In the PR description:
- include the change overview
- mention migrations explicitly (if any)
- mention test coverage changes only when tests were part of the task

PRs must be understandable without reading the entire diff.

---

## Final rule

If a step is skipped, explain why explicitly.

Consistency and clarity are more important than speed.
