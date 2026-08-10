---
paths:
  - "**/Core/Infrastructure/**"
---

# Infrastructure — playbook checks

- Universal: file opens with `declare(strict_types=1)` and a namespace matching its folder; the class is `final`; names are singular.
- Domain code imports only the `Contracts/` interface (or the resolver) — never a vendor SDK or a concrete adapter.
- Contract, implementation, and provider wiring are all explicitly named — no ad-hoc bindings.
- External payloads stop at the adapter; the Domain receives `final readonly` DTOs.
- The adapter is free of business decisions — it executes; the Domain decides.
- Credentials are injected from `config/services.php` — no `env()` outside config files, nothing hardcoded.
- Vendor selection is centralized in a resolver keyed by a domain enum — no vendor `match`/`if` chains in Domain code.
- Slow or unreliable external calls go through queued Jobs rather than blocking request-bound flows.
- Infrastructure stays out of `App/Layer` and free of Domain business rules.

Full anatomy and templates → labrodev-infrastructure skill.
