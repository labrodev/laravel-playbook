# Labrodev Actions, Services & Rules

Always-on law. The **write side** of a domain lives in Core: Actions (business use cases that mutate state), Services (reusable domain logic), and `{Model}Rule` classes (pure can-I checks). Staged multi-step workflows are pipeline territory (→ labrodev-pipeline). Anatomy and templates → the `labrodev-action` skill; per-file checks → `rules/actions.md`.

## Musts

- Actions live **only** in `Core/Domain/{Domain}/Actions/` — `App/Layer` never defines Action classes. Controllers map input into Data objects and delegate every mutation to a Core Action.
- **An Action is a singular mutation of ONE model object**: create, update, or remove of a single `{Model}` instance. That is the entire scope of the class type.
- **Action names always end in `Create`, `Update`, or `Remove`** — `{Model}Create`, `{Model}Update`, `{Model}Remove`; partial/state updates take an aspect infix, still ending in the verb (`BookingStatusUpdate`, `MonitorEnabledStatusUpdate`). No other verb suffixes exist for Actions.
- Every Action is `final readonly` with a **single public `__invoke()`** entry point; private helper methods are allowed.
- **The can-trio is enforced at Policy level**: `{Model}Rule::canCreate` / `canUpdate` / `canRemove` are called from the corresponding Policy methods, so an ineligible request is rejected (403) **before** any Action runs (→ labrodev-authorization). Actions never re-run the can-trio.
- **Guard-then-mutate ordering**: when an Action guards a business invariant the Policy cannot see (conditions depending on submitted input or cross-record state at write time), the guard runs BEFORE any mutation, delegates the condition to a static `{Model}Rule` method — never inlined — and **throws a dedicated domain exception via `::make()`** on failure (→ labrodev-exception).
- **Rule gate methods mirror the operation verb**: `can{Verb}` (`canCreate`, `canUpdate`, `canRemove`); finer-grained invariant checks get descriptive names (`periodIsAvailable`, `projectBelongsToTeam`).
- Create Actions wrap the write in `DB::transaction(...)`; the closure is typed and **returns the model**. Any Action performing multiple writes is also transaction-wrapped.
- Create Actions assign the UUID explicitly — `$booking->uuid = (string) Str::uuid();` — never in the model, `boot()`, or a trait.
- Attributes are assigned **explicitly, one by one**, from the typed Data object — mass assignment is banned architecture-wide (→ labrodev-core).
- Exactly **one Rule class per model** (`BookingRule` for `Booking`) — the single source of truth for "can I do X with this model?", shared by Policies, Actions, Observers, and Pipeline steps.
- Rule methods are `public static`, pure checks: they return booleans or small decision values and may run complex conditions — reads go through the model's Query class (→ labrodev-query).
- Services express domain language, live in `Core/Domain/{Domain}/Services/`, and expose one public `__invoke()` for a single operation or explicit named methods for multiple operations — never `execute()` / `handle()` as generic method names on a Service.
- **Service names end in an `-er` agent noun** naming the manipulation inside: `BookingPriceCalculator`, `EmailSender`, `BookingSearcher`, `UncoveredMonitorDisabler`. The one ruled exception is the pipeline-orchestrating `{Workflow}Service` (→ labrodev-pipeline).
- **Services stay atomic.** A Service that accumulates a lot of logic gets decomposed — into smaller `-er` Services, Utilities, Rules, and Actions it coordinates, or staged as a pipeline (→ labrodev-pipeline). One manipulation per Service, glance-sized (→ labrodev-core philosophy).
- **Jobs declare their settings via class attributes** — `#[OnQueue(...)]`, `#[OnConnection(...)]`, `#[WithoutRelations]`, `#[DeleteWhenMissingModels]` — never via public properties or constructor calls; same attribute-wiring style as models (→ labrodev-model). A Job stays a thin async wrapper delegating to an Action or Service.
- **Notifications always travel the chain Event → Listener → queued Job → Notification**: the domain dispatches a past-tense fact Event; a Listener reacts by dispatching the Job; the Job sends the Notification. Never send a Notification inline from an Action, Service, controller, or Observer. (Critical business flow still never rides event chains — this chain is for notifications precisely because they are non-critical side effects → labrodev-core anti-patterns.)

## Must-nots

- **Actions never throw `Illuminate\Validation\ValidationException` — ever.** That class belongs exclusively to the Data validation layer (→ labrodev-data). A failed business gate is a business failure, not a validation failure: it throws that condition's own exception class (`BookingPeriodUnavailableException::make(...)`), never `ValidationException::withMessages(...)`.
- No silent early-return no-ops on failed guards — a guard that fails throws its named exception. Silent skips hide bugs from callers outside HTTP (pipelines, jobs, console).
- **No batch or multi-object Actions.** The moment a use case modifies several objects, it is not an Action — it is a **Service that calls Actions** (one per object mutation), or a pipeline for staged workflows (→ labrodev-pipeline).
- **No `->update([...])` (or `->delete()`) on a query Builder or Collection — ever.** No mass updates via query chain (`Booking::where(...)->update([...])`), no single updates via chain, no `$collection->each->update(...)`. Every modification loads the exact model object(s) and runs the singular Action per model — that is the only write path.
- No mutation before guards; no guards after `save()`.
- Rules never mutate state or trigger Actions, Jobs, workflows, or any other side effect.
- No `fetchRuleClass()` (or similar indirection) on the model — import `{Model}Rule` directly where needed.
- Services never mutate state when that mutation is a business use case — delegate it to an Action.
- Actions never call pipeline-orchestrating Services — pipelines call Actions, not the reverse (→ labrodev-pipeline).
- No presentation concerns (Resources, Inertia, HTTP) anywhere on the write side.
- App-specific ownership/scoping concerns never appear in playbook Actions (→ labrodev-core).
