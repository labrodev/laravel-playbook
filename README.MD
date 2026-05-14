# Labrodev Playbook

> Architecture conventions, coding standards, and AI skills forLaravel projects.

A opinionated way to organizee and work with latest Laravel using a Domain-driven architecture. This is not a framework, not a standard, and not the only way. It is the Labrodev way — shaped by experience, refined over time, and open to change.

---

## What's inside

- **`docs/`** — 13 chapters covering philosophy, structure, naming, boundaries, data, models, testing, anti-patterns, immutability, tooling, stubs, Inertia+React, workflow, and review checklist
- **`stubs/`** — code templates used by AI tools as executable specifications
- **`recipes/`** — task-specific execution guides
- **`index.md`** — Laravel Boost skill entry point
- **`SKILL.md`** — Claude Code skill entry point

---

## Requirements

- PHP **^8.5**
- Laravel **^13.0**

---

## Installation

### Via Laravel Boost

```bash
composer require laravel/boost
php artisan boost:install
php artisan boost:add-skill labrodev/playbook
```

### Via Composer

```bash
composer require labrodev/playbook --dev
```
---

## Updating

```bash
composer update labrodev/playbook
php artisan boost:update
```

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

