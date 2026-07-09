# Architecture Stubs (Source of Truth)

This document defines the **authoritative role of architectural stubs**
in the Labrodev system.

Architectural stubs are the **single source of truth**
for how code in this project must be structured.

They are **normative**, not illustrative.

---

## Location and scope

All architectural stubs live in `stubs/`, relative to the playbook root.

Stub paths mirror the target namespace and directory structure exactly.

No stub may generate files outside its declared scope.

This document governs:
- all handwritten code
- all generated code
- all AI-assisted code

---

## Role of stubs

Stubs define the **canonical structure** of classes in this system.

They define:
- file placement
- directory layout
- class name shape
- required modifiers (`final`, `readonly`)
- constructor signature
- method names
- method visibility
- allowed dependencies
- extension points (if any)

Stubs do **not** define business logic.

---

## What stubs are used for

Stubs are used for:
- code generation
- architectural consistency
- AI-assisted development

If documentation explains *why*, stubs define *how*.

---

## Normative authority

Stubs are authoritative.

This means:

- Stubs must be followed exactly
- Stubs must not be simplified
- Stubs must not be reinterpreted
- Stubs must not be partially applied
- Stubs must not be duplicated in documentation

If architecture changes:
- stubs must be updated first
- documentation must be updated second
- code must be regenerated or aligned last

Modifying code without updating the relevant stub is forbidden.

---

## Mandatory AI usage rules

AI systems must treat stubs as **executable specifications**.

When generating or modifying code, AI must follow this sequence:

1. Identify the required class type  
   (Action, Data, Model, Service, Query, etc.)

2. Locate the governing stub in `stubs/`

3. Use the stub as the **structural template**
   for the file

4. Apply domain-specific logic **only inside**
   the structural boundaries defined by the stub

5. Preserve all required modifiers, signatures, visibility,
   and method names

Skipping or reinterpreting any step is forbidden.

Additionally, naming conventions defined by the playbook are part of the stub contract:
- Prefer `{Model}CreateData` and `{Model}UpdateData` when store and update shapes differ (required vs nullable fields, different relations, different validation). A single `{Model}Data` is fine when the shape is truly the same; when in doubt, split — obscuring divergent contracts behind nullable fields in one class is worse than two small Data classes.
- A typed DTO parameter must use the class short name in camelCase exactly:
  `DraftLogData $draftLogData`, `CustomerData $customerData`
- Never shorten typed DTO parameters to `$data`

---

## Strict prohibitions for AI

AI must **never**:

- invent a new class structure
- merge multiple stubs into one file
- introduce public methods not defined by the stub
- remove required methods
- rename methods defined by the stub
- alter constructor shape
- change method visibility
- omit required `final` or `readonly` modifiers
- move a class outside the directory implied by its stub
- generate code when no suitable stub exists

If no suitable stub exists, AI must **stop**
and request explicit guidance.

---

## Modification rules

When modifying existing code:

- AI must first identify which stub governs the file
- All changes must remain compatible with that stub
- If a change requires altering the stub:
    - the stub must be updated first
    - this document must be consulted
    - documentation must be aligned
    - code must be regenerated or updated accordingly

Silent deviation is not allowed.

---

## Stubs vs documentation

- Documentation explains intent, rationale, and boundaries
- Stubs define exact implementation structure

If a stub and a documentation chapter appear to conflict:

1. Consult this document
2. Default to the stub
3. Resolve the conflict explicitly
4. Update the stub if the architecture has changed

---

## Tooling compatibility

All code produced using stubs must:

- be compatible with Pint
- pass PHPStan at the configured level
- be compatible with Rector rules
- avoid suppressed errors or undocumented exceptions

Tooling compliance is mandatory and non-negotiable.

---

## Stub inventory

### Core / Domain

- `stubs/core/domain/actions/create.stub`
- `stubs/core/domain/actions/update.stub`
- `stubs/core/domain/actions/remove.stub`
- `stubs/core/domain/casters/collectionCaster.stub`
- `stubs/core/domain/casters/objectCaster.stub`
- `stubs/core/domain/collections/collection.stub`
- `stubs/core/domain/data/data.stub`
- `stubs/core/domain/enums/enum.stub`
- `stubs/core/domain/events/event.stub`
- `stubs/core/domain/exceptions/exception.stub`
- `stubs/core/domain/factories/factory.stub`
- `stubs/core/domain/jobs/job.stub`
- `stubs/core/domain/models/model.stub`  
  (Laravel 13: `#[Table('{table}')]`, `#[ObservedBy]`, `#[CollectedBy]`, `#[UsePolicy]`, `#[UseFactory]`; replace `{table}` when generating. Models carry NO `@property` lists and NO comments — column metadata comes from the ide-helper mixin; relation methods DO carry `@return` generics docblocks; see `docs/05-database-and-models.md`)
- `stubs/core/domain/observers/observer.stub`
- `stubs/core/domain/orchestrators/orchestrator.stub`
- `stubs/core/domain/payloads/payload.stub`
- `stubs/core/domain/pipelines/pipeline.stub`
- `stubs/core/domain/policies/policy.stub`
- `stubs/core/domain/queries/query.stub`
- `stubs/core/domain/resources/resource.stub`
- `stubs/core/domain/rules/rule.stub`
- `stubs/core/domain/services/service.stub`
- `stubs/core/domain/utilities/utility.stub`

### Core / Shared

- `stubs/core/shared/concerns/aware.stub`
- `stubs/core/shared/models/baseModel.stub`

### App layer

- `stubs/app/api/controllers/controller.api.stub`  
  (invokable API controllers: class-level **`#[Authorize(...)]`**; write-side uses Data objects; read-side uses no Request classes and no Data classes.)
- `stubs/app/dashboard/controllers/controller.inertia.stub`  
  (invokable READ controller: class-level **`#[Authorize({Model}Policy::…, {Model}::class)]`**.)
- `stubs/app/dashboard/controllers/controller.inertia.write.stub`  
  (invokable WRITE controller: Data + Action injected, Action invoked as a callable with named arguments, `Inertia::flash('toast', …)` + `to_route(...)`.)
- `stubs/app/dashboard/jsonControllers/jsonController.stub`  
  (same: class-level **`#[Authorize]`**.)
- `stubs/app/dashboard/exports/export.stub`
- `stubs/app/dashboard/indexQueries/indexQuery.stub`
- `stubs/app/dashboard/viewModels/viewModel.general.stub`
- `stubs/app/dashboard/viewModels/viewModel.inertia.index.stub`
- `stubs/app/dashboard/viewModels/viewModel.inertia.show.stub`

### Routes 

- `stubs/routes/routes.stub`
