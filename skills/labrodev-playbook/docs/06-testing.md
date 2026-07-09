# Testing

This document defines how testing is approached in Labrodev projects.

Testing exists to protect behavior, enable refactoring, and make intent explicit. Tests are not written to satisfy coverage metrics or to mirror implementation details. A good test explains what the system does and fails only when behavior changes.

---

## Core principles

- Test behavior, not implementation.
- Prefer explicit tests over clever ones.
- Tests must reflect the architecture, not fight it.
- Business behavior is tested in Core, not in `App/Layer`.
- If a test breaks during refactoring but behavior did not change, the test is wrong.

---

## Test pyramid (how we think about tests)

We follow a pragmatic test pyramid:

1. Action and Orchestrator tests (most important)
2. Domain unit tests (Rules, Services, Pipelines)
3. Job tests
4. HTTP / Layer tests (few, high-level)

The closer a test is to business behavior, the more valuable it is.

---

## What to test (by layer)

### Actions

Actions are the primary entry point for business behavior and must always be tested.

Test Actions when:
- they mutate state
- they enforce business rules
- they coordinate multiple domain objects

Action tests should:
- construct required Data objects
- call the Action as invokable: `$productCreate(productData: $productData);` (or `($productCreate)(...)`)
- assert resulting state changes
- assert thrown domain exceptions when applicable

Action tests must not:
- assert internal method calls
- mock domain logic excessively
- rely on HTTP, controllers, or UI concerns

Actions are tested directly, not through controllers.

---

### Orchestrators

Orchestrators coordinate workflows and must be tested when they exist.

Test Orchestrators when:
- multiple Actions are coordinated
- conditional flows exist
- sequencing matters

Orchestrator tests should:
- assert the final outcome of the workflow
- verify side effects (state changes, events dispatched)
- avoid testing internal step ordering unless behavior depends on it

Orchestrators are tested as behavior units, not as sequences of calls.

---

### Rules

Rules encapsulate small decision logic and are ideal unit test candidates.

Rule tests should:
- cover edge cases clearly
- use simple inputs
- avoid database access when possible

Rules should be easy to test. If testing a Rule is hard, its responsibility is likely too large.

---

### Services

Services should be tested when they contain non-trivial logic.

Service tests should:
- focus on inputs and outputs
- avoid mocking internal helpers
- avoid testing through Actions unless integration is required

Pure services should be testable without Laravel bootstrapping when possible.

---

### Pipelines

Pipelines should be tested when:
- order of steps affects outcome
- steps transform payloads in non-trivial ways

Pipeline tests should:
- assert final payload state
- avoid asserting intermediate steps unless required

---

### Jobs

Jobs are infrastructure wrappers and are tested differently.

Test Jobs when:
- retry/backoff configuration matters
- job dispatching is critical
- job delegates correctly to Actions or Orchestrators

Job tests should:
- assert delegation, not business logic
- avoid re-testing Action behavior inside Job tests

Jobs must not be tested as business units.

---

### Events and listeners

Events are data-only and usually do not require direct tests.

Test listeners when:
- they trigger important side effects
- they delegate to Actions or Orchestrators

Listener tests should:
- assert that the correct Action/Orchestrator is called
- avoid deep assertions about domain behavior

---

## What not to test

Do not write tests for:
- getters/setters
- trivial accessors
- framework behavior
- Eloquent relationships (unless customized logic exists)
- simple casts or visibility configuration

If a test only proves that Laravel works, it should not exist.

---

## Layer and HTTP tests

HTTP and Layer tests exist only to verify wiring, not business logic.

Use HTTP tests to:
- verify routes are connected
- verify authentication/authorization wiring
- verify request-to-Action delegation

HTTP tests must not:
- re-test business rules
- assert internal domain logic
- contain complex setup for domain state

Keep HTTP tests few and high-level.

---

## Factories in tests

Factories are used to build test fixtures.

Rules:
- use factories to set up state, not to perform behavior
- prefer explicit factory states over random data
- avoid factories that hide important invariants

Factories must not replace Actions when testing business behavior.

---

## Data objects in tests

Data objects are part of the input contract and should be used in tests.

Rules:
- construct Data objects explicitly in tests
- avoid passing raw arrays to Actions
- test validation behavior separately when needed

Validation failures should be tested at the Data level, not via Actions.

---

## Mocks and fakes

Use mocks sparingly.

Allowed:
- external services (email, APIs)
- time, UUID generation when required
- infrastructure adapters

Avoid mocking:
- domain logic
- Actions
- Rules
- Services under test

If heavy mocking is required, the design likely needs improvement.

---

## Naming and structure

Test names must be descriptive and behavior-focused.

Good examples:
- it_creates_product_with_valid_data
- it_fails_when_product_is_archived
- it_dispatches_event_after_creation

Avoid:
- test1
- handle_test
- process_product

Test files should mirror domain structure where possible.

---

## Summary

- Test business behavior in Actions and Orchestrators
- Test small logic in Rules and Services
- Jobs are tested as wrappers, not brains
- Layer / HTTP tests verify wiring only
- Use factories for setup, not behavior
- Avoid testing Laravel itself
- Favor clarity over coverage