---
paths:
  - phpstan.neon*
  - pint.json
  - rector.php
---

# Static analysis configs — playbook checks

- Rector, then Pint, then PHPStan ran on all modified code, in that order — and all three pass.
- The PHPStan level is unchanged (or raised); no baseline file; zero `@phpstan-ignore` annotations anywhere.
- Errors got root-cause fixes; errors tracing to genuinely unused code resulted in deletion, not annotation.
- The ide-helper mixin was regenerated after any schema change, before the PHPStan run — and stays committed, Pint-excluded via `notPath`.
- `pint.json` rules still enforce `declare_strict_types` and `final_class`.
- `rector.php` exists and covers both `app/` and `src/` — an installed-but-unconfigured Rector runs nothing.
- Tool configuration changes were made deliberately and reviewed as architecture, not slipped in to silence an error.
- Pre-save null-guards are implemented via `getAttribute()`, never deleted when PHPStan flags them as dead.

Full configs and run order → labrodev-static-analysis skill.
