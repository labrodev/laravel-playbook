---
name: labrodev-infrastructure
description: "Use when integrating any external system in a Labrodev Laravel project — payment gateways, email/SMS/messenger providers, ERP or CRM APIs, webhooks, file storage, third-party SDKs — or when creating/reviewing anything under Core/Infrastructure: contracts, per-vendor adapters, resolvers, and external-data mapping."
license: MIT
metadata:
  author: labrodev
---

# Infrastructure: adapters between the Domain and the outside world

Part of the Labrodev playbook. **The law for this component lives in the always-on `labrodev-infrastructure` guideline** (musts, must-nots); the per-file checklist is `rules/infrastructure.md`. This skill holds the craft: anatomy, canonical templates, and edge cases.

Four pieces make up the pattern: a **contract** the Domain depends on, one **adapter** per vendor implementing it, a **resolver** for when the vendor is chosen at runtime, and a **boundary DTO** mapping vendor payloads into Domain-friendly shapes. Splitting it this way means adding or swapping a vendor never touches Domain code.

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

## Inbound payload mapping (fail loud)

Inbound vendor data is mapped by a static factory on the DTO (or a dedicated `{Thing}Mapper` when the mapping is large). The mapping commits to the vendor's documented contract — one payload key per field, required fields throw, nullable only where the docs say optional (→ labrodev-core contract commitment):

```php
<?php

declare(strict_types=1);

namespace Core\Infrastructure\Crm;

use Core\Infrastructure\Crm\Exceptions\CrmContactPayloadException;

final readonly class CrmContact
{
    public function __construct(
        public int $externalId,
        public string $email,
        public ?string $phone, // optional per CRM v3 docs
    ) {}

    /** Contract: CRM v3 API — GET /contacts/{id} */
    public static function fromPayload(array $payload): self
    {
        $externalId = $payload['id'] ?? null;
        $email = $payload['email'] ?? null;
        $phone = $payload['phone'] ?? null;

        if (! is_int($externalId) || ! is_string($email) || ($phone !== null && ! is_string($phone))) {
            throw CrmContactPayloadException::make(payload: $payload);
        }

        return new self(
            externalId: $externalId,
            email: $email,
            phone: $phone,
        );
    }
}
```

`?? null` here is isset-safe reading of the **one** documented key, immediately followed by a throw — not a fallback. The exception lives in `Core/Infrastructure/{Integration}/Exceptions` and follows the `make()` convention (→ labrodev-exception skill).

The anti-pattern this exists to prevent — hedged mapping that guards against imagined shape variants:

```php
// ❌ contract-blind: guesses keys, coerces everything to null
$firstname = $this->stringOrNull($client['first_name'] ?? $client['firstname'] ?? null);
$phone = $this->stringOrNull($client['mobile'] ?? $client['phone'] ?? null);
```

When the vendor renames a key, the hedged version silently writes `null` into persisted data; the strict mapper throws at the boundary, where the bug is visible and attributable. If the payload shape is genuinely unknown, capture a real payload or read the vendor docs before writing the mapper — the fallback chain is never the answer.

## Placement decision

| Situation | Home |
|---|---|
| Talking to an external system (transport, auth, mapping) | `Core/Infrastructure/{Integration}` |
| Deciding whether/when to talk to it | Domain Action/Service/Rule |
| Multi-step flow that includes external calls | Pipeline with an Infrastructure-calling step → see the labrodev-pipeline skill |
| Async/retryable external work | Queued Job delegating to the contract |
| Generic technical helper with no external system | `Core/Support` → see the labrodev-core skill |
