---
name: labrodev-static-analysis
description: "Use when configuring, running, or fixing findings from the code-quality toolchain in a Labrodev Laravel project — Pint (pint.json, code style), PHPStan/Larastan (phpstan.neon, levels, baselines, generics errors), or Rector (rector.php, automated refactoring) — or when deciding whether a tool error may be suppressed."
license: MIT
metadata:
  author: labrodev
---

# Static Analysis: Pint, PHPStan, Rector

Part of the Labrodev playbook. **The law for this component lives in the always-on `labrodev-static-analysis` guideline** (musts, must-nots); the per-file checklist is `rules/static-analysis.md`. This skill holds the craft: configs, run order, and troubleshooting.

## Commands

```bash
vendor/bin/rector process app/ src/       # automated refactoring (review the diff!)
vendor/bin/pint                           # code style — config is canonical
vendor/bin/phpstan analyse --memory-limit=2G   # static analysis (Larastan) — must pass
```

When the local PHP version differs from the project's required version, run the tools through the project runtime (Sail): `vendor/bin/sail bin phpstan analyse --memory-limit=2G` — never against a mismatched local PHP.

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
    "notPath": [
        "_ide_helper_models.php"
    ],
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

The generated ide-helper mixin file stays committed (CI needs it for PHPStan) and is excluded from Pint via `notPath`.

- `declare_strict_types` and `final_class` turn two core Musts into machine-enforced facts (see the labrodev-core skill for the rules themselves).
- `final_class` must not touch abstract base classes (it skips abstract classes by design) — `BaseModel` stays abstract and unfinalized.

## PHPStan (Larastan)

```neon
includes:
    - vendor/larastan/larastan/extension.neon
    - vendor/nesbot/carbon/extension.neon

parameters:
    level: 6
    paths:
        - app
        - routes
        - tests
        - src
    scanFiles:
        - _ide_helper_models.php
```

- **Level 6 is the default**; the level may vary per project — check `phpstan.neon`.
- The Carbon extension (`nesbot/carbon`) closes date-handling false positives; include it alongside Larastan.
- `scanFiles` pulls in the ide-helper mixin so PHPStan knows every model column without `@property` lists in the models → see the labrodev-model skill. Regenerate the mixin after any schema change, BEFORE analysing.
- Larastan resolves relation and cast types; the playbook's generics docblocks close the rest: `@return Builder<Model>` on Query methods (labrodev-query), `@extends QueryBuilder<Model>` on IndexQueries (labrodev-query), `@extends Collection<int, Model>` on Collections (labrodev-model), `@return BelongsTo<User, $this>` on relation methods (labrodev-model).

### Recurring trap: pre-save null guards vs. docblock non-null

A generated `@property string $uuid` (NOT NULL column) makes PHPStan flag `=== null` checks inside `creating()`/`saving()` hooks as dead code — but before the first save, the in-memory model genuinely has no value there. The docblock describes a **persisted row**, not a pre-save instance.

- **Wrong fix**: deleting the null-guard because PHPStan calls it dead — this has broken real test suites (NOT NULL violations on insert).
- **Right fix**: read via `$model->getAttribute('uuid')` inside creating/saving hooks — it returns `mixed`, so the guard stays live AND PHPStan-clean.
- Related nuance when "simplifying": `$a->b ?? $c` is null-safe only for the **left** arm (`??` uses isset semantics) — the right arm still needs `?->` if it can be null. Never strip both operators at once.

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
- Rector output goes through Pint before PHPStan (the run order in Commands above).

## Relationship to architecture tests

Pint/PHPStan/Rector enforce style, types, and syntax. Playbook *structure* (no FormRequests, Core never imports App\Layer, controllers stay thin) is enforced by Pest architecture tests → see the labrodev-testing skill. Both gates run before every PR; neither substitutes for the other.
