# Labrodev Actions, Services & Rules

Always-on law. The **write side** of a domain lives in Core: Actions (business use cases that mutate state), Services (reusable domain logic), and `{Model}Rule` classes (pure can-I checks). Staged multi-step workflows are pipeline territory (→ labrodev-pipeline). Anatomy and templates → the `labrodev-action` skill; per-file checks → `rules/actions.md`.

## Musts

- Actions live **only** in `Core/Domain/{Domain}/Actions/` — `App/Layer` never defines Action classes. Controllers map input into Data objects and delegate every mutation to a Core Action.
- Every Action is `final readonly` with a **single public `__invoke()`** entry point; private helper methods are allowed.
- **Rule-then-mutate ordering**: business-condition guards run BEFORE any mutation and delegate to static `{Model}Rule` methods — never inline the condition in the Action.
- Create Actions wrap the write in `DB::transaction(...)`; the closure is typed and **returns the model**. Any Action performing multiple writes is also transaction-wrapped.
- Create Actions assign the UUID explicitly — `$booking->uuid = (string) Str::uuid();` — never in the model, `boot()`, or a trait.
- Attributes are assigned **explicitly, one by one**, from the typed Data object — mass assignment is banned architecture-wide (→ labrodev-core).
- Exactly **one Rule class per model** (`BookingRule` for `Booking`) — the single source of truth for "can I do X with this model?", shared by Policies, Actions, Observers, and Pipeline steps.
- Rule methods are `public static`, pure checks: they return booleans or small decision values and may run complex conditions and queries.
- Services express domain language, live in `Core/Domain/{Domain}/Services/`, and expose one public `__invoke()` for a single operation or explicit named methods for multiple operations — never `execute()` / `handle()` as generic method names on a Service.

## Must-nots

- No mutation before guards; no guards after `save()`.
- Rules never mutate state or trigger Actions, Jobs, workflows, or any other side effect.
- No `fetchRuleClass()` (or similar indirection) on the model — import `{Model}Rule` directly where needed.
- Services never mutate state when that mutation is a business use case — delegate it to an Action.
- Actions never call pipeline-orchestrating Services — pipelines call Actions, not the reverse (→ labrodev-pipeline).
- No presentation concerns (Resources, Inertia, HTTP) anywhere on the write side.
- App-specific ownership/scoping concerns never appear in playbook Actions (→ labrodev-core).
