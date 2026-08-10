# Labrodev Static Analysis — Pint, PHPStan, Rector

Always-on law. The toolchain is part of the architecture, not optional: Pint, PHPStan (Larastan), and Rector enforce formatting, types, and modern syntax automatically so humans review architecture and behavior instead. If a change requires fighting the tools, reconsider the design — not the tool. Anatomy and configs → the `labrodev-static-analysis` skill; per-file checks → `rules/static-analysis.md`.

## Musts

- **All three tools run on the modified code after the work is done, before commit/PR** — and all three must pass.
- **Run in this order: Rector → Pint → PHPStan.** Rector rewrites code (needs a style pass afterwards); PHPStan verifies the final result.
- **Tool configuration files** (`pint.json`, `phpstan.neon`, `rector.php`) are canonical and live in the repo root. Changing them is an architectural decision, not a convenience fix.
- **PHPStan runs with Larastan** and must pass at the configured level for **all** modified code. Type safety beats convenience.
- **The ide-helper mixin** (`_ide_helper_models.php`) must be current before analysing — it carries model column metadata (→ labrodev-model).
- **Dead code found via PHPStan gets deleted** — orphaned tests for never-created classes, zero-call-site services, dead configs. Never annotate around it or leave a stub; if it is never used anywhere, remove it.

## Must-nots

- Never hand-fix what Pint fixes — run the tool. Pint's output is never "reformatted back".
- Never suppress a PHPStan error: no `@phpstan-ignore`, no baseline file, no level drop. Fix the root cause; if that seems impossible, the design is wrong — reconsider it.
- Never ignore a Rector suggestion without a reason. If a Rector rule fights the playbook, exclude the rule in `rector.php` explicitly, with the reason in the PR that excludes it — do not skip runs.
- Never delete a "dead" null-guard in `creating()`/`saving()` hooks just because PHPStan flags it — read via `getAttribute()` instead (trap anatomy → labrodev-static-analysis skill).
- Never run tools only on the happy path: exports, jobs, observers, and casters are modified code too.
