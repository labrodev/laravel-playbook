# Labrodev Infrastructure

Always-on law. `Core/Infrastructure/{IntegrationName}/` holds everything that talks to the outside world — payment gateways, email/SMS/messenger providers, ERP/CRM and marketplace APIs, webhooks, file storage, third-party REST/GraphQL clients. Infrastructure is **technical, not business**: the adapter layer between the Domain and external systems. Anatomy and templates → the `labrodev-infrastructure` skill; per-file checks → `rules/infrastructure.md`.

## Musts

- **One module per integration**: `Core/Infrastructure/Paddle/`, `Core/Infrastructure/Postmark/`, `Core/Infrastructure/Slack/` — or one module per capability with vendor adapters inside (`Core/Infrastructure/Messaging/`) when several vendors serve the same purpose.
- **Domains reach Infrastructure only through contracts** (interfaces) living in the module's `Contracts/` subfolder. The Domain never imports a vendor SDK or a concrete adapter.
- Infrastructure contracts are the **sanctioned interface use case**: the contract, its implementation(s), and the provider wiring are all named explicitly — this satisfies the "no unplanned bindings" rule (→ labrodev-core): plan them, name them, wire them.
- **Adapters wrap the vendor SDK/HTTP client** and own authentication, request formatting, retries, and rate limits.
- **External payloads are mapped at the boundary** into internal, domain-friendly structures: plain `final readonly` DTOs with camelCase properties — NOT Spatie Data, which is the user-input boundary (→ labrodev-data).
- **Boundary DTOs are named with a `Payload` suffix** (`OutboundMessagePayload`, `CrmContactPayload`) — or an `Envelope` suffix when the class wraps a payload with transport metadata (headers, routing, signature). A bare noun (`OutboundMessage`) is not a boundary DTO name. (Distinct from the mutable pipeline `{Workflow}Payload` → labrodev-pipeline.)
- **The vendor's documented contract is the single mapping source**: each DTO field reads exactly one payload key, per the vendor docs or a captured real payload. Required data that is missing or malformed throws the module's named exception (→ labrodev-exception) at the mapper — the Domain never receives half-empty DTOs (→ labrodev-core contract commitment).
- When several vendors implement one capability, add a **Resolver that picks the adapter by a domain enum** — the caller never switches on vendor names.
- **Vendor credentials/config come from `config/services.php`** (env-backed) and are injected into the adapter — never read `env()` outside config files, never hardcode.

## Must-nots

- No business rules, business state decisions, or workflows in Infrastructure. The moment an adapter decides *whether* something should happen, that logic belongs in a Domain Service or Action — Infrastructure executes, the Domain decides.
- Infrastructure never accesses `App/Layer` and never depends on specific Domain business rules.
- No raw vendor payloads leaking into the Domain: arrays and SDK response objects stop at the adapter; Domains see typed DTOs.
- No hedged mapping: no alternative-key fallbacks (`$payload['mobile'] ?? $payload['phone']`), no `looksLike*()` shape detection, no blanket `stringOrNull()`-style coercion of documented fields. Uncertainty about a payload's shape is resolved by reading the vendor docs or capturing a real payload — never by guarding every imaginable variant.
- No inline HTTP calls from Domain code (Actions, Services, Pipeline steps) — always through the contract.
- No slow or unreliable external calls inside request-bound flows — dispatch a queued Job that calls the contract instead (→ labrodev-pipeline).
