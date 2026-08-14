# Labrodev Playbook

## About

Architecture conventions, coding standards, and AI context for Laravel projects. An opinionated way to organize and work with the latest Laravel 13+ using a Domain-driven architecture. This is not a framework, not a standard, and not the only way. It is the Labrodev way — shaped by experience, refined over time, and open to change.
Designed for AI development agents — Cursor, Claude Code, Codex, and Junie — and aligned with [Laravel Boost](https://laravel.com/docs/13.x/boost)'s three context tiers.

---

## The three tiers

The playbook ships as **guidelines, skills, and rules** — the same split Laravel Boost uses. Each statement of the playbook has exactly one home:

| Tier | Owns | Loaded | Where it lives |
|---|---|---|---|
| [`resources/boost/guidelines/`](resources/boost/guidelines/) | **The law** — musts and must-nots per component | Upfront, always in context | composed into your agents' context by `boost:install` |
| [`resources/boost/skills/`](resources/boost/skills/) | **The craft** — anatomy, canonical templates, worked examples | On-demand, when building that component | installed to `.ai/skills/` by `boost:install` |
| [`resources/boost/rules/`](resources/boost/rules/) | **The check** — per-file checklists bound to path globs | When a matching file is edited | app's `.ai/rules/` (one-time copy, see below) |

- A guideline tells the agent the component's law before it writes a line.
- A skill shows how to build the component well, loaded only when relevant.
- A rule fires exactly when a matching file is touched — the review checklist at the point of edit (`.ai/rules/index.md` maps globs to rule files).

**Install the triple per component.** Skills are not standalone: a skill assumes its guideline is in context — installing a skill without its guideline loses the law.

## Components

| Component | Guideline | Skill | Rule |
|---|---|---|---|
| **Core — structure & boundaries** (foundation) | `labrodev-core.md` | `labrodev-core` | — (always-on; universal checks folded into every rule) |
| **Naming & named arguments** (foundation) | `labrodev-naming.md` | `labrodev-naming` | — |
| Controllers & routes | `labrodev-controller.md` | `labrodev-controller` | `controllers.md` |
| ViewModels & Resources | `labrodev-viewmodel-resource.md` | `labrodev-viewmodel-resource` | `viewmodels-resources.md` |
| Queries & IndexQueries | `labrodev-query.md` | `labrodev-query` | `queries.md` |
| Data classes | `labrodev-data.md` | `labrodev-data` | `data.md` |
| Models & migrations | `labrodev-model.md` | `labrodev-model` | `models.md`, `migrations.md` |
| Actions, Services & Rules | `labrodev-action.md` | `labrodev-action` | `actions.md` |
| Pipelines | `labrodev-pipeline.md` | `labrodev-pipeline` | `pipelines.md` |
| Authorization | `labrodev-authorization.md` | `labrodev-authorization` | `policies.md` |
| Enums | `labrodev-enum.md` | `labrodev-enum` | `enums.md` |
| Infrastructure | `labrodev-infrastructure.md` | `labrodev-infrastructure` | `infrastructure.md` |
| Exceptions | `labrodev-exception.md` | `labrodev-exception` | `exceptions.md` |
| Testing | `labrodev-testing.md` | `labrodev-testing` | `tests.md` |
| Static analysis | `labrodev-static-analysis.md` | `labrodev-static-analysis` | `static-analysis.md` |
| Inertia + React | `labrodev-inertia-react.md` | `labrodev-inertia-react` | `frontend.md` |

The two **foundation** components (`labrodev-core`, `labrodev-naming`) must always be installed — every other component assumes them. Backend-only project? Skip `labrodev-inertia-react` and `frontend.md`.

---

## Requirements

- PHP **^8.5**
- Laravel **^13.0**

---

## Installation

### Via Laravel Boost (recommended)

The playbook is a Composer package shipping its guidelines and skills in `resources/boost/`, which Boost auto-discovers. Install both, then let Boost's installer pick the playbook up:

```bash
composer require laravel/boost --dev
composer require labrodev/laravel-playbook --dev
php artisan boost:install
```

`boost:install` lists `labrodev/laravel-playbook (guidelines, skills)` under third-party guidelines/skills — select it, and the guidelines are composed into your agents' always-on context (`CLAUDE.md`, `AGENTS.md`, ...) while the skills install to `.ai/skills/`.

Boost has no third-party mechanism for **rules**, so copy those once:

```bash
cp vendor/labrodev/laravel-playbook/resources/boost/rules/*.md .ai/rules/
```

Cherry-picking? Always keep `labrodev-core` and `labrodev-naming` guidelines + skills, and install the guideline+skill+rule triple for every component you take.

Your own app-specific conventions do **not** go into these files — record them with Boost's `record-rule` tool into `.ai/rules/` alongside the playbook's rules. Where an app rule conflicts with a playbook default, the app rule wins.

### Direct install (Claude Code / Cursor without Boost)

```bash
git clone https://github.com/labrodev/laravel-playbook.git /tmp/laravel-playbook
cp -R /tmp/laravel-playbook/resources/boost/skills/labrodev-* ~/.claude/skills/
# or for Cursor:
cp -R /tmp/laravel-playbook/resources/boost/skills/labrodev-* ~/.cursor/skills/
```

Without Boost there is no guideline/rule loader — paste the contents of `resources/boost/guidelines/*.md` for your installed components into your project's always-on context file (`CLAUDE.md`, `AGENTS.md`, or Cursor rules) so the law is present in every session, and add one line instructing the agent to consult `.ai/rules/index.md` before editing any file if you also copy the rules.

---

## Updating

```bash
composer update labrodev/laravel-playbook
php artisan boost:update
```

Re-copy `resources/boost/rules/*.md` to `.ai/rules/` if rule files changed. Direct install: re-run the `git clone` + `cp` commands above.

---

## Contributing

1. Fork the repository
2. Create a feature branch
3. Keep the tiers consistent: a **law** change touches the guideline *and* its rule file; a **template** change touches only the skill. Never restate law in a skill.
4. Submit a pull request

---

## Credits

- [Petro Lashyn](https://github.com/labrodev)

---

## Support

- [Buy me a Coffee](https://buymeacoffee.com/lshnpt.dev)

## License

MIT. See [LICENSE](LICENSE) for details.
