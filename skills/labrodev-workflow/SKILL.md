---
name: labrodev-workflow
description: "Use when starting or finishing a task in a Labrodev Laravel project, planning an implementation, creating a branch, preparing or reviewing a pull request, or running quality gates (Pint, PHPStan/Larastan, Rector). Owns the end-to-end task workflow, the pre-PR review checklist, tooling gates, template precedence, and the recipes index."
license: MIT
metadata:
  author: labrodev
---

# Labrodev Workflow

Part of the Labrodev playbook skill set — assumes labrodev-core and labrodev-naming are installed. If absent, minimum global rules: `declare(strict_types=1)`, final classes, App/Layer depends on Core (never the reverse), named-argument invocation.

This skill defines HOW to execute a task end-to-end. WHAT each class looks like is owned by the component skills (see the routing table below).

## Rules

Musts:

- Follow the standard workflow steps in order. If a step is skipped, explain why explicitly.
- Create a branch before writing any code, using the timestamped naming format below.
- Write an implementation plan (exact file paths, exact class names, migration yes/no, factories/seeders yes/no, tests in scope yes/no) before writing code.
- Run Pint, PHPStan (Larastan), and Rector on the modified code after the work is done and before commit/PR. All three must pass.
- Treat the canonical templates embedded in the atomic labrodev-* skills as normative structural specifications — follow them exactly (modifiers, constructor shape, method names, visibility, placement).
- Verify every "Must" item of the pre-PR review checklist before opening a PR. Any failing Must means the change is not ready.
- Include a change overview in the PR description and mention migrations explicitly when present.
- If a workflow step conflicts with an architectural or naming rule, propose the closest compliant alternative — never break the rule to complete the step.

Must-nots:

- Do not add, expand, or run automated tests unless the task or the implementation plan explicitly asks for tests.
- Do not fix code style manually where Pint can fix it.
- Do not suppress PHPStan errors without strong, stated justification.
- Do not introduce new interfaces, contracts, abstraction layers, or `bind`/`singleton` registrations unless the task or plan explicitly names them.
- Do not add comments by default — comment only genuinely non-obvious logic; clarity comes from naming and structure.
- Do not invent a class structure when no governing template exists — stop and request explicit guidance.

## Standard workflow

Example task used throughout: "Add cancellation reason to bookings" in the Booking domain.

**1. Pre-checks.** Identify which components the task touches and load the matching labrodev-* skills (routing table below). Treat the pre-PR review checklist in this skill as the acceptance criteria. No code before this is done.

**2. Task interpretation.** State explicitly: a 1–3 sentence summary of the desired outcome, unavoidable assumptions only, impacted modules (`App/Layer` and/or `Core/Domain/{Domain}`), data impact (database changes: yes/no), and risk level (low/medium/high). Avoid overplanning.

**3. Implementation plan.** List files and folders to create or modify (exact paths), new classes (exact names), whether a migration is required, whether factories or seeders must be updated, whether tests are in scope (only when explicitly required), and notable edge cases. The plan must respect structure, naming, and boundaries.

**4. Branch.** Format: `YYYY-MM-DD_HH-MM-SS_short-description` — lowercase, underscores, no spaces, action-oriented, short but explicit.

```text
2026-07-09_14-32-05_add_booking_cancellation_reason
```

**5. Database changes (if any).** Migrations are mandatory for schema changes. Then update the affected model (casts, `$visible`, relations, `#[CollectedBy(...)]`) → see the labrodev-model skill. Update Data validation rules (store/update only) → see the labrodev-data skill. Update factories/seeders when the model is used in tests or seeded defaults; factories and seeders create data only and must not encode workflows. No `$fillable`, no mass assignment; attributes are assigned row by row in Actions/Orchestrators.

**6. Write-side changes (store/update).** Canonical chain: Layer controller (I/O only) → Data class (validation + casting) → Action (`__invoke(...)` as single public entry point, explicit attribute assignment); Orchestrators only for multi-step or cross-action workflows. Controllers inject Actions as method arguments, never resolve them manually, never introduce Request classes, never pass raw arrays into Core. Anatomy details: → see the labrodev-controller, labrodev-data, and labrodev-action skills.

**7. Read-side changes.** Read logic belongs exclusively to `App/Layer`: IndexQueries for listing/filtering/pagination (→ see the labrodev-query skill), ViewModels for presentation shaping (→ see the labrodev-viewmodel-resource skill), Exports for file generation. Read-side uses no Data classes, no validation, and must not mutate domain state.

**8. Code style and comments.** No custom comments by default; small methods; strict adherence to naming → see the labrodev-naming skill.

**9. Tests (only when in scope).** Business behavior through Actions/Orchestrators invoked as callables; Rules and Services as unit tests when non-trivial; Jobs as wrappers; HTTP tests verify wiring only; factories for setup, not behavior. Full strategy: → see the labrodev-testing skill.

**10. Self-review.** Walk the pre-PR review checklist below; verify boundaries, naming, no Request classes, no `$fillable`/mass assignment.

**11. Tooling gates.** Run and pass all three tools (next section).

**12. Change overview and PR.** Prepare a concise overview — it becomes the PR summary:

```markdown
## What changed
Added `cancellation_reason` to Booking: migration, model cast/$visible,
BookingUpdateData rule, BookingUpdate action assignment, show page ViewModel.

## Why
Operators need to record why a booking was cancelled.

## Affected modules
Core/Domain/Booking, App/Dashboard/Booking

## Database changes
Yes — one migration adding `cancellation_reason` (nullable string) to `booking`.
```

Target the branch the task originated from. Mention migrations explicitly; mention test coverage changes only when tests were part of the task. The PR must be understandable without reading the entire diff.

## Tooling gates

The tools are part of the architecture, not optional. Run them for the changed code after work, before commit:

```bash
vendor/bin/pint <modified files>      # code style — Pint config is canonical
vendor/bin/phpstan analyse            # static analysis (Larastan) — must pass
vendor/bin/rector process <paths>     # automated refactoring on modified files
```

- Pint: run on all modified files; never hand-fix what Pint fixes; its configuration is canonical.
- PHPStan: must pass at the configured level for all modified code; type safety beats convenience; suppression requires strong justification.
- Rector: run on modified files when applicable; do not ignore its suggestions without reason.

If a change requires fighting the tools, reconsider the design.

## Template precedence

The canonical templates embedded in each atomic labrodev-* skill are **normative, not illustrative**. They define file placement, class name shape, required modifiers (`final`, `readonly`), constructor signature, method names and visibility, and allowed dependencies. They do not define business logic.

When generating or modifying a class:

1. Identify the class type (Action, Data, Model, Query, Controller, ...).
2. Load the governing skill from the routing table below and use its embedded template as the structural specification.
3. Apply domain-specific logic only inside the structural boundaries the template defines.
4. Preserve all required modifiers, signatures, visibility, and method names.

Never: invent a new class structure, merge two templates into one file, add public methods a template does not define, remove or rename required methods, alter constructor shape, change visibility, omit `final`/`readonly`, or place a class outside its implied directory. If no suitable template exists, stop and request guidance. When a template and prose appear to conflict, the template wins; if the architecture genuinely changed, update the template first, prose second, code last. Silent deviation is forbidden.

Component → governing skill:

| Class / concern | Skill |
|---|---|
| Folder layout, dependency direction, headers, immutability, anti-patterns, legacy zones | labrodev-core |
| All naming, named-argument invocation, typed-parameter mirror naming | labrodev-naming |
| Controllers, routes, JsonControllers | labrodev-controller |
| ViewModels, Resources | labrodev-viewmodel-resource |
| Core Queries, Layer IndexQueries | labrodev-query |
| Data classes, validation, casters | labrodev-data |
| Models, Collections, Observers, migrations | labrodev-model |
| Actions, Services, Rules, Orchestrators/Pipelines/Payloads, Jobs delegation | labrodev-action |
| Policies, #[UsePolicy], #[Authorize] | labrodev-authorization |
| Enums | labrodev-enum |
| Pest tests, arch tests | labrodev-testing |
| React/Inertia pages, props, i18n | labrodev-inertia-react |

## Pre-PR review checklist

Fail the review if any Must fails. Each line is the check; component detail lives in the pointed skill.

Architecture and dependencies (Must):
- `App/Layer/*` depends on `Core/*` only; `Core/*` never imports `App/Layer/*`; `Core/Shared` depends on no specific Domain; `Core/Support` is domain-language-free → see the labrodev-core skill
- Cross-cutting workflows live in `Core/Feature/{FeatureName}/`, not duplicated across domains
- No new bindings/interfaces/abstractions unless the task or plan names them

Delivery layer (Must):
- Controllers thin, invokable, single-responsibility; class-level `#[Authorize(...)]`; no business rules → see the labrodev-controller and labrodev-authorization skills
- `{model:uuid}` route binding; write controllers use `Inertia::flash('toast', ...)` + `to_route(...)` → see the labrodev-controller skill
- IndexQueries read-only; ViewModels presentation-only; Exports never mutate → see the labrodev-query and labrodev-viewmodel-resource skills

Controller → Action (Must):
- Actions injected as method arguments (no `app()`/`resolve()`/`new`), invoked as callables with named arguments; no raw arrays into Core; no Request classes anywhere → see the labrodev-naming and labrodev-action skills
- Create Actions assign the UUID → see the labrodev-action skill

Naming (Must):
- Singular everywhere; entity-first Actions with single public `__invoke()`; verb-first `*Job`; no `Enum` suffix; snake_case model/Data attributes; typed DTO params mirror the class name; no widening required types to nullable → see the labrodev-naming skill

Data and validation (Must):
- Validation only for store/update, only in Data classes; controllers never validate; validation targets raw input keys, casting after; `prepareForPipeline()` where the skill requires it → see the labrodev-data skill

Models and database (Must):
- No `$fillable`, no mass assignment; explicit row-by-row assignment in Actions/Orchestrators; `casts()` method (never `$casts`); explicit `$visible`; relations with PHPStan generics docblocks; `{Model}Collection` + `#[CollectedBy]`; `#[UsePolicy]`/`#[ObservedBy]`/`#[UseFactory]` attributes → see the labrodev-model skill

Enums (Must):
- Full enum contract (label, options, validation rule, cast, frontend value+label) → see the labrodev-enum skill

Jobs, Observers, Events (Must):
- Jobs contain no business logic and delegate to Actions/Orchestrators; Observers never coordinate workflows or implicitly dispatch business jobs; Events are descriptive facts → see the labrodev-action and labrodev-model skills

Datetime and domain Queries (Must):
- No `CarbonImmutable`; use `Illuminate\Support\Carbon` with explicit `copy()` when mutation safety matters
- Core Query methods return `Illuminate\Database\Eloquent\Builder` by default → see the labrodev-query skill

UI copy (Must):
- User-visible strings via `t('Readable English')` backed by `lang/*.json`; ViewModels do not pass translations → see the labrodev-inertia-react skill

Testing (Should):
- Behavior via Actions/Orchestrators, not controllers; HTTP tests verify wiring only → see the labrodev-testing skill

Outcome guidance: any Must fails → request changes. Multiple Shoulds fail → request changes unless justified. A new pattern is introduced → it must be documented and its governing template updated first.

## Recipes index

Multi-step playbooks for common large tasks, shipped with this skill in `recipes/`:

- `scaffold-new-laravel-project.md` — greenfield project: PHP 8.5 + Laravel 13 only, Sail (PostgreSQL + Redis), Inertia + React starter, Fortify, selected Spatie packages, Horizon, Pint + Larastan + ESLint + Prettier. Use when creating a new application from an empty directory.
- `generate-dashboard-crud-inertia.md` — generate a complete dashboard CRUD slice (read/write controllers, IndexQuery, index/show ViewModels, Export) for an existing model. Use when adding standard CRUD screens.
- `extend-domain-classes.md` — fill out the minimal domain class set (create/update/remove Actions, Data classes with schema-derived validation, Enums, casts) across all models of existing domains. Use after models/migrations exist but the domain layer is incomplete.
- `implement-page-from-prototype.md` — end-to-end checklist for taking a designed page from prototype to production following the architecture. Use when implementing or enhancing a page against a design.
- `adopt-prototype-from-google-ai-studio.md` — map React prototype pages from a Google AI Studio export onto real routes, controllers, and views. Use when a stakeholder hands over an AI Studio prototype.

## Edge cases

- **Task conflicts with a rule:** never break the rule; propose the closest compliant alternative and state the trade-off.
- **A workflow step does not apply** (e.g. no database change): skip it, but say so explicitly in the plan/PR.
- **Hotfix pressure:** the tooling gates and the Must checks still apply; only the plan may be shortened.
- **No governing template for a needed class type:** stop; do not generate; request guidance.
- **Legacy vendor/starter code** encountered mid-task: → see the labrodev-core skill (two-zone policy).
- **Tests exist and behavior changed, but tests were not in scope:** flag it in the PR overview instead of silently editing or ignoring failures; expanding scope requires the task owner's call.

## Review checklist

- Was a branch created with the `YYYY-MM-DD_HH-MM-SS_short-description` format before any code was written?
- Does the implementation plan name exact file paths, exact class names, and state migration/factory/test scope?
- Did Pint, PHPStan, and Rector all run on the modified code and pass before commit?
- Were tests added or run only because the task or plan explicitly required them?
- Does every new class follow the canonical template of its governing labrodev-* skill with no structural deviation?
- Were all Must items of the pre-PR review checklist verified before opening the PR?
- Does the PR description contain the change overview and explicitly mention migrations (if any)?
- Is every skipped workflow step explicitly justified in the plan or PR?
- If a new pattern was introduced, was the governing template updated before the code?
