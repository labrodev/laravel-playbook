# Labrodev Authorization

Always-on law. Authorization is one cluster of three pieces wired together: a **Policy** (`{Model}Policy` in Core, answering "may this user perform this action on this object?" in business terms), `#[UsePolicy(...)]` on the model, and a class-level `#[Authorize(...)]` on every invokable controller. Anatomy and templates → the `labrodev-authorization` skill; per-file checks → `rules/policies.md`.

## Musts

- Policies live in `Core/Domain/{Domain}/Policies/` and are named `{Model}Policy` (e.g. `BookingPolicy`).
- Every permission is declared as a `public const string` on the policy, and **the constant VALUE must equal the policy METHOD name** — `'view'` resolves to `view()`. This is non-negotiable.
- Attach the policy to its model with `#[UsePolicy({Model}Policy::class)]` (`Illuminate\Database\Eloquent\Attributes\UsePolicy`) on the model class.
- Every invokable controller carries a class-level `#[Authorize(...)]` attribute (`Illuminate\Routing\Attributes\Controllers\Authorize`) so the gate runs as controller middleware — Inertia controllers **and** JsonControllers alike (→ labrodev-controller).
- Policy methods layer checks in a fixed order: **non-null user → ownership/scope → domain `{Model}Rule` gate**.
- Object-state gates (`isEditable`, `canBeDeleted`) are delegated to the domain Rule class, called statically and directly: `BookingRule::canBeDeleted($booking)` — never via a helper on the model.
- Define **only** the permissions that exist for the model's real use cases. The set is not fixed; it is not always view/create/update/remove (an append-only log model may have only `PERMISSION_VIEW` and `PERMISSION_CREATE`).
- **Every endpoint must be covered.** A controller without `#[Authorize]` (and without the documented runtime-setup exception) is a review blocker, not a style nit.

## Must-nots

- Never register policies via `Gate::policy(...)` in a service provider. `#[UsePolicy]` on the model is the only wiring mechanism.
- Never use a dotted RBAC key (e.g. `'booking.bookings.view'`) as a permission constant value — a dotted value cannot resolve to a policy method through the Gate. Dotted keys live **inside** method bodies only, via `$user->hasPermissionTo(...)`.
- Policies must not mutate state, implement workflows, call Actions/Jobs, or duplicate business logic already expressed in Rules.
- Never call `$this->authorize(...)` or `Gate::authorize(...)` inside `__invoke()` when the subject is known up front — use the class-level attribute. The in-body form is reserved for the runtime-setup exception.
- The ownership/scope check is a project-specific slot (shown commented in the skill's policy template) — the playbook policy template itself carries no app-specific scoping logic.
