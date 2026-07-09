# Labrodev Playbook

## About

Architecture conventions, coding standards, and AI skills for Laravel projects. An opinionated way to organize and work with the latest Laravel 13+ using a Domain-driven architecture. This is not a framework, not a standard, and not the only way. It is the Labrodev way — shaped by experience, refined over time, and open to change.
Designed to be installed via Laravel Boost and used as a skill source for AI development agents — Cursor, Claude Code, Codex, and Junie.

---

## What's inside

The playbook ships as **15 atomic skills** under [`skills/`](skills/) — each one self-contained, covering one architecture component or aspect, installable individually or all at once via Laravel Boost:

**Foundation (install always):**

| Skill | Covers |
|---|---|
| `labrodev-core` | Project structure, dependency direction, golden paths, file header contract, immutability, anti-patterns |
| `labrodev-naming` | All naming rules + the named-arguments invocation contract |

**Structural (one per component):**

| Skill | Covers |
|---|---|
| `labrodev-controller` | Invokable controllers, JsonControllers, routes, `{model:uuid}` binding |
| `labrodev-viewmodel-resource` | ViewModels (Spatie) + Resources (the presentation boundary) |
| `labrodev-query` | Core Query classes + Layer IndexQueries (read side) |
| `labrodev-data` | Spatie Data validation layer + UUID casters (no FormRequests) |
| `labrodev-model` | Models, `casts()`, relations, Collections, Observers, migrations |
| `labrodev-action` | Actions, Services, domain Rules (write side) |
| `labrodev-pipeline` | Staged workflows: Pipeline steps, Payloads, and the orchestrating Service |
| `labrodev-authorization` | Policies, `#[UsePolicy]`, `#[Authorize]`, permission constants |
| `labrodev-enum` | The full enum contract: `label()`, `Rule::enum`, casts, frontend emission |
| `labrodev-infrastructure` | External integrations: contracts, per-vendor adapters, resolvers |

**Process:**

| Skill | Covers |
|---|---|
| `labrodev-testing` | Pest strategy + architecture tests that enforce the playbook |
| `labrodev-static-analysis` | Pint, PHPStan/Larastan, and Rector: configs, run order, suppression policy |
| `labrodev-inertia-react` | React/Inertia pages, props, forms, i18n |

Every skill is self-contained: rules, canonical code templates, and a review checklist in one `SKILL.md`, with one-line pointers between skills.

---

## Requirements

- PHP **^8.5**
- Laravel **^13.0**

---

## Installation

### Via Laravel Boost (recommended)

```bash
composer require laravel/boost
php artisan boost:install

# Everything (recommended):
php artisan boost:add-skill labrodev/laravel-playbook --all

# Or cherry-pick — always include the two foundation skills:
php artisan boost:add-skill labrodev/laravel-playbook \
  --skill=labrodev-core --skill=labrodev-naming \
  --skill=labrodev-controller --skill=labrodev-model

# List what's available:
php artisan boost:add-skill labrodev/laravel-playbook --list
```

Skills install to `.ai/skills/<skill-name>/` and are synced to your configured agents (Cursor, Claude Code, Junie) by `boost:update`, which Boost runs automatically.

> Backend-only project? Skip `labrodev-inertia-react`. But `labrodev-core` and `labrodev-naming` should always be installed — every other skill assumes them.

### Direct install (Claude Code / Cursor without Boost)

```bash
git clone https://github.com/labrodev/laravel-playbook.git /tmp/laravel-playbook
cp -R /tmp/laravel-playbook/skills/labrodev-* ~/.claude/skills/
# or for Cursor:
cp -R /tmp/laravel-playbook/skills/labrodev-* ~/.cursor/skills/
```

---

## Updating

```bash
php artisan boost:add-skill labrodev/laravel-playbook --all --force
```

Or, for direct installs, re-run the `git clone` + `cp -R` from the section above.

---

## Contributing

1. Fork the repository
2. Create a feature branch
3. Update docs and stubs together — never one without the other
4. Submit a pull request

---

## Credits

- [Petro Lashyn](https://github.com/labrodev)
- 
---

## Support
- [Buy me a Coffee][https://buymeacoffee.com/lshnpt.dev]

## License

MIT. See [LICENSE](LICENSE) for details.

