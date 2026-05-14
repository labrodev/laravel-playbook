# Playbook

Non-runtime materials used to design, reason about, and generate code for the project.

- `docs/` — architecture documentation and conventions
- `stubs/` — code templates consumed by AI tools (not Artisan)
- `recipes/` — step-by-step guides for recurring implementation patterns

Nothing in this folder is used directly at runtime.

---

## Quick norms

- Do not use `CarbonImmutable` — use `Illuminate\Support\Carbon` with explicit `copy()` when mutation safety matters. (Normative detail: `docs/07-anti-patterns.md` §25.)

- Domain `Queries` must return `Builder` only — do not return `int`, `Collection`, arrays, or other resolved results from domain query methods that act like composable query entry points. (Narrow exception for explicitly named terminal methods: `docs/07-anti-patterns.md` §26.)

- No interface/container `bind`/`singleton` wiring in application service providers **unless** the task or plan specifies it (`docs/07-anti-patterns.md` §27).

- React UI copy: **`t('Readable English')`** wired from Laravel **`lang/*.json`** through frontend i18n — not dotted slug keys for new copy (`docs/11-inertia-react.md` §10, `docs/07-anti-patterns.md` §19).

- Do not nullable-widen required domain parameters (`Product $product`, not `?Product $product`) to hide missing loads (`docs/07-anti-patterns.md` §28).

---

## Documentation map

- `docs/00-philosophy.md` — core principles, values, and non-negotiables
- `docs/01-project-structure.md` — folder layout, layers, and responsibility boundaries
- `docs/02-naming.md` — naming rules for domains, classes, methods, variables
- `docs/03-boundaries.md` — dependency direction and forbidden couplings
- `docs/04-data-and-validation.md` — input mapping, validation, write/read separation
- `docs/05-database-and-models.md` — models, Eloquent attributes, collections, observers
- `docs/06-testing.md` — testing strategy, structure, and allowed patterns
- `docs/07-anti-patterns.md` — explicitly forbidden approaches and shortcuts (Carbon §25, Queries §26, i18n pipeline §19, container §27, nullability §28)
- `docs/08-immutability-and-inheritance.md` — `final`, `readonly`, and extensibility rules
- `docs/09-tooling.md` — Pint, PHPStan, Rector, and enforcement expectations
- `docs/10-stubs.md` — architectural stubs, generation rules, and conflict resolution
- `docs/11-inertia-react.md` — Inertia + React: page structure, forms, and frontend boundaries
- `docs/12-workflow.md` — standard workflow for implementing tasks
- `docs/13-review-checklist.md` — acceptance criteria before opening a PR

Canonical chapter index and reading order: `docs/index.md`.

---

## Recipes

| File | When to use it |
|------|----------------|
| [`recipes/adopt-prototype-from-google-ai-studio.md`](recipes/adopt-prototype-from-google-ai-studio.md) | Port a simple React prototype (e.g. from Google AI Studio) into Inertia pages, routes, and Layer controllers; maps prototype screens to Laravel endpoints. |
| [`recipes/extend-domain-classes.md`](recipes/extend-domain-classes.md) | Flesh out the domain stack from DB schema (Data validation, Actions, Rules, Policies, Factories, Observers, enums/casts) for create/update/remove/read flows. |
| [`recipes/scaffold-new-laravel-project.md`](recipes/scaffold-new-laravel-project.md) | Bootstrap a new **Laravel 13** / **PHP 8.5** app with Sail (PostgreSQL, Redis), Inertia/React starter, Fortify, Spatie Data/ViewModels/Permission/Query Builder, Horizon, Pint, and Larastan. |
| [`recipes/generate-dashboard-crud-inertia.md`](recipes/generate-dashboard-crud-inertia.md) | Prompt scaffold for generating Dashboard-surface CRUD: controller, `IndexQuery`, ViewModels, and Export under `App/Layer/Dashboard/…` with Spatie Data on writes. |
| [`recipes/implement-page-from-prototype.md`](recipes/implement-page-from-prototype.md) | Full checklist to take a feature from the `prototype/` folder through Core domain, Layer delivery, Inertia UI, and wiring to production quality. |

---

## Precedence

1. Stubs define *how* code must look → `stubs/`
2. Docs define *why* and *where* → `docs/`

If stubs and docs conflict — follow the stub, then resolve the conflict explicitly.

---
