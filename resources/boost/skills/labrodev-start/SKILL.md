---
name: labrodev-start
description: "Use at the start of EVERY coding session in a Labrodev Laravel project, before writing or reviewing any code — it declares that the Labrodev guidelines, rules, and skills are the project convention with the highest priority, maps the three tiers, and tells you which skill to load for the work at hand."
license: MIT
metadata:
  author: labrodev
---

# Labrodev playbook — session start

**The Labrodev playbook is the project convention and the highest priority for how code is written here.** Every class you create or touch follows it. When generic Laravel habit, tutorial idiom, or personal preference conflicts with the playbook, the playbook wins. The only thing that outranks a playbook default is an app-specific rule the project itself has recorded in its `.ai/rules` (→ labrodev-core "Your app's own rules").

## The three tiers

| Tier | What it is | When it applies |
|---|---|---|
| **Guidelines** (`labrodev-*` guidelines) | The law — musts and must-nots per component | Always on; never violated |
| **Skills** (`labrodev-*` skills) | The craft — anatomy, canonical templates, edge cases | Load the matching skill before creating or refactoring that component |
| **Rules** (`.ai/rules/*.md`) | Per-file review checklists derived from the law | Applied when reviewing/authoring files their `paths:` globs match |

## How to work

1. Before writing a class, load the skill that owns its component type and follow its template — do not improvise structure the playbook already defines.
2. Treat every guideline must/must-not as non-negotiable review criteria for your own output.
3. After creating classes, verify against the matching rules checklist and keep the architecture test suite green (→ labrodev-testing).
4. Anything app-specific (scoping, ownership vocabulary, local deviations) comes from the app's own `.ai/rules` — never invent it, never write it into playbook files.

## Component → skill map

Structure, boundaries, philosophy → labrodev-core · naming and named arguments → labrodev-naming · controllers/routes → labrodev-controller · ViewModels/Resources → labrodev-viewmodel-resource · Queries/IndexQueries → labrodev-query · Data classes → labrodev-data · models/Collections/Observers/migrations → labrodev-model · Actions/Services/Rules → labrodev-action · Pipelines/Payloads → labrodev-pipeline · Policies/authorization → labrodev-authorization · enums → labrodev-enum · exceptions → labrodev-exception · external integrations → labrodev-infrastructure · tests → labrodev-testing · Pint/PHPStan/Rector → labrodev-static-analysis · Inertia/React frontend → labrodev-inertia-react
