# Labrodev Playbook

## About

Architecture conventions, coding standards, and AI skills for Laravel projects. An opinionated way to organize and work with the latest Laravel 13+ using a Domain-driven architecture. This is not a framework, not a standard, and not the only way. It is the Labrodev way — shaped by experience, refined over time, and open to change.
Designed to be installed via Laravel Boost and used as a skill source for AI development agents — Cursor, Claude Code, Codex, and Junie.

---

## What's inside

The whole playbook lives under [`skills/labrodev-playbook/`](skills/labrodev-playbook/) so Laravel Boost installs it as a single, self-contained skill:

- **`SKILL.md`** — agent skill entry point
- **`index.md`** — human-readable table of contents
- **`docs/`** — 13 chapters covering philosophy, structure, naming, boundaries, data, models, testing, anti-patterns, immutability, tooling, stubs, Inertia+React, workflow, and review checklist
- **`stubs/`** — code templates used by AI tools as executable specifications
- **`recipes/`** — task-specific execution guides

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
php artisan boost:add-skill labrodev/laravel-playbook --skill=labrodev-playbook
```

The skill installs to `.ai/skills/labrodev-playbook/` and is then synced to your configured agents (Cursor, Claude Code, Junie) by `boost:update`, which Boost runs automatically.

### Direct install (Claude Code / Cursor without Boost)

```bash
git clone https://github.com/labrodev/laravel-playbook.git /tmp/laravel-playbook
cp -R /tmp/laravel-playbook/skills/labrodev-playbook ~/.claude/skills/labrodev-playbook
# or for Cursor:
cp -R /tmp/laravel-playbook/skills/labrodev-playbook ~/.cursor/skills/labrodev-playbook
```

---

## Updating

```bash
php artisan boost:add-skill labrodev/laravel-playbook --skill=labrodev-playbook --force
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

---

## Support
- [Buy me a Coffee](https://buymeacoffee.com/lshnpt.dev)

## License

MIT. See [LICENSE](LICENSE) for details.

