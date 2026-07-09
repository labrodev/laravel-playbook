# Naming

This document defines naming conventions used at Labrodev. Naming is treated as architecture: consistent naming reduces cognitive load, improves discoverability, and makes AI-assisted generation more reliable.

The rules below are defaults. If a domain has strong ubiquitous language that conflicts with a rule, the domain language wins, but the decision must be applied consistently.

---

## 1) General rules

All names are singular.

Examples:
- ProductController (not ProductsController)
- ProductCollection (not ProductsCollection)
- Order (not Orders)
- OrderPolicy (not OrdersPolicy)

Domain names are singular as well:
- Core/Domain/Order
- App/Layer/Dashboard/Order

Names must be distinctive and self-explanatory. Avoid generic terms such as:
- Manager
- Handler
- Processor
- Helper (except in Core/Support/Helpers where it is explicit)
- Util (prefer a real name)

---

## 2) Domain folder and namespace naming

Domain folders use singular business nouns:
- Core/Domain/Product
- Core/Domain/Order
- Core/Domain/Invoice

Layer folders mirror domains and are singular:
- App/Layer/Dashboard/Product
- App/Layer/Dashboard/Order

---

## 3) Actions

Actions represent business use cases and are named in the format:

Entity + VerbInBaseForm

The entity comes first.

Examples:
- ProductCreate (not CreateProduct)
- ProductUpdate
- ProductDelete
- OrderCancel
- InvoiceGenerate

Actions must not be named with generic verbs such as:
- Handle
- Process
- ExecuteAction

Actions must expose a single public entry point: `__invoke()`.

Rule:
- Use `__invoke()` for Actions (invokable single-responsibility objects).
- Do not use `handle()` for Actions.
- Do not add other public entry methods on Actions.

Example:

- `$productCreate(productData: $productData)` returning `Product` (PHP invokes `__invoke` on the action instance)

---

## 4) Jobs

Jobs represent asynchronous work and must start with a verb that describes what the job does:

Verb + Object + Job

Examples:
- SendEmailToCustomerJob
- RecalculatePriceJob
- SyncProductToCrmJob
- GenerateInvoicePdfJob

Jobs should be thin wrappers. They delegate real business behavior to Core Actions (invoked as callables) and should not contain domain logic.

---

## 5) Services

Services represent reusable domain-level behavior and must end with a self-explanatory verb-related name:

Noun + VerbAgent

Examples:
- EmailSender
- UserNotificator
- PriceCalculator
- TokenGenerator
- PdfRenderer

Avoid vague names:
- Service
- Manager
- Processor
- Helper

A service name must communicate the responsibility without reading the code.

For **single-operation** services (one clear operation), prefer a single public `__invoke()` as the entry point, consistent with Actions. Multi-step or multi-role services may use explicit named methods instead.

---

## 6) Prefix rules for domain components

In most cases, classes should be prefixed with the domain entity name to make code searchable and prevent collisions across domains.

This rule applies to everything except Models and Actions.

Preferred examples:
- ProductRule
- ProductObserver
- ProductUuidCast
- ProductCollection
- ProductData
- ProductPolicy
- ProductAddedEvent
- ProductEvaluationService

Exceptions:
- Models are named as the entity itself: Product, Order, Invoice
- Actions follow the dedicated Action naming rule: ProductCreate, OrderCancel

Data classes — one shared `{Model}Data` is the default:
- A single `ProductData` serves BOTH the create and the update Action.
- Split into `ProductCreateData` / `ProductUpdateData` ONLY when the two
  operations genuinely accept different fields — never pre-emptively.

---

## 7) Enums

Enums are named as concept nouns without the "Enum" suffix or prefix.

Examples:
- ProductStatus
- OrderState
- InvoiceType

Do not use:
- ProductStatusEnum
- EnumProductStatus

Enum cases should follow the project’s standard style (choose one and keep it consistent). If no style is defined yet, prefer PascalCase for readability.

---

## 8) Variables and attributes

Variables and attributes must be distinctive, self-explanatory, and avoid abbreviations unless the abbreviation is universally understood in the codebase.

General rule for variables:
- use camelCase for local variables and most properties
- prefer explicit semantic names over short placeholders (`$currentDate`, `$bookingStartTimestamp`) and avoid vague or abbreviated forms (`$cursor`, `$bStart`, `$bEnd`, `$prevTs`, `$nextTs`)

Example:
- $customerEmail
- $totalPrice
- $availableStock

Important exception for identifiers:
- model attributes and Data class attributes use snake_case

Examples:
- $product_id (not $productId)
- $order_id
- $customer_id

This exception exists because these values map directly to database columns and external representations.

### Typed value-object parameters (envelopes, DTOs, named domain types)

For **parameters** and **closure parameters** whose type is a **concrete named class** (e.g. `EmployeeEnvelope`, `BookingEnvelope`, `GapEnvelope`, `*Data`), the variable name MUST be the **class short name in camelCase**. Do not shorten to a generic domain noun that duplicates another meaning in scope.

Examples:

- `EmployeeEnvelope $employeeEnvelope` — not `$employee`
- `BookingEnvelope $bookingEnvelope` — not `$booking` or `$b`
- `GapEnvelope $gapEnvelope` — not `$gap` (when the declared type is `GapEnvelope`)

This keeps names aligned with the type, avoids ambiguity next to IDs or models, and makes code easier to search and review.

When **two parameters** share the same class type (e.g. sort comparators), disambiguate with clear prefixes: `$firstEmployeeEnvelope` / `$secondEmployeeEnvelope`, not `$a` / `$b`.

### Local variables holding a `*Data` / envelope instance

The same rule applies to **local variables** that hold an instance of a concrete `*Data` class (or other named DTO / envelope type): the name MUST be the **class short name in camelCase**. Do not replace it with abbreviated aliases.

Examples:

- `CustomerAddressData $customerAddressData = $payload->customerAddressData` — not `$addr`, `$a`, or `$addressData` when the declared type is `CustomerAddressData`
- `MissionData $missionData = …` — not `$mission` or `$m`

This applies whether the value comes from a method argument, a payload property, or a factory; the variable name should reflect the type, not a shortened nickname.

---

## 9) Call-site naming and arguments

Method and function calls should be as explicit as declarations.

Rules:
- when calling a function or method with **more than one argument**, use **named arguments**
- keep named arguments in a consistent order for readability; if the project has no stronger local convention, sort them alphabetically
- single-argument calls may use positional arguments

Examples:
- `$draftCreate(draftData: $draftData, supplier: $supplier);`
- `$draftUpdate(draft: $draft, draftData: $draftData);`
- `$draftDelete($draft);`

Avoid:
- `$draftCreate($supplier, $draftData);`

This reduces parameter-order mistakes and keeps call sites self-documenting.

---

## 10) Method naming

Methods must be verbal and self-explanatory.

Good method names:
- create()
- createProduct()
- fetchProduct()
- update()
- updateProductQuantity()
- remove()
- calculatePrice()
- fetchCategories()

Avoid method names that hide intent:
- handle()
- process()
- do()
- run()
- make()

Method names should reflect what they do, not how they do it.

---

## 11) Consistency and collisions

If two domains have the same concept name, collisions must be avoided by scoping and explicit prefixes.

Example:
- ProductAddedEvent in Core/Domain/Product/Events
- CustomerAddedEvent in Core/Domain/Customer/Events

Prefer explicit naming over short naming. Short names save typing but cost comprehension.

---

## 12) Summary checklist

- Everything is singular (folders, domains, classes)
- Actions: EntityFirst, single public entry `__invoke()` (not `handle()`)
- Jobs: VerbFirst and end with Job
- Services: end with an agent-like verb name (Sender, Calculator, Generator, Notificator)
- Most domain components: prefix with entity name (ProductRule, ProductPolicy, ProductUuidCast, etc.)
- Enums: no Enum suffix/prefix (ProductStatus)
- Variables: camelCase, except *_id in Models and Data uses snake_case
- Typed envelope/DTO parameters and local `*Data` holders: variable name = class short name in camelCase (`$employeeEnvelope`, not `$employee`; `$customerAddressData`, not `$addr`)
- Multi-argument calls use named arguments in a consistent order; single-argument calls may remain positional
- Methods: verbal and explicit
- Required domain dependencies in signatures stay non-nullable (`Product $product`, not `?Product $product`) unless the plan models absence explicitly — see `docs/07-anti-patterns.md` §28