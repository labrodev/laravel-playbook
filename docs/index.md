# Documentation Index (Source of Truth)

This folder contains the authoritative documentation for the Labrodev
architecture and conventions used in this project.

All documents are **normative**, not illustrative.

If a rule is documented here, it must be followed.
If a rule is not documented here, it must not be assumed.

---

## How to use this documentation

- This index defines the **canonical reading order**
- Each numbered document is a self-contained chapter
- Rules in later chapters build on earlier ones
- When in doubt, consult the most specific chapter
- For domain-specific context and ubiquitous language, consult the
  `README.md` file inside the relevant `Core/Domain/<Domain>` module.
---

## Precedence rules

1. **Stubs** define *how* code must look  
   → see `docs/10-stubs.md`

2. **Documentation** defines *why* and *where*  
   → see relevant chapter below

3. **Workspace rules** (e.g. `.cursor/rules`) enforce behavior  
   → they summarize this tree; they are not a substitute for reading `docs/`

If documentation and stubs appear to conflict:
- Follow the resolution rules in `docs/10-stubs.md`

---

## Documentation map

- `00-philosophy.md`  
  Core principles, values, and non-negotiables

- `01-project-structure.md`  
  Folder layout, what `App/Layer/{Layer}` means, how Layers mirror Domains, and responsibility boundaries

- `02-naming.md`  
  Naming rules for domains, classes, methods, variables

- `03-boundaries.md`  
  Dependency direction and forbidden couplings

- `04-data-and-validation.md`  
  Input mapping, validation rules, and write/read separation

- `05-database-and-models.md`  
  Models, Eloquent attributes (`ObservedBy`, `CollectedBy`, `UsePolicy`, `UseFactory`), collections, observers, invokable controller **`#[Authorize]`**

- `06-testing.md`  
  Testing strategy, structure, and allowed patterns

- `07-anti-patterns.md`  
  Explicitly forbidden approaches and shortcuts (Carbon §25, Queries §26, i18n §19, DI §27, nullability §28)

- `08-immutability-and-inheritance.md`  
  `final`, `readonly`, and extensibility rules

- `09-tooling.md`  
  Pint, PHPStan, Rector, and enforcement expectations

- `10-stubs.md`  
  Architectural stubs, generation rules, and conflict resolution

- `11-inertia-react.md`  
  Inertia + React: page structure, Wayfinder, forms, layouts, and frontend boundaries

- `12-workflow.md`  
  Standard workflow for implementing tasks

- `13-review-checklist.md`  
  Acceptance criteria before opening a PR

---

## Scope

This documentation applies to:
- all handwritten code
- all generated code
- all AI-assisted code

Deviation is not allowed unless explicitly documented.
