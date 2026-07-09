# Philosophy

This document describes the **core principles** behind how we build Laravel applications at Labrodev.  
These principles guide structure, naming, abstractions, and trade-offs.

They are not rules for rules’ sake.  
They exist to reduce entropy, cognitive load, and long-term maintenance cost.

---

## 1. There are no best practices — only contexts

We reject universal “best practices”.

Every architectural decision is:
- contextual
- reversible (when done well)
- a trade-off between simplicity, clarity, and flexibility

This codebase optimizes for:
- long-lived SaaS products
- small to medium teams
- frequent feature changes
- AI-assisted development

If a rule no longer serves the context, it should be questioned — not defended.

---

## 2. Readability beats cleverness

Code is read far more often than it is written.

We prefer:
- explicit code over magic
- boring solutions over smart ones
- duplication over premature abstraction

If something requires explanation to understand, it is likely too complex.

---

## 3. Structure is a communication tool

Folder structure is not an implementation detail — it is a **map of intent**.

A developer should understand:
- where business logic lives
- where infrastructure concerns live
- where orchestration happens

…without reading the code itself.

We use structure to answer the question:

> “Where should I put this?”

before it becomes:

> “Why is this here?”

---

## 4. Controllers are I/O, not business logic

Controllers exist to:
- receive input
- validate intent
- delegate work
- return output

They do **not**:
- make decisions
- contain workflows
- mutate business state directly

If logic grows, it moves out — not deeper in.

---

## 5. Business logic must have a home

Every piece of business logic must live **somewhere intentional**:
- an Action сlass
- a Domain service
- a Policy
- a Rule object
- an Aggregate-like coordinator
- an Orchestrator (if need Pipelines)

Logic that “floats” between controllers, models, and helpers is technical debt.

If logic has no clear home, that is a design smell.

---

## 6. Eloquent models are not the domain

Eloquent models represent **persistence**, not **business concepts**.

They may:
- define relationships
- define casts
- enforce simple invariants
- attach the observer via `#[ObservedBy]` on the model class (Laravel 13)

They should not:
- expose query scopes
- coordinate workflows
- perform cross-entity logic
- contain application use cases

We prefer anemic models with explicit orchestration layers.

---

## 7. Naming is architecture

Good naming reduces the need for comments and explanations.

We value:
- verbs for actions (`InvoiceCreate`)
- nouns for data (`InvoiceData`)
- model names are singular
- controller names are singular
- explicit intent over generic terms (`Handle`, `Process`, `Manager` are avoided)
- use named arguments when a method/function has more than one argument (for example: `$productCreate(productData: $productData, supplier: $supplier)`); keep names in a consistent order — see [02-naming.md](02-naming.md)
- typed Data/envelope variables use the full class short name in camelCase (e.g. `DraftData` → `$draftData`, not `$data`) — see [02-naming.md](02-naming.md)

If naming is hard, the abstraction is probably wrong.

---

## 8. Explicit flows over hidden side effects

Hidden behavior is expensive.

We are cautious with:
- model observers
- global events
- magic hooks
- implicit mutations

Observers may enforce **invariants**, but should not implement workflows.

If behavior matters, it should be visible in the call stack.

---

## 9. Tests describe behavior, not implementation

Tests exist to:
- describe intent
- protect behavior
- allow refactoring

They are not:
- documentation of internal methods
- duplication of implementation logic

If tests break during refactoring but behavior does not change, the test is wrong.

---

## 10. Change is expected

This architecture assumes:
- features will change
- requirements will shift
- early decisions may be wrong

Therefore:
- abstractions should be shallow
- refactoring should be safe
- removing code should be easy

We optimize for **change velocity**, not theoretical purity.

---

## 11. AI is a collaborator, not a source of truth

AI tools are part of the development workflow.

This codebase is designed so that:
- AI suggestions are constrained by structure
- generated code follows our conventions
- architectural intent is machine-readable

AI accelerates execution.  
Humans remain responsible for decisions.

---

## 12. Consistency is kindness

Consistency helps:
- future teammates
- future you
- AI tools
- reviewers

We prefer a consistent “good enough” solution over multiple “perfect” ones.

---

## Closing note

These principles are intentionally opinionated.

They are meant to:
- guide decisions
- reduce debate
- encode experience

They are not immutable laws.

When reality disagrees with philosophy — **revisit the philosophy**.
