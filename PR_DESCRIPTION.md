# Split the playbook into 15 atomic skills

## What changed

The monolithic `labrodev-playbook` skill is replaced by **15 atomic skills**, one per architecture component or aspect, each self-contained (rules + embedded canonical templates + review checklist) and installable individually or together via Laravel Boost (`boost:add-skill labrodev/laravel-playbook --all`).

**Foundations:** `labrodev-core` (structure, boundaries, golden paths, immutability, philosophy: atomic classes / puzzle pieces / vertical slices / single source of truth), `labrodev-naming` (all naming rules + the named-arguments invocation contract).

**Structural:** `labrodev-controller`, `labrodev-viewmodel-resource`, `labrodev-query`, `labrodev-data`, `labrodev-model`, `labrodev-action`, `labrodev-pipeline`, `labrodev-authorization`, `labrodev-enum`, `labrodev-infrastructure`.

**Process:** `labrodev-testing` (incl. Pest architecture tests), `labrodev-static-analysis` (Pint/PHPStan/Rector), `labrodev-inertia-react`.

## Architecture rulings encoded along the way

- One shared `{Model}Data` serves both create and update; split only when fields genuinely differ
- Controllers are single-action invokable classes — no multi-method controllers, no exceptions
- Clean models: no `@property` lists, no comments; ide-helper mixin mode (`-M`) + relation `@return` generics only
- Orchestrator class type retired: staged workflows = Payload + `Pipelines/{Workflow}/` steps + a Service playing the orchestrator role (cross-domain → `Core/Feature`)
- Full enum contract: `label()`, `Rule::enum`, model `casts()`, backend `value` + `*_label` emission via EnumMapper
- PHPStan: level 6 default, root-cause fixes only (no baseline, no ignores), pre-save `getAttribute()` guard pattern
- Infrastructure: contract → per-vendor adapter → resolver-by-enum pattern
- Stubs repaired before the split (policy constant contract, `final readonly` order, `#[UsePolicy]` wiring, Spatie API fixes, uuid route binding, toast + `to_route()` idiom, `prepareForPipeline()`)

## Removed

- The `labrodev-playbook` monolith (docs/stubs/recipes) — the 15 SKILL.md files are now the single source of truth
- The `labrodev-workflow` skill (by decision)

Rule ownership is strict: every rule lives in exactly one skill; others carry one-line pointers. Verified by multi-pass review: zero dangling cross-references, zero placeholder tokens in embedded templates, no project-specific (tenant/vendor) vocabulary.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
