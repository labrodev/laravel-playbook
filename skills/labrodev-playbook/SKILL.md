---
name: labrodev-playbook
description: "Use this skill whenever working on a Labrodev Laravel project. Covers the full architecture: project structure, naming conventions, domain boundaries, data and validation, models and Eloquent, testing strategy, anti-patterns, immutability, tooling (Pint/PHPStan/Rector), stubs, Inertia+React conventions, workflow, and review checklist. Apply before writing any code, generating any class, reviewing any PR, or answering any architecture question."
license: MIT
metadata:
  author: labrodev
---

# Labrodev Playbook (skill)

Use **`index.md`** as the human-readable table of contents. This file repeats the **Documentation map**, **Recipes**, and **Precedence** so the skill is self-contained when loaded without the rest of the repo tree.

---

## Documentation map

- `docs/00-philosophy.md` — core principles, values, and non-negotiables
- `docs/01-project-structure.md` — folder layout, layers, and responsibility boundaries
- `docs/02-naming.md` — naming rules for domains, classes, methods, variables
- `docs/03-boundaries.md` — dependency direction and forbidden couplings
- `docs/04-data-and-validation.md` — input mapping, validation, write/read separation
- `docs/05-database-and-models.md` — models, Eloquent attributes, collections, observers
- `docs/06-testing.md` — testing strategy, structure, and allowed patterns
- `docs/07-anti-patterns.md` — explicitly forbidden approaches and shortcuts
- `docs/08-immutability-and-inheritance.md` — `final`, `readonly`, and extensibility rules
- `docs/09-tooling.md` — Pint, PHPStan, Rector, and enforcement expectations
- `docs/10-stubs.md` — architectural stubs, generation rules, and conflict resolution
- `docs/11-inertia-react.md` — Inertia + React: page structure, forms, and frontend boundaries
- `docs/12-workflow.md` — standard workflow for implementing tasks
- `docs/13-review-checklist.md` — acceptance criteria before opening a PR

Canonical chapter index: `docs/index.md`.

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
