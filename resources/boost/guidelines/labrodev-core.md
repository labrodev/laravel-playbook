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

- `App/Layer/*` → `Core/*`. **Never the reverse** — no `Core/*` file imports **any `App\` class at all**: not `App\Layer\*`, not `App\Http\*`, not `App\Models\User`. The dependency arrow points one way only.
- **The user entity inside Core is the Core mirror model** — `Core/Domain/User/Models/User`, a normal Core model on the same `users` table (relations, casts, `$visible`, trio-wired like any model). `App\Models\User` exists only at the delivery/auth boundary (guards, Fortify, `#[CurrentUser]` in controllers, `actingAs()` in tests). The same mirror pattern applies to any App-owned entity Core must reference.
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
- Datetime type is `Illuminate\Support\Carbon` (explicit `->copy()` when mutation safety matters). **Never `CarbonImmutable`.** Obtain the current time as `Carbon::now()` — never the raw `now()` helper.
- `Core/Shared` stays generic, `Core/Support` stays technical — a class named with a domain noun belongs in that domain.
- **A simple static helper method is a Utility class**: domain-specific → `Core/Domain/{Domain}/Utilities`, purely technical → `Core/Support`. Static helpers never accumulate on Models, Actions, Services, or other classes (the only sanctioned static methods elsewhere: `{Model}Rule` gates and `::make()` factories).

## Contract commitment — no defensive hedging

Every data shape has exactly one declared contract: the Data class for user input, the DTO plus vendor docs for external payloads, the model for persisted state. Code commits to that contract.

- Read every value from exactly one contractual key or property. Never hedge with alternative-key fallbacks (`$p['first_name'] ?? $p['firstname']`), `looksLike*()` shape-guessing, or blanket "maybe" coercion (`stringOrNull()`, `is_numeric` guards on documented fields).
- Required data that is missing or malformed is a bug at the boundary — throw a named exception there (→ labrodev-exception). Silent coercion to `null` doesn't prevent the failure, it moves it into the database.
- A field is nullable only when its contract says so — never because the author is unsure of the shape. Unknown shape means: find the contract first (docs, a captured real payload, the existing mapper), then map. Don't guard against imagined variants.
- **No isset/is_string/is_array/trim scatter.** When code deals with an array, it checks exactly what it needs, once, at the boundary — not an endless chain of `isset`/`is_*`/null-coalescing silencing an unsure structure. Past the boundary, methods pass **strictly typed values** between each other (typed parameters, typed returns), so downstream code never re-checks.
- **Where a mapped structure must be guaranteed, introduce a typed DTO** — a `{Name}Payload` (or `{Name}Envelope` when wrapping with transport metadata → labrodev-infrastructure) — and map into it once, failing loud. The DTO is the proof of shape; defensive re-checks after it are dead code.

## Must-nots

- No business logic in `App/Layer` — no calculations affecting stored values, business branching, or domain mutation in controllers, IndexQueries, ViewModels, Exports.
- No `Actions/`, `Services/`, `Queries/`, `Rules/`, or `Policies/` folders under `App/Layer`.
- **No concept-free folders** inside `Core/Domain/{Domain}`, `Core/Feature/{Feature}`, `Core/Infrastructure/{Integration}`, or `App/Layer/{Layer}/{Domain}` — every subfolder names a playbook class-type concept (Actions, Rules, Queries, Utilities, ...). A domain-local `Support/` folder is the canonical offender: those classes belong in `Utilities/`. (Top-level `Core/Support` is the one Support that exists.)
- No mass assignment anywhere: no `fill()`, no `Model::create()`, no `$fillable`.
- No manual resolution of Actions (`app()`, `resolve()`, `new`) — inject as method arguments.
- **No comments inside any class — non-negotiable.** No `//` line comments, no `/* */` blocks, no prose docblocks anywhere in a class body. Naming carries the intent; anything needing explanation belongs in better names or its own atomic class. Docblocks exist solely as typed PHPStan annotations (`@return`, `@param`, `@extends`, `@use`, `@mixin`, `@throws`, `@template` generics) with zero prose (models stricter still → labrodev-model).
- No interface bindings or provider registrations unless the plan names the contract, the implementation, and the wiring. Prefer concrete constructor injection.
- No nullable-widening of required domain parameters (`?Booking` to hide a caller bug) — fix the caller.
- No frontend workarounds for missing backend behavior.
- No app-specific ownership/scoping vocabulary in playbook code — record such conventions as project rules in your app (see below).

## Anti-patterns (refactor on sight)

Controllers that do work · hidden workflows in Observers · Jobs containing business logic · Actions with `execute()`/`handle()` or multiple entry points · raw arrays passed into Core · Shared/Support as dumping grounds · generic names (`Manager`, `Handler`, `Processor`, `Util`) · critical flows implemented via event-listener chains · Domain Query methods returning resolved results instead of `Builder` · raw models or sensitive fields in Inertia props · hedged input mapping — alternative-key fallbacks (`$p['first_name'] ?? $p['firstname']`), `looksLike*()` shape-guessing, `stringOrNull()`-style coercion that turns missing required data into silent `null` · private/static-method spaghetti — a flow chopped into single-use `private` or `static` helpers, or chains of static methods calling further static methods, hiding the reading order; logic worth extracting becomes its own atomic class (Action, Rule, Service, Utility), otherwise it stays inline.

## Legacy zones

Framework/starter scaffolding under `app/Http`, `app/Models`, `app/Actions/Fortify` is exempt from playbook rules — never flag it, never refactor it unasked, never add new business code there. New behavior is built in Core and delegated to.

## Your app's own rules

Conventions specific to your application (scoping, base classes, local deviations) are recorded via Boost's `record-rule` into your app's `.ai/rules` — never edited into playbook files. Where an app rule conflicts with a playbook default, the app rule wins.
