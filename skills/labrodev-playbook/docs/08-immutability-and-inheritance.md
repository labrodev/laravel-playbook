## Immutability and inheritance

By default, classes are not designed for inheritance.

Rules:

- Classes should be declared `final` unless extension is explicitly required.
- Data-carrying objects should be declared `readonly` when possible.
- Mutability and inheritance must be intentional decisions, not defaults.

### final classes

Use `final` for:
- Actions
- Casts
- Collections
- Data
- Events
- Factories
- Jobs
- Models
- Observers
- Payloads
- Pipelines
- Policies
- Queries
- Resources
- Rules
- Services
- Utilities

PHP traits cannot be declared `final`; do not list traits as `final` in reviews.

- Controllers
- ViewModels
- IndexQueries

Do not use `final` only when:
- the class is a framework base abstraction
- the class is explicitly designed for extension (rare)

### readonly classes

Use `readonly` for:
classes where there are no necessity to modify attributes.

Readonly guarantees:
- no accidental mutation
- safer refactoring
- clearer intent

If a class is not readonly, there must be a clear reason.
