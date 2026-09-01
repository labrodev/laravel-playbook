---
paths:
  - "**/Core/Domain/**/Pipelines/**"
  - "**/Payloads/**"
  - "**/Core/Feature/**"
---

# Pipelines & Payloads — playbook checks

- Universal: file opens with `declare(strict_types=1)` and a namespace matching its folder; the class is `final`; names are singular; the class body contains no comments — typed PHPStan annotations only.
- Confirm the use case is genuinely a staged workflow (3+ distinct side-effecting steps) — an atomic mutation belongs in a plain Action.
- The trio lives in the right place: single-domain → `Core/Domain/{Domain}`, cross-domain → `Core/Feature/{FeatureName}`.
- The orchestrator is a Service in `Services/` with a single `__invoke()`, invoked as a callable with named arguments.
- The Service validates the pipeline result with `instanceof` + `PipelinePayloadIncorrect::make(...)`.
- Every step does exactly one thing, mutates the Payload, and returns `$next($payload)`.
- Steps are verb-first with no suffix, housed under `Pipelines/{Workflow}/`.
- Mutating steps delegate to Actions (own transactions); persistence steps run before mail/CRM/external steps.
- External integrations go through Infrastructure contracts; slow/unreliable calls are pushed to queued Jobs.
- The Payload is a mutable `final class` dedicated to this one workflow, with a `make()` constructor.

Full anatomy and templates → labrodev-pipeline skill.
