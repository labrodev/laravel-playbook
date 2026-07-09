# Recipe: Implement Page from Prototype

## Task Description

This recipe provides a comprehensive checklist for implementing a complete CRUD page from prototype to production, following Labrodev architecture patterns. Use this when implementing new pages or enhancing existing ones based on prototype designs.

## Prerequisites

Before starting:
- [ ] Read `../index.md` (or `../SKILL.md`) and follow the documentation map / precedence rules
- [ ] Consult `docs/index.md` and relevant chapters for deeper context
- [ ] Read `docs/11-inertia-react.md` for all frontend structure and conventions
- [ ] Review `docs/13-review-checklist.md` as acceptance criteria
- [ ] Identify the domain and model for the page
- [ ] Review prototype component(s) in `../prototype/components/`

## Implementation Checklist

### 1. Backend - Core Domain Layer

#### 1.1 Create/Verify Resource Class
- [ ] File: `core/src/Domain/{Domain}/Resources/{Model}Resource.php`
- [ ] Follow pattern from existing Resource (e.g., `CustomerResource.php`)
- [ ] Include all visible fields from model
- [ ] Format dates properly (`Y-m-d H:i:s`)
- [ ] Use `fetchModel()` pattern for type safety
- [ ] Extend `JsonResource`

#### 1.2 Update Policy
- [ ] File: `core/src/Domain/{Domain}/Policies/{Model}Policy.php`
- [ ] Add `show` method if missing:
  - [ ] Check `PERMISSION_VIEW` permission
  - [ ] If user has Supplier role, verify `$model->supplier_id === $user->supplier_id`
  - [ ] Import `Core\Domain\Platform\Enums\UserRoleCode`
  - [ ] **Add logging**: Log when model is viewed (who, when, which model)
    - Use `Log::channel('{domain}')->info()`
    - Format: `'{Model} %s has been viewed by %s (%s)'`
- [ ] Verify `view`, `create`, `update`, `remove` methods exist
- [ ] Ensure all methods check permissions properly

#### 1.3 Verify Observer Logging
- [ ] Model declares `#[ObservedBy({Model}Observer::class)]` (not `observe()` in `boot()`)
- [ ] File: `core/src/Domain/{Domain}/Observers/{Model}Observer.php`
- [ ] Verify `created()` method logs:
  - [ ] Who created it (`auth()->user()->name`)
  - [ ] When (automatic via Log)
  - [ ] Which model (uuid or id)
  - [ ] Format: `'%s %s has been created by %s (%s)'` with `Model::class` as the first value
- [ ] Verify `updated()` method logs:
  - [ ] Who updated it
  - [ ] When
  - [ ] Which model
  - [ ] Format: `'%s %s has been updated by %s (%s)'` with `Model::class` as the first value
- [ ] Verify `deleted()` method logs:
  - [ ] Who deleted it
  - [ ] When
  - [ ] Which model
  - [ ] Format: `'%s %s has been deleted by %s (%s)'` with `Model::class` as the first value
- [ ] All observer logs use the unified channel: `Log::channel('observer')`

### 2. Backend - Dashboard Layer

#### 2.1 Create IndexQuery
- [ ] File: `supplier/app/Dashboard/{Domain}/IndexQueries/{Model}IndexQuery.php`
- [ ] Follow pattern from `CustomerIndexQuery.php`
- [ ] Use Spatie QueryBuilder
- [ ] Configure allowed filters:
  - [ ] Exact filters: `AllowedFilter::exact('id', '{table}.id')`
  - [ ] Partial filters: `AllowedFilter::partial('name', '{table}.name')`
  - [ ] DateRange filters: `AllowedFilter::custom('created_at', new DateRangeFilter, '{table}.created_at')`
- [ ] Configure allowed sorts:
  - [ ] `AllowedSort::field('id', '{table}.id')`
  - [ ] `AllowedSort::field('name', '{table}.name')`
  - [ ] `AllowedSort::field('created_at', '{table}.created_at')`
- [ ] Set default sort: `$this->defaultSort('-{table}.id')`
- [ ] Use proper table name/alias if joins are used

#### 2.2 Update Controller
- [ ] File: `supplier/app/Dashboard/{Domain}/Controllers/{Model}Controller.php`
- [ ] **index method**:
  - [ ] Inject `{Model}IndexQuery` as parameter
  - [ ] Authorize using `{Model}Policy::PERMISSION_VIEW`
  - [ ] Paginate query: `$query->paginate(request()->get('per_page', 50))->withQueryString()`
  - [ ] Create ViewModel and render Inertia page
- [ ] **show method**:
  - [ ] Authorize using `{Model}Policy::show()` (includes supplier check)
  - [ ] Load relations if needed
  - [ ] Create ViewModel and render Inertia page
- [ ] **create method** (if needed):
  - [ ] Return Inertia render with empty form data
- [ ] **store method**:
  - [ ] Inject `{Model}Create` action as parameter
  - [ ] Authorize using `{Model}Policy::PERMISSION_CREATE`
  - [ ] Validate: `{Model}Data::from($request->all())`
  - [ ] Invoke: `$productCreate(productData: $productData);`
  - [ ] Redirect to show page with success message
- [ ] **update method**:
  - [ ] Inject `{Model}Update` action as parameter
  - [ ] Authorize using `{Model}Policy::PERMISSION_UPDATE`
  - [ ] Validate: `{Model}Data::from($request->all())`
  - [ ] Invoke: `$productUpdate(product: $product, productData: $productData);`
  - [ ] Redirect back with success message
- [ ] **delete method**:
  - [ ] Inject `{Model}Remove` action as parameter
  - [ ] Authorize using `{Model}Policy::PERMISSION_REMOVE`
  - [ ] Invoke: `$productRemove($product);`
  - [ ] Redirect to index with success message
- [ ] **No Request classes** - use Data objects only
- [ ] **No validation in controller** - validation in Data classes
- [ ] **No business logic** - delegate to Actions

#### 2.3 Update IndexViewModel
- [ ] File: `supplier/app/Dashboard/{Domain}/ViewModels/{Model}IndexViewModel.php`
- [ ] Extend `IndexViewModel`
- [ ] Use `{Model}Resource::collection()` to transform items
- [ ] Include pagination data: current_page, last_page, per_page, total, from, to, links
- [ ] Add translations for all columns
- [ ] Follow pattern from `CustomerIndexViewModel.php`

#### 2.4 Update ShowViewModel
- [ ] File: `supplier/app/Dashboard/{Domain}/ViewModels/{Model}ShowViewModel.php`
- [ ] Extend `ViewModel`
- [ ] Use `{Model}Resource` to transform model
- [ ] Add all necessary translations
- [ ] Include relations if needed (load in controller)
- [ ] Follow pattern from `CustomerShowViewModel.php`

#### 2.5 Add Routes
- [ ] File: `supplier/routes/supplier.php` (or appropriate route file)
- [ ] Add route group with prefix and name:
  ```php
  Route::prefix('{resource}')->name('{resource}.')->group(function () {
      Route::get('/', [{Model}Controller::class, 'index'])->name('index');
      Route::get('create', [{Model}Controller::class, 'create'])->name('create');
      Route::post('/', [{Model}Controller::class, 'store'])->name('store');
      Route::get('{model:uuid}', [{Model}Controller::class, 'show'])->name('show');
      Route::put('{model:uuid}', [{Model}Controller::class, 'update'])->name('update');
      Route::delete('{model:uuid}', [{Model}Controller::class, 'delete'])->name('delete');
  });
  ```
- [ ] Use `{param:uuid}` on every route segment that binds a model (do not use `getRouteKeyName()` on the model)
- [ ] Ensure proper middleware/authorization

#### 2.6 Route parameters (UUID)
- [ ] Declare explicit UUID binding in routes (`{booking:uuid}`, `{service:uuid}`, etc.); do not add `getRouteKeyName()` to the model
- [ ] Ensure all route references use UUID (not ID)
- [ ] Frontend DataTable must use `idKey="uuid"` when URLs use UUID segments
- [ ] All `viewHref` callbacks must use `item.uuid` (not `item.id`)

### 3. Frontend - React/TypeScript

#### 3.1 Update/Create Index Page
- [ ] File: `supplier/resources/js/pages/{resource}/index.tsx`
- [ ] Use `DataTable` component (from `@/components/dashboard/data-table`)
- [ ] Define columns with proper configuration:
  - [ ] `key`: field name
  - [ ] `label`: translation key or default text
  - [ ] `sortable`: true/false
  - [ ] `filterable`: true/false
  - [ ] `filterType`: 'text' | 'select' | 'boolean' | 'daterange'
  - [ ] `filterOptions`: for select type
  - [ ] `render`: custom render function if needed
- [ ] Use `supplier.{resource}.index().url` for baseUrl
- [ ] Use `supplier.{resource}.show(item.uuid).url` for viewHref
- [ ] **IMPORTANT**: Set `idKey="uuid"` when routes bind by UUID (`{model:uuid}`)
  - [ ] DataTable must use `idKey="uuid"` (not default `id`) for those resources
  - [ ] All route references must use `item.uuid` (not `item.id`)
- [ ] Add proper TypeScript interface for model
- [ ] Add translations support
- [ ] Add breadcrumbs
- [ ] Add page header with title and description
- [ ] Add "Create" button linking to create page
- [ ] Handle empty state

#### 3.2 Create Show/Create/Edit Form Page
- [ ] File: `supplier/resources/js/pages/{resource}/show.tsx` (or separate create.tsx/edit.tsx)
- [ ] Follow pattern from `customers/show.tsx` or similar pages
- [ ] Create form with fields matching `{Model}Data`:
  - [ ] Required fields marked appropriately
  - [ ] Optional fields handled correctly
  - [ ] Proper input types (text, email, number, date, etc.)
  - [ ] Array fields (if applicable)
  - [ ] Select/dropdown for relations
- [ ] Use Inertia form handling:
  - [ ] `useForm` hook from `@inertiajs/react`
  - [ ] Handle form submission
  - [ ] Show validation errors
  - [ ] Show success messages
- [ ] Handle both create and edit modes:
  - [ ] Check if model exists (edit) or null (create)
  - [ ] Pre-fill form data in edit mode
  - [ ] Use correct route for submission (store vs update)
- [ ] Add proper breadcrumbs
- [ ] Add back button to index
- [ ] Add cancel button
- [ ] Add save/submit button
- [ ] Show loading state during submission

#### 3.3 Update Routes Helper
- [ ] File: `supplier/resources/js/routes/supplier.ts` (if exists)
- [ ] Ensure route helper methods exist:
  - [ ] `supplier.{resource}.index()`
  - [ ] `supplier.{resource}.create()`
  - [ ] `supplier.{resource}.store()`
  - [ ] `supplier.{resource}.show(uuid)`
  - [ ] `supplier.{resource}.update(uuid)`
  - [ ] `supplier.{resource}.delete(uuid)`

### 4. Database Migrations

#### 4.1 UUID Population Migration (if needed)
- [ ] Check if model table has UUID column (from `2026_02_03_144414_add_uuid_columns_to_domain_tables.php`)
- [ ] If UUID column exists but may have NULL values, create migration to populate:
  - [ ] File: `supplier/database/migrations/YYYY_MM_DD_HHMMSS_populate_{table}_uuid_fields.php`
  - [ ] Follow pattern from `2026_02_03_200000_populate_customers_uuid_fields.php`
  - [ ] Process records in chunks of 100 to avoid memory issues
  - [ ] Only update records where `uuid` is NULL or empty
  - [ ] Generate unique UUIDs using `Str::uuid()`
  - [ ] Include uniqueness check (very unlikely but safe)
  - [ ] Note: Migration cannot be reversed (as per pattern)
- [ ] Run migration: `php artisan migrate`
- [ ] Verify UUIDs are populated for existing records

### 5. Code Quality Checks

#### 5.1 Run Code Quality Tools
- [ ] Run `./vendor/bin/pint` to check and fix code style
  - [ ] Fix any style issues found
  - [ ] Ensure consistent formatting
- [ ] Run `./vendor/bin/rector` to check for refactoring opportunities
  - [ ] Review suggested changes
  - [ ] Apply safe refactorings
- [ ] Run `./vendor/bin/phpstan` to check for static analysis issues
  - [ ] Fix any type errors
  - [ ] Ensure proper type hints
  - [ ] Resolve any static analysis warnings

#### 5.2 Architecture Compliance
- [ ] Controllers are thin and delegate to Actions
- [ ] Actions injected as method arguments (no `app()`, `resolve()`, or `new`)
- [ ] No Request classes introduced (use Data objects)
- [ ] No mass assignment or `$fillable` used
- [ ] IndexQueries use Spatie QueryBuilder
- [ ] ViewModels use Resources for transformation
- [ ] Frontend uses DataTable component for consistency
- [ ] Proper authorization checks in all methods
- [ ] Follow naming conventions (singular, entity-first)
- [ ] No business logic in Dashboard layer

### 6. Logging Verification

#### 6.1 Observer Logging
- [ ] Verify `created()` logs are written:
  - [ ] Check log channel is `observer`
  - [ ] Verify format includes: model identifier, user name, user id
  - [ ] Test by creating a new record
- [ ] Verify `updated()` logs are written:
  - [ ] Check log channel is `observer`
  - [ ] Verify format
  - [ ] Test by updating a record
- [ ] Verify `deleted()` logs are written:
  - [ ] Check log channel is `observer`
  - [ ] Verify format
  - [ ] Test by deleting a record

#### 6.2 Policy View Logging
- [ ] Verify `show()` method logs are written:
  - [ ] Check log channel matches domain
  - [ ] Verify format: `'{Model} %s has been viewed by %s (%s)'`
  - [ ] Test by viewing a record
  - [ ] Verify logs include correct user information

### 7. Testing

#### 7.1 Automated Tests
- [ ] Create tests for IndexQuery:
  - [ ] Test filters (exact, partial, dateRange)
  - [ ] Test sorts (ascending, descending)
  - [ ] Test pagination
- [ ] Create tests for Controller:
  - [ ] Test `index` method (authorization, pagination)
  - [ ] Test `show` method (authorization, supplier check)
  - [ ] Test `store` method (validation, authorization, creation)
  - [ ] Test `update` method (validation, authorization, update)
  - [ ] Test `delete` method (authorization, deletion)
- [ ] Test Policy authorization:
  - [ ] Test supplier relationship check
  - [ ] Test permission checks
- [ ] Test Observer logging:
  - [ ] Verify logs are created for created/updated/deleted
- [ ] Test Policy logging:
  - [ ] Verify view logs are created
- [ ] Test form validation:
  - [ ] Test required fields
  - [ ] Test field types
  - [ ] Test validation messages
- [ ] Test pagination:
  - [ ] Test empty results
  - [ ] Test single page
  - [ ] Test multiple pages
- [ ] Ensure all tests pass

#### 7.2 Manual Testing
- [ ] Test index page:
  - [ ] List displays correctly
  - [ ] Filters work
  - [ ] Sorting works
  - [ ] Pagination works
- [ ] Test create form:
  - [ ] Form displays correctly
  - [ ] Validation works
  - [ ] Submission creates record
  - [ ] Redirects correctly
- [ ] Test edit form:
  - [ ] Form pre-fills correctly
  - [ ] Validation works
  - [ ] Submission updates record
  - [ ] Redirects correctly
- [ ] Test delete:
  - [ ] Confirmation works (if applicable)
  - [ ] Record is deleted
  - [ ] Redirects correctly
- [ ] Test authorization:
  - [ ] Unauthorized users cannot access
  - [ ] Supplier users can only see their own records

### 8. Final Checklist

- [ ] All code quality tools pass (pint, rector, phpstan)
- [ ] All automated tests pass
- [ ] Manual testing completed
- [ ] Logging verified (observer and policy)
- [ ] Code review completed (if applicable)
- [ ] Documentation updated (if needed)
- [ ] Ready for deployment

## Common Patterns

### DataTable Column Configuration

Use the project i18n hook (see `docs/11-inertia-react.md` §10); do not pull column labels from ViewModel `translations` props.

```typescript
const t = useI18n();

const columns: Column<Model>[] = [
    {
        key: 'name',
        label: t('Name'),
        sortable: true,
        filterable: true,
        filterType: 'text',
    },
    {
        key: 'created_at',
        label: t('Created at'),
        sortable: true,
        filterable: true,
        filterType: 'daterange',
    },
];

// IMPORTANT: When routes use {resource:uuid}, set idKey="uuid"
<DataTable
    columns={columns}
    data={items}
    baseUrl={supplier.resource.index().url}
    viewHref={(item) => supplier.resource.show(item.uuid).url}
    idKey="uuid"
/>
```

### Policy Show Method Pattern
```php
public function show(Authenticatable $user, Model $model): bool
{
    if (!$user->hasPermissionTo(self::PERMISSION_VIEW)) {
        return false;
    }

    if ($user->hasRole(UserRoleCode::Supplier->value)) {
        if ($user->supplier_id !== $model->supplier_id) {
            return false;
        }
    }

    Log::channel('{domain}')->info(sprintf(
        '{Model} %s has been viewed by %s (%s)',
        $model->uuid ?? $model->id,
        $user->name ?? 'system',
        $user->id ?? 'noid'
    ));

    return true;
}
```

### Observer Logging Pattern
```php
public function created(Model $model): void
{
    Log::channel('observer')->info(sprintf(
        '%s %s has been created by %s (%s)',
        Model::class,
        $model->uuid ?? $model->id,
        auth()->user()->name ?? 'system',
        auth()->user()->id ?? 'noid'
    ));
}
```

### UUID Population Migration Pattern
```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Str;

return new class extends Migration
{
    public function up(): void
    {
        DB::table('{table_name}')
            ->where(function ($query) {
                $query->whereNull('uuid')
                    ->orWhere('uuid', '');
            })
            ->orderBy('id')
            ->chunkById(100, function ($records) {
                foreach ($records as $record) {
                    $uuid = (string) Str::uuid();
                    
                    while (DB::table('{table_name}')->where('uuid', $uuid)->exists()) {
                        $uuid = (string) Str::uuid();
                    }

                    DB::table('{table_name}')
                        ->where('id', $record->id)
                        ->update(['uuid' => $uuid]);
                }
            });
    }

    public function down(): void
    {
        // Cannot be reversed
    }
};
```

## Notes

- Always follow Labrodev architecture patterns
- Use stubs as reference for structure
- Keep controllers thin - delegate to Actions
- Use Data objects for validation, not Request classes
- Ensure proper authorization at all levels
- Log all important actions (create, update, delete, view)
- **Route UUID binding**: Routes must use `{param:uuid}`; models must not rely on `getRouteKeyName()`. Ensure:
  - Frontend DataTable uses `idKey="uuid"` when listing by UUID URLs
  - All route references use `item.uuid` (not `item.id`)
  - UUIDs are populated for existing records before UUID route binding in production
- **UUID Migration**: If UUID column exists but may have NULL values, create and run UUID population migration before deploying route model binding
- Test thoroughly before considering complete

## Lessons Learned (from ABO-5 Implementation)

### Route and Action Verification
- **Always verify routes return 200 or 302, not 404**: Test all buttons and form submissions to ensure routes are correctly defined and match controller method names
- **Route naming consistency**: If a page has a "Save" button that calls an `update` action, the route should be `edit` (not `show`). The route name should reflect the action being performed:
  - `show` route → read-only display page
  - `edit` route → form page with save/update functionality
- **Controller method naming**: Ensure controller method names match route names (`show` vs `edit`)

### UUID Usage
- **Always use UUIDs in routes**: Declare `{param:uuid}` on route segments; use `item.uuid` in Wayfinder/links:
  - All route helpers must use `item.uuid` (not `item.id`)
  - DataTable must have `idKey="uuid"` prop for those resources
  - Frontend navigation must use UUIDs consistently
- **UUID population**: Ensure UUIDs are populated for all related tables before creating migrations that reference them (e.g., geo tables before area_places migration)

### Code Organization and Architecture
- **Use ViewModels for all pages**: Both `create` and `edit` operations should use ViewModels to prepare data. Consider creating a shared `{Model}FormViewModel` for both operations instead of separate ViewModels
- **Atomize ViewModel methods**: Break down large `toArray()` methods into smaller, focused helper methods (e.g., `getAreaData()`, `getAvailablePlaces()`, `getTravelModeOptions()`)
- **Constructor injection**: Always use constructor injection for dependencies in Actions, ViewModels, and other classes. Never use `resolve()`, `app()`, or `new` for dependencies
- **Filter by supplier_id**: In `IndexQuery`, always filter results by the authenticated user's `supplier_id` when applicable to ensure users only see their own data

### Enums and Type Safety
- **Use Enums for constrained values**: Create PHP Enums (e.g., `TravelMode`) for fields with limited options instead of magic numbers or strings
- **Use EnumMapper for options**: When you need enum options for selects/validation, use `EnumMapper::keyValues(Enum::cases())` instead of custom methods or `array_column`
- **Custom casts when stored values disagree with the enum**: When the database still contains historical strings or other values that do not match current enum cases, use a custom cast (e.g. `TravelModeCast`) to normalize or reject invalid values; pair with data migrations when you need to fix rows at rest
- **Import Rule explicitly**: Use `use Illuminate\Validation\Rule;` instead of fully qualified class names in validation rules

### Data Classes and Validation
- **Data classes should NOT be readonly**: `Spatie\LaravelData\Data` classes cannot be `readonly` because they extend a non-readonly base class. Remove `readonly` modifier from all Data classes
- **Always include all items in arrays**: When submitting form data with nested arrays (e.g., `place_postal_codes`), always include entries for all items, even if their arrays are empty. Don't conditionally exclude items based on whether they have values
- **Named arguments**: Use named arguments for all custom method calls (Actions, `ObjectFailedToCreate::make`, etc.) for clarity and to prevent parameter order mistakes

### Frontend Considerations
- **Validation error display**: Ensure validation errors are prominently displayed:
  - Add a general validation error alert at the top of forms
  - Display field-level errors near each input
  - Check that `FlashAlert` component correctly detects validation errors from Inertia
- **Default form state**: When adding items via modals (e.g., places to an area), set sensible defaults:
  - Newly added items should be collapsed by default (`isExpanded: false`)
  - All related items (e.g., postal codes) should be selected by default
  - Reset modal filters when closing

### Testing and Quality
- **Test everything**: After implementation, verify:
  - All routes work (200/302 responses)
  - Form submissions work correctly
  - Validation errors display properly
  - Data persists correctly
  - Authorization works (supplier filtering, permissions)
- **Run code quality tools**: Always run `pint`, `rector`, and `phpstan` before considering the task complete
- **Run autotest**: Add "autotest" as the final step in the implementation process

### Database Migrations
- **Migration order matters**: When migrations depend on UUIDs from other tables, ensure UUID population migrations run first
- **Existing rows after enum or shape changes**: When changing enum values or data structures, add migrations that update already-persisted records so stored data matches the new schema before the app relies on it
- **Mass assignment avoidance**: Never use mass assignment. Always explicitly set model attributes one by one, even when creating related records

### Common Pitfalls to Avoid
- **Don't use `resolve()`**: Use constructor injection instead
- **Don't make Data classes readonly**: They extend non-readonly base classes
- **Don't skip ViewModels**: Even `create` operations need ViewModels
- **Don't forget supplier filtering**: Always filter by `supplier_id` in IndexQueries for supplier-scoped resources
- **Don't conditionally exclude form data**: Always send complete data structures, even with empty arrays
- **Don't mix route naming**: If it's an edit form, use `edit` route, not `show`
