---
name: labrodev-infrastructure
description: "Use when integrating any external system in a Labrodev Laravel project — payment gateways, email/SMS/messenger providers, ERP or CRM APIs, webhooks, file storage, third-party SDKs — or when creating/reviewing anything under Core/Infrastructure: contracts, per-vendor adapters, resolvers, and external-data mapping."
license: MIT
metadata:
  author: labrodev
---

# Infrastructure: adapters between the Domain and the outside world

Part of the Labrodev playbook. **The law for this component lives in the always-on `labrodev-infrastructure` guideline** (musts, must-nots); the per-file checklist is `rules/infrastructure.md`. This skill holds the craft: anatomy, canonical templates, and edge cases.

`Core/Infrastructure/{IntegrationName}/` holds everything that talks to the outside world: payment gateways, email/SMS/messenger providers, ERP and CRM integrations, marketplace APIs, webhook handling, file storage, third-party REST/GraphQL clients. Infrastructure is **technical, not business** — it is the adapter layer between the Domain and external systems.

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
