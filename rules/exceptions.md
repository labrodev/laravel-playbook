---
paths:
  - "**/Exceptions/**"
---

# Exceptions — playbook checks

- Universal: file opens with `declare(strict_types=1)` and a namespace matching its folder; the class is `final`; names are singular.
- Constructor is `private` (or deliberately `protected`); the only other public member is a single static `make()` factory.
- Every throw site calls `::make()` — with named arguments when there is more than one — never `new SomeException(...)` or a bare SPL exception.
- Placement follows origin — Domain, Shared (domain-noun-free), Infrastructure `{Integration}`, Feature `{Feature}`, or Layer/App — per the guideline's placement table.
- Name follows `{DescriptiveCondition}Exception`.
- Infrastructure exceptions wrap the vendor SDK exception (passing `previous`) before it crosses into the Domain.
- No business decision logic in the exception class — it formats and carries context; the caller decides whether to throw.
- No silent empty `catch {}` blocks.

Full anatomy and templates → labrodev-exception skill.
