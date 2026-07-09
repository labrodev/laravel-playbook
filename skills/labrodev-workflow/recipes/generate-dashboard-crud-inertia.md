# Recipe: Generate Dashboard CRUD (Inertia)

Attach:
- docs/index.md
- docs/10-stubs.md
- docs/11-inertia-react.md
- stubs/app/dashboard/controllers/controller.inertia.stub
- stubs/app/dashboard/controllers/controller.inertia.write.stub
- stubs/app/dashboard/indexQueries/indexQuery.stub
- stubs/app/dashboard/viewModels/viewModel.inertia.index.stub
- stubs/app/dashboard/viewModels/viewModel.inertia.show.stub
- stubs/app/dashboard/exports/export.stub
- stubs/routes/routes.stub

Prompt:
Generate Layer code for Domain {Domain} with Model {Model} (Dashboard surface):
- Controllers (one invokable class per action, under `App/Layer/Dashboard/{Domain}/Controllers/`):
  `{Model}IndexController.php`, `{Model}ShowController.php`, `{Model}StoreController.php`,
  `{Model}UpdateController.php`, `{Model}RemoveController.php`
- IndexQuery: `App/Layer/Dashboard/{Domain}/IndexQueries/{Model}IndexQuery.php`
- ViewModels: {Model}IndexViewModel, {Model}ShowViewModel (business data and options only; UI copy in Laravel `lang/` + `t()` per `docs/11-inertia-react.md` §10)
- Export: {Model}Export uses IndexQuery->getEloquentBuilder()
- Routes: explicit per-action routes with `{model:uuid}` binding per `stubs/routes/routes.stub`
  Constraints:
- No FormRequest, no validate() in controller.
- Store/Update params via Spatie Data classes only (injected into __invoke).
- Actions invoked as callables with named arguments.
- Controller is dispatcher only, business logic in Core Actions/Orchestrators.
  Output:
- Provide exact file paths and full code for each file.
