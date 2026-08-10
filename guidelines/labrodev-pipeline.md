# Labrodev Pipelines — staged workflows

Always-on law. A staged workflow — several distinct side-effecting steps that run as one business scenario — is a trio: a `{Workflow}Payload` (flow state), verb-first step classes under `Pipelines/{Workflow}/`, and an orchestrating `{Workflow}Service`; single-domain trios live in `Core/Domain/{Domain}/`, cross-domain ones in `Core/Feature/{FeatureName}/`. Anatomy and templates → the `labrodev-pipeline` skill; per-file checks → `rules/pipelines.md`.

## Musts

- **The orchestrating Service** exposes a single public `__invoke(...)`, builds the Payload via `{Workflow}Payload::make(...)`, runs `app(Pipeline::class)->send($payload)->through([...])->thenReturn()`, validates the result with an `instanceof` check throwing `PipelinePayloadIncorrect::make(...)`, and returns the final value from the Payload.
- **Each Pipeline step does one atomic thing**, mutates the Payload, and returns `$next($payload)`. Steps are `final readonly` with a single `handle({Workflow}Payload $payload, Closure $next): mixed` method.
- **Steps are named verb-first, no suffix** (`CreateCustomer`, `SendBookingConfirmationMail`, `PushBookingToCrm`) and live under `Pipelines/{Workflow}/` — a subfolder named after the workflow.
- **Mutations happen through Actions.** A step that persists something calls the domain Action — it never writes models directly. Each Action manages its own transaction (→ labrodev-action).
- **Persistence before external side effects.** Order the steps so DB-mutating steps run first; mail, CRM pushes, and other external calls run after domain state is safely persisted. Never wrap the whole pipeline in one `DB::transaction()` — external calls do not belong inside a transaction.
- **External integrations are reached through Infrastructure contracts** (e.g. a `CrmClient` interface from `Core/Infrastructure/...`), never called inline with HTTP code in a step.
- **The Payload is a mutable `final class`** (NOT readonly — it is the one sanctioned mutability exception) with public flow-state properties and a static `make()` constructor that validates prerequisites.

## Must-nots

- No business branching in the orchestrating Service beyond guard clauses and step selection — logic lives in the steps and the Actions they call.
- No step that does two things ("create customer and send mail") — split it.
- No pipelines inside pipelines; if a step needs its own staged workflow, that workflow is its own Service the step calls.
- No swallowing step failures: a failing step throws; slow/unreliable external steps may dispatch a queued Job instead of calling synchronously.
- No Payload reuse across workflows — one Payload class per workflow, named after it.
