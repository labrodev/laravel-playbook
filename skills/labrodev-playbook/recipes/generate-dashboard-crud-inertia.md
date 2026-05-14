# Recipe: Generate Dashboard CRUD (Inertia)

Attach:
- docs/index.md
- docs/10-stubs.md
- docs/11-inertia-react.md
- stubs/app/controller.inertia.stub
- stubs/app/indexQuery.stub
- stubs/app/viewModel.inertia.index.stub
- stubs/app/viewModel.inertia.show.stub
- stubs/app/export.stub

Prompt:
Generate Layer code for Domain {Domain} with Model {Model} (Dashboard surface):
- Controller: `App/Layer/Dashboard/{Domain}/Controllers/{Model}Controller.php`
- IndexQuery: `App/Layer/Dashboard/{Domain}/IndexQueries/{Model}IndexQuery.php`
- ViewModels: {Model}IndexViewModel, {Model}ShowViewModel (business data and options only; UI copy in Laravel `lang/` + `t()` per `docs/11-inertia-react.md` §10)
- Export: {Model}Export uses IndexQuery->getQuery()
  Constraints:
- No FormRequest, no validate() in controller.
- Store/Update params via Spatie Data classes only.
- Controller is dispatcher only, business logic in Core Actions/Orchestrators.
  Output:
- Provide exact file paths and full code for each file.