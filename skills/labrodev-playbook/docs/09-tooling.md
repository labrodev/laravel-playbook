# Tooling

This project uses automated tooling to enforce code quality, consistency, and long-term maintainability.

These tools are not optional. They are part of the architecture.

---

## Pint (Code style)

Pint is used to enforce consistent PHP code style.

Rules:
- Pint must be run on all modified files before opening a PR.
- Code style issues must not be fixed manually if Pint can handle them.
- Pint configuration is treated as canonical.

Pint enforces:
- formatting
- imports
- spacing
- modern PHP conventions

---

## PHPStan (Static analysis)

PHPStan is used to detect type errors, dead code, and unsafe behavior.

Rules:
- PHPStan must pass for all modified code.
- Type safety is preferred over convenience.
- Suppressing errors requires strong justification.

PHPStan enforces:
- correct types
- safer refactoring
- fewer runtime surprises

---

## Rector (Automated refactoring)

Rector is used to keep the codebase modern and consistent.

Rules:
- Rector must be run on modified files when applicable.
- Rector suggestions should not be ignored without reason.
- Rector helps enforce architectural and language-level rules.

Rector enforces:
- modern PHP syntax
- removal of outdated PHP patterns and disallowed structures (see `docs/07-anti-patterns.md`)
- consistent refactoring rules

---

## Tooling philosophy

These tools exist to:
- reduce review noise
- prevent trivial mistakes
- enforce consistency automatically
- free humans to review architecture and behavior

If a change requires fighting the tools, the design should be reconsidered.