# Labrodev Core — structure, boundaries, global code rules

Always-on law for every Labrodev Laravel project (Laravel 13+, PHP 8.5). Anatomy, templates, and worked examples → the `labrodev-core` skill.

## Philosophy

- **Keep it simple, smart**: intentional simplicity — explicit over magic, boring over smart, duplication over premature abstraction.
- **Atomic classes**: every class does exactly one thing and is small enough to understand at a glance.
- **Every class is a puzzle piece**: typed inputs in, typed output out; swappable without reshaping its neighbors.
- **Vertical slices**: organized by business domain, not technical layer — a feature maps to one slice.
- **Single source of truth**: mutations in Actions, reads in Query classes, business gates in Rules, input validation in Data, authorization in Policies, presentation in ViewModels/Resources.

## Structure

```
src/Core/
├── Domain/{Domain}/       business logic: Actions, Data, Enums, Events, Exceptions,
│                          Models, Observers, Payloads, Pipelines, Policies, Queries,
│                          Rules, Services, Casts, Collections, Traits, Utilities
├── Feature/{FeatureName}/ isolated cross-domain workflows
├── Infrastructure/{Integration}/  external adapters — technical, never business
├── Shared/                generic base abstractions; no domain nouns
└── Support/               technical helpers only; no business logic

app/Layer/{Layer}/{Domain}/  delivery surfaces (Dashboard, Api):
└── Controllers/ JsonControllers/ IndexQueries/ ViewModels/ Resources/ Exports/
```

Core may physically live in a separate package (`src/Core`, `../core/src`) — rules apply to the logical module regardless.

## Dependency law

- `App/Layer/*` → `Core/*`. **Never the reverse** — no `Core/*` file imports `App\Layer\*`.
- `Core/Domain/*` → `Core/Shared`, `Core/Support`, other Domains (when the relationship is explicit), and `Core/Infrastructure` only via explicit contracts (contract + adapter + resolver).
- `Core/Feature/*` → `Core/Domain/*`, `Core/Shared`, `Core/Support`. `Core/Shared` → `Core/Support` only — never a Domain. `Core/Support` depends on nothing above it.
- Domain code never depends on HTTP concepts, the request lifecycle, ViewModels, Resources, or exports — Resources are Layer classes (→ labrodev-viewmodel-resource).
- Business logic lives only in Core (Actions, Services, Rules, Policies). A Layer is a delivery surface, never a second business domain — two Layers needing the same logic means it belongs in Core.

## Golden paths

- Write: `Layer Controller → Data object → Core Action → Models` (staged workflows: `→ pipeline-orchestrating Service → Pipeline steps`).
- Read: `Layer Controller → Layer IndexQuery → ViewModel → Resource → response`; Core-internal reads → Core Domain Queries.

## Global code rules

- Every PHP file: `<?php`, blank line, `declare(strict_types=1);`, blank line, `namespace` matching the folder path.
- Every concrete class is `final`; modifier order `final readonly` (readonly when no mutable state — Actions, Services, Events). No `readonly` on Spatie Data subclasses, framework-extending classes, or static-only classes.
- Datetime type is `Illuminate\Support\Carbon` (explicit `->copy()` when mutation safety matters). **Never `CarbonImmutable`.**
- `Core/Shared` stays generic, `Core/Support` stays technical — a class named with a domain noun belongs in that domain.

## Must-nots

- No business logic in `App/Layer` — no calculations affecting stored values, business branching, or domain mutation in controllers, IndexQueries, ViewModels, Exports.
- No `Actions/`, `Services/`, `Queries/`, `Rules/`, or `Policies/` folders under `App/Layer`.
- No mass assignment anywhere: no `fill()`, no `Model::create()`, no `$fillable`.
- No manual resolution of Actions (`app()`, `resolve()`, `new`) — inject as method arguments.
- No comments that restate code — only essential, non-obvious context.
- No interface bindings or provider registrations unless the plan names the contract, the implementation, and the wiring. Prefer concrete constructor injection.
- No nullable-widening of required domain parameters (`?Booking` to hide a caller bug) — fix the caller.
- No frontend workarounds for missing backend behavior.
- No app-specific ownership/scoping vocabulary in playbook code — record such conventions as project rules in your app (see below).

## Anti-patterns (refactor on sight)

Controllers that do work · hidden workflows in Observers · Jobs containing business logic · Actions with `execute()`/`handle()` or multiple entry points · raw arrays passed into Core · Shared/Support as dumping grounds · generic names (`Manager`, `Handler`, `Processor`, `Util`) · critical flows implemented via event-listener chains · Domain Query methods returning resolved results instead of `Builder` · raw models or sensitive fields in Inertia props.

## Legacy zones

Framework/starter scaffolding under `app/Http`, `app/Models`, `app/Actions/Fortify` is exempt from playbook rules — never flag it, never refactor it unasked, never add new business code there. New behavior is built in Core and delegated to.

## Your app's own rules

Conventions specific to your application (scoping, base classes, local deviations) are recorded via Boost's `record-rule` into your app's `.ai/rules` — never edited into playbook files. Where an app rule conflicts with a playbook default, the app rule wins.
