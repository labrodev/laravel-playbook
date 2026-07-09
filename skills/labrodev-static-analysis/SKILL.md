---
name: labrodev-static-analysis
description: "Use when configuring, running, or fixing findings from the code-quality toolchain in a Labrodev Laravel project — Pint (pint.json, code style), PHPStan/Larastan (phpstan.neon, levels, baselines, generics errors), or Rector (rector.php, automated refactoring) — or when deciding whether a tool error may be suppressed."
license: MIT
metadata:
  author: labrodev
---

# Static Analysis: Pint, PHPStan, Rector

Part of the Labrodev playbook skill set — assumes labrodev-core and labrodev-naming are installed. If absent, minimum global rules: `declare(strict_types=1)`, final classes, App/Layer depends on Core (never the reverse), named-argument invocation.

These tools are not optional. They are part of the architecture: they reduce review noise, prevent trivial mistakes, and enforce consistency automatically so humans review architecture and behavior instead of formatting and types. If a change requires fighting the tools, reconsider the design — not the tool.

## Musts

- All three tools run on the modified code **after the work is done, before commit/PR** — and all three must pass. When the gate fires → see the labrodev-workflow skill.
- Run in this order: **Rector → Pint → PHPStan**. Rector rewrites code (needs a style pass afterwards); PHPStan verifies the final result.
- Tool configuration files (`pint.json`, `phpstan.neon`, `rector.php`) are canonical and live in the repo root. Changing them is an architectural decision, not a convenience fix.
- PHPStan runs with **Larastan** and must pass at the configured level for **all** modified code. Type safety beats convenience.
- The ide-helper mixin (`_ide_helper_models.php`) must be current before analysing — it carries model column metadata → see the labrodev-model skill.

## Must-nots

- Never hand-fix what Pint fixes — run the tool.
- Never suppress a PHPStan error (`@phpstan-ignore`, baseline entry, level drop) without a strong, **stated** justification in the PR. A suppression without a reason is a defect.
- Never ignore a Rector suggestion without a reason. If a Rector rule fights the playbook, exclude the rule in `rector.php` explicitly — do not skip runs.
- Never commit a grown baseline: `phpstan-baseline.neon` may only shrink. New code adds zero baseline entries.
- Never run tools only on the happy path: exports, jobs, observers, and casters are modified code too.

## Commands

```bash
vendor/bin/rector process app/ src/       # automated refactoring (review the diff!)
vendor/bin/pint                           # code style — config is canonical
vendor/bin/phpstan analyse                # static analysis (Larastan) — must pass
```

Composer scripts (recommended, so every agent and CI runs the same thing):

```json
"scripts": {
    "refactor": "rector process",
    "lint": "pint",
    "analyse": "phpstan analyse --memory-limit=1G",
    "check": [
        "@refactor",
        "@lint",
        "@analyse"
    ]
}
```

## Pint

Pint enforces formatting, imports, spacing, and modern PHP conventions. The `laravel` preset is the base; add the rules that mechanically enforce the playbook's file-header contract instead of relying on discipline:

```json
{
    "preset": "laravel",
    "rules": {
        "declare_strict_types": true,
        "final_class": true,
        "fully_qualified_strict_types": true,
        "ordered_imports": {
            "sort_algorithm": "alpha"
        }
    }
}
```

- `declare_strict_types` and `final_class` turn two core Musts into machine-enforced facts (see the labrodev-core skill for the rules themselves).
- `final_class` must not touch abstract base classes (it skips abstract classes by design) — `BaseModel` stays abstract and unfinalized.
- Run on all modified files before every PR; Pint's output is never "reformatted back".

## PHPStan (Larastan)

```neon
includes:
    - vendor/larastan/larastan/extension.neon

parameters:
    level: 7
    paths:
        - app
        - src
    scanFiles:
        - _ide_helper_models.php
```

- **Level 7 minimum** for new projects; never lower an existing project's level to make an error disappear.
- `scanFiles` pulls in the ide-helper mixin so PHPStan knows every model column without `@property` lists in the models → see the labrodev-model skill.
- Larastan resolves relation and cast types; the playbook's generics docblocks close the rest: `@return Builder<Model>` on Query methods (labrodev-query), `@extends QueryBuilder<Model>` on IndexQueries (labrodev-query), `@extends Collection<int, Model>` on Collections (labrodev-model), `@return BelongsTo<User, $this>` on relation methods (labrodev-model).
- A baseline is acceptable ONLY when adopting the toolchain on an existing codebase — and from that moment it only shrinks.

## Rector

Rector keeps the codebase modern and consistent: PHP-version upgrades, dead-code removal, and Laravel-specific refactors (via `driftingly/rector-laravel`). Installing the package without a `rector.php` means Rector never runs — configuration is part of adoption, not a later step.

```php
<?php

declare(strict_types=1);

use Rector\Config\RectorConfig;
use Rector\Php85\Rector\LevelSetList;
use RectorLaravel\Set\LaravelSetList;

return RectorConfig::configure()
    ->withPaths([
        __DIR__.'/app',
        __DIR__.'/src',
    ])
    ->withPhpSets()
    ->withSets([
        LaravelSetList::LARAVEL_CODE_QUALITY,
    ])
    ->withImportNames(removeUnusedImports: true);
```

- Always review Rector's diff before committing — it is a refactoring tool, not a formatter.
- Rector output goes through Pint before PHPStan (the run order above).
- A rule that conflicts with a playbook convention gets excluded explicitly in `rector.php` with the reason in the PR that excludes it.

## Relationship to architecture tests

Pint/PHPStan/Rector enforce style, types, and syntax. Playbook *structure* (no FormRequests, Core never imports App\Layer, controllers stay thin) is enforced by Pest architecture tests → see the labrodev-testing skill. Both gates run before every PR; neither substitutes for the other.

## Review checklist

1. Did Rector, then Pint, then PHPStan run on all modified code — and do all three pass?
2. Is the PHPStan level unchanged (or raised), with zero new baseline entries?
3. Is every suppression (`@phpstan-ignore`, rule exclusion) accompanied by a stated justification?
4. Was the ide-helper mixin regenerated after any schema change, before the PHPStan run?
5. Do `pint.json` rules still enforce `declare_strict_types` and `final_class`?
6. Does `rector.php` exist and cover both `app/` and `src/` (an installed-but-unconfigured Rector runs nothing)?
7. Were tool configuration changes made deliberately and reviewed as architecture, not slipped in to silence an error?
