---
paths:
  - "**/Core/Domain/**/Actions/**"
  - "**/Core/Domain/**/Services/**"
  - "**/Core/Domain/**/Rules/**"
---

# Actions & Services — playbook checks

- Universal: file opens with `declare(strict_types=1)` and a namespace matching its folder; the class is `final`; names are singular; the class body contains no comments — typed PHPStan annotations only.
- Actions live in `Core/Domain/{Domain}/Actions/`, are `final readonly`, and expose exactly one public `__invoke()`.
- Every Action performs a **singular mutation of one model object**, and its name ends in `Create`, `Update`, or `Remove` (aspect infix allowed: `BookingStatusUpdate`) — no other verb suffixes, no batch/multi-object Actions.
- The can-trio (`canCreate`/`canUpdate`/`canRemove`) is NOT checked in the Action — it is enforced at Policy level (→ rules/policies.md).
- Remaining invariant guards (input-dependent, cross-record) run before any mutation, delegate to static `{Model}Rule` methods, and **throw a dedicated domain exception via `::make()`** — never `Illuminate\Validation\ValidationException`, never a silent early return.
- Create Actions wrap the write in `DB::transaction` with a typed closure returning the model, and assign the UUID via `(string) Str::uuid()` inside it.
- All attributes are assigned explicitly from the typed Data object — no `fill()`, `create()`, `update([...])`, or mass assignment.
- No Builder/Collection `->update([...])` or `->delete()` chains anywhere — mass and single modifications alike load the exact model(s) and run the per-model Action.
- Exactly one Rule class per model, with static, pure, side-effect-free methods; any DB read inside a Rule goes through the model's Query class, never inline `Model::query()`.
- Batch/multi-object modifications are Services that invoke the singular Actions; any Service coordinating multiple Actions in stages goes through Pipelines (→ labrodev-pipeline).
- Services are free of business-use-case mutations of their own.
- Notifications follow the chain Event → Listener → queued Job → Notification — no inline `->notify()` / `Notification::send()` in Actions, Services, or Observers.
- Jobs declare queue settings via class attributes (`#[OnQueue]`, `#[OnConnection]`, ...) — no `public $queue` properties or constructor `onQueue()` wiring.
- Payloads are data-only — public flow-state properties plus a `make()` factory, no behavior, no anonymous arrays between steps (→ labrodev-pipeline).
- No `App\` imports anywhere in Core — the user entity in Core is the Core mirror User model (→ labrodev-core).

Full anatomy and templates → labrodev-action skill.
