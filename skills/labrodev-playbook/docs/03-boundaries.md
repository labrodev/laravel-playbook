# Boundaries

This document defines architectural boundaries and dependency rules for Labrodev Laravel projects.

Boundaries exist to protect business logic, keep delivery layers replaceable, and prevent accidental coupling. When a boundary is violated, the code may still work today, but it will increase complexity, reduce clarity, and slow down change over time.

---

## 1) Dependency direction (non-negotiable)

Allowed dependencies:

- `App/Layer/*` depends on `Core/*`
- `Core/Domain/*` may depend on `Core/Shared`, `Core/Support`, and other `Core/Domain/*` modules (types, models, queries) when the relationship is explicit
- `Core/Feature/*` may depend on `Core/Domain/*`, `Core/Shared`, and `Core/Support`
- `Core/Shared` may depend on `Core/Support`

Forbidden dependencies:

- `Core/*` must not depend on `App/Layer/*`
- `Core/Shared` must not depend on any specific Domain
- `Core/Support` must not depend on Domain, delivery layers, or Feature

Cross-cutting behavior that does not belong to a single domain belongs in `Core/Feature/{FeatureName}/` with a small, explicit surface API—not buried in unrelated domains.

This dependency direction is the foundation of the architecture.

---

## 1b) Core/Feature for cross-domain workflows

Use `Core/Feature/{FeatureName}/` when:

- multiple domains must cooperate in one workflow, and
- the workflow is not a natural home inside any single `Core/Domain/{Domain}` module

Feature modules may contain the structure they need; they must not become a second place for ordinary domain rules. If a concept stabilizes, promote it into a proper Domain.

---

## 2) Responsibility boundaries by layer

### Delivery layer boundary (`App/Layer`)

Layer code may:
- handle HTTP and UI concerns
- receive raw input from controllers
- map input directly into Data objects (Spatie Data)
- prepare read-side data using IndexQueries
- delegate business behavior to Core Actions
- shape output using ViewModels, Resources, or exports

Layer code must not:
- implement business rules
- mutate domain state directly through models
- contain workflows or complex decision logic
- contain authorization rules that belong to Core policies

If business logic appears in `App/Layer`, it is a boundary violation.

---

### Core Domain boundary (business layer)

Domain code may:
- implement business rules, invariants, and calculations
- mutate state through Actions
- define domain events, policies, rules, and exceptions
- provide domain-specific read queries

Domain code must not:
- depend on controllers, UI, ViewModels, or exports
- depend on HTTP concepts or request lifecycle
- assume how input data was obtained or mapped
- return presentation-oriented objects

Domain logic must be expressible without Laravel HTTP context.

---

### Shared boundary (cross-domain abstractions)

Shared code may:
- provide base models, shared traits, and concerns
- provide generic abstractions reusable across domains

Shared code must not:
- encode domain-specific rules or terminology
- depend on a specific domain
- become a dumping ground for unrelated logic

If a class references a specific business concept (Product, Order, Invoice), it does not belong in Shared.

---

### Support boundary (technical utilities)

Support code may:
- provide technical helpers and infrastructure utilities
- include framework glue (time, UUIDs, serialization, formatting)
- solve purely technical problems

Support code must not:
- contain business rules
- reference domain language
- be used to bypass architectural boundaries

Support should remain boring and predictable.

---

### Feature boundary (isolated vertical logic)

Feature modules may:
- implement isolated, cross-domain logic
- contain whatever internal structure they need
- expose a small, explicit API to the rest of Core

Feature modules must not:
- become an alternative home for domain logic
- duplicate domain concepts or entities

If a Feature becomes a stable business concept, it should be promoted to a Domain.

---

## 3) Input mapping boundary (Data as input)

There are no Request classes in this architecture.

Input mapping rules:
- Controllers receive raw input
- Input is mapped directly into Data objects using Spatie Data
- Data objects define the input contract explicitly

Data objects:
- replace Request classes entirely
- handle validation and transformation
- act as the boundary between delivery and domain

Domain code must never depend on how Data objects are created.

---

## 4) Allowed call patterns (the golden path)

Write-side (mutations):

- Layer Controller -> Data object -> Domain Action -> Models
- Layer Controller -> Data object -> pipeline-orchestrating Service -> Pipeline steps -> Actions

Read-side (queries):

- Layer Controller/ViewModel -> Layer IndexQuery
- Domain internal reads -> Domain Queries

IndexQueries are delivery-layer concerns and must not be used from Core.

---

## 5) Rules for specific components

### Actions vs pipeline-orchestrating Services

- Action: a single, explicit business use case
- Pipeline-orchestrating Service: a Service (in `Services/`) that plays the orchestrator role — it coordinates multiple Actions or Services into a staged workflow by driving Pipeline steps under `Pipelines/{Workflow}/` with a `{Workflow}Payload`

Rules:
- Controllers call Actions or pipeline-orchestrating Services
- Jobs call Actions or pipeline-orchestrating Services
- Pipeline-orchestrating Services may call Actions
- Actions must not call controllers, jobs, or UI-related classes

---

### Jobs

Jobs are infrastructure wrappers for asynchronous execution.

Rules:
- Jobs contain no business logic
- Jobs delegate to Actions
- Retry, backoff, and queue configuration live in Jobs
- Business decisions must not be implemented inside Jobs

---

### Observers

Observers may enforce persistence invariants or synchronization rules.

Rules:
- Wire observers with `#[ObservedBy]` on the model (Laravel 13); do not call `observe()` from `boot()`
- Observers must not implement workflows
- Observers must not coordinate multiple Actions
- Observers must not dispatch business processes implicitly

If behavior matters, it must be explicit and callable.

---

### Events

Events describe facts: something happened.

Rules:
- Events contain no behavior
- Listeners may delegate to Actions
- Avoid long, implicit event chains that hide primary flows

Events should increase clarity, not obscure execution paths.

---

## 6) Boundary smells

Common signs of boundary violations:

- business logic inside controllers
- domain code importing `App/Layer` namespaces
- Shared code referencing domain terms
- Support utilities containing business rules
- Observers coordinating workflows
- Jobs implementing business decisions
- Domain returning ViewModels or Resources

These smells should trigger refactoring.

---

## Closing note

Boundaries protect long-term velocity.

When in doubt:
- keep business logic in Core/Domain
- keep delivery logic in `App/Layer`
- use Data objects as the input boundary
- keep Shared generic
- keep Support technical and boring