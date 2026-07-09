---
name: labrodev-infrastructure
description: "Use when integrating any external system in a Labrodev Laravel project — payment gateways, email/SMS/messenger providers, ERP or CRM APIs, webhooks, file storage, third-party SDKs — or when creating/reviewing anything under Core/Infrastructure: contracts, per-vendor adapters, resolvers, and external-data mapping."
license: MIT
metadata:
  author: labrodev
---

# Infrastructure: adapters between the Domain and the outside world

Part of the Labrodev playbook skill set — assumes labrodev-core and labrodev-naming are installed. If absent, minimum global rules: `declare(strict_types=1)`, final classes, App/Layer depends on Core (never the reverse), named-argument invocation.

`Core/Infrastructure/{IntegrationName}/` holds everything that talks to the outside world: payment gateways, email/SMS/messenger providers, ERP and CRM integrations, marketplace APIs, webhook handling, file storage, third-party REST/GraphQL clients. Infrastructure is **technical, not business** — it is the adapter layer between the Domain and external systems.

## Musts

- One module per integration: `Core/Infrastructure/Paddle/`, `Core/Infrastructure/Postmark/`, `Core/Infrastructure/Slack/` — or one per capability with vendor adapters inside (`Core/Infrastructure/Messaging/` when several vendors serve the same purpose).
- **Domains reach Infrastructure only through contracts** (interfaces) that live in the Infrastructure module's `Contracts/` subfolder. The Domain never imports a vendor SDK or a concrete adapter.
- Infrastructure contracts are the **sanctioned interface use case**: the contract, its implementation(s), and the provider wiring are all named explicitly (this satisfies labrodev-core's "no unplanned bindings" rule — plan them, name them, wire them).
- Adapters wrap the vendor SDK/HTTP client and own authentication, request formatting, retries, and rate limits.
- External payloads are mapped at the boundary into internal, domain-friendly structures: plain `final readonly` DTOs with camelCase properties (NOT Spatie Data — that is the user-input boundary → see the labrodev-data skill).
- When several vendors implement one capability, add a **Resolver** that picks the adapter by a domain enum — the caller never switches on vendor names.
- Vendor credentials/config come from `config/services.php` (env-backed) and are injected into the adapter — never read `env()` outside config files, never hardcode.

## Must-nots

- No business rules, no business state decisions, no workflows in Infrastructure. The moment an adapter decides *whether* something should happen, that logic belongs in a Domain Service or Action.
- Infrastructure must never access `App/Layer` and must not depend on specific Domain business rules.
- No raw vendor payloads leaking into the Domain: arrays and SDK response objects stop at the adapter; Domains see typed DTOs.
- No inline HTTP calls from Domain code (Actions, Services, Pipeline steps) — always through the contract.
- Slow or unreliable external calls inside request-bound flows: dispatch a queued Job that calls the contract instead of calling synchronously (→ see the labrodev-pipeline skill for step ordering).

## The pattern: contract → adapters → resolver

Worked example — one messaging capability, several vendors:

```
Core/Infrastructure/Messaging/
├── Contracts/
│   └── MessageNotifier.php          the capability contract
├── SlackNotifier.php                per-vendor adapter
├── TelegramNotifier.php             per-vendor adapter
├── MessageNotifierResolver.php      picks the adapter by domain enum
└── OutboundMessage.php              boundary DTO (final readonly)
```

### Contract

```php
<?php

declare(strict_types=1);

namespace Core\Infrastructure\Messaging\Contracts;

use Core\Infrastructure\Messaging\OutboundMessage;

interface MessageNotifier
{
    public function send(OutboundMessage $outboundMessage): void;
}
```

### Boundary DTO

```php
<?php

declare(strict_types=1);

namespace Core\Infrastructure\Messaging;

final readonly class OutboundMessage
{
    public function __construct(
        public string $recipient,
        public string $subject,
        public string $body,
    ) {}
}
```

### Adapter (one per vendor)

```php
<?php

declare(strict_types=1);

namespace Core\Infrastructure\Messaging;

use Core\Infrastructure\Messaging\Contracts\MessageNotifier;
use Illuminate\Support\Facades\Http;

final readonly class SlackNotifier implements MessageNotifier
{
    public function __construct(
        private string $webhookUrl,
    ) {}

    public function send(OutboundMessage $outboundMessage): void
    {
        Http::asJson()
            ->post($this->webhookUrl, [
                'text' => sprintf('%s — %s', $outboundMessage->subject, $outboundMessage->body),
            ])
            ->throw();
    }
}
```

### Resolver (when the vendor is chosen at runtime)

```php
<?php

declare(strict_types=1);

namespace Core\Infrastructure\Messaging;

use Core\Domain\Channel\Enums\ChannelType;
use Core\Infrastructure\Messaging\Contracts\MessageNotifier;

final readonly class MessageNotifierResolver
{
    public function __invoke(ChannelType $channelType): MessageNotifier
    {
        return match ($channelType) {
            ChannelType::Slack => resolve(SlackNotifier::class),
            ChannelType::Telegram => resolve(TelegramNotifier::class),
        };
    }
}
```

### Wiring (deliberate, in a provider)

```php
$this->app->when(SlackNotifier::class)
    ->needs('$webhookUrl')
    ->giveConfig('services.slack.webhook_url');
```

### Consuming from the Domain

A Pipeline step, Service, or Job injects the contract (single vendor) or the resolver (runtime vendor) — never a concrete adapter:

```php
final readonly class PushBookingToCrm
{
    public function __construct(
        private CrmClient $crmClient, // Contracts\CrmClient
    ) {}
}
```

## Placement decision

| Situation | Home |
|---|---|
| Talking to an external system (transport, auth, mapping) | `Core/Infrastructure/{Integration}` |
| Deciding whether/when to talk to it | Domain Action/Service/Rule |
| Multi-step flow that includes external calls | Pipeline with an Infrastructure-calling step → see the labrodev-pipeline skill |
| Async/retryable external work | Queued Job delegating to the contract |
| Generic technical helper with no external system | `Core/Support` → see the labrodev-core skill |

## Review checklist

1. Does the Domain import only the `Contracts/` interface (and the resolver), never a vendor SDK or concrete adapter?
2. Are contract, implementation, and provider wiring all explicitly named (no ad-hoc bindings)?
3. Do external payloads stop at the adapter, with the Domain receiving `final readonly` DTOs?
4. Is the adapter free of business decisions (it executes; the Domain decides)?
5. Are credentials injected from `config/services.php` — no `env()` outside config, nothing hardcoded?
6. Is vendor selection centralized in a resolver keyed by a domain enum (no vendor `match`/`if` chains in Domain code)?
7. Are slow/unreliable calls pushed to queued Jobs rather than blocking request-bound flows?
8. Does Infrastructure stay out of `App/Layer` and free of Domain business rules?
