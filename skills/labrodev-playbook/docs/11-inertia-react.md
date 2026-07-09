# Inertia + React

This document defines the canonical structure, conventions, and boundaries for the Inertia + React frontend layer in Labrodev projects.

The frontend is a **delivery surface**, not a business layer. It receives shaped data from ViewModels and renders it. All business logic, authorization, and state mutation happen on the backend.

These rules apply to all handwritten, generated, and AI-assisted frontend code.

---

## 1) Stack

The frontend stack is:

- React 19
- Inertia.js v2 (React adapter)
- TypeScript (strict)
- Tailwind CSS v4
- Wayfinder (route generation)
- Vite (bundling)

No additional frontend frameworks, state managers, or routing libraries may be introduced without approval.

---

## 2) Directory structure

Frontend code lives under `resources/js/` with the following layout:

- `pages/` — Inertia page components (one file per route)
- `layouts/` — layout wrappers (app, auth, settings)
- `components/` — shared reusable components
- `components/ui/` — primitive UI components (button, input, card, etc.)
- `components/dashboard/` — dashboard-specific composed components (DataTable, etc.)
- `hooks/` — custom React hooks
- `types/` — shared TypeScript type definitions
- `lib/` — utility functions (`cn()`, `http.ts`, etc.)

Generated Wayfinder routes live in:

- `@/actions/` — controller action functions
- `@/routes/` — named route functions

These directories are auto-generated. Do not edit files inside them manually.

### Co-located helper components

Non-page helper components that are scoped to a single domain may be co-located inside that domain's `pages/` subfolder. These are not Inertia page components and must not be default-exported.

Examples:
- `pages/supplier/drafts/DraftLayout.tsx` — layout wrapper specific to draft pages
- `pages/supplier/drafts/status-badge.tsx` — badge component used only in draft pages

If a helper component is used across multiple domains, move it to `components/`.

---

## 3) Page conventions

### File placement

Each Inertia page corresponds to a backend `Inertia::render('path/name', ...)` call. The page file lives at `resources/js/pages/{path/name}.tsx`.

Examples:
- `Inertia::render('supplier/bookings/index', ...)` → `pages/supplier/bookings/index.tsx`
- `Inertia::render('supplier/company-information', ...)` → `pages/supplier/company-information.tsx`

### Component shape

Every page is a default-exported React function component. The component receives props that match the ViewModel output.

```tsx
import { useI18n } from '@/hooks/use-i18n'; // project i18n hook

interface Props {
    items?: PaginatedData<Item>;
}

export default function ItemIndex({ items }: Props) {
    const t = useI18n();

    return (
        <AppLayout breadcrumbs={breadcrumbs}>
            <Head title={t('Items')} />
            {/* page content — all user-facing strings via t() per §10 */}
        </AppLayout>
    );
}
```

Required elements in every page:
- `<Head title={t('…')} />` — page title from i18n (`t()`), not hardcoded JSX (see §10)
- Layout wrapper (`AppLayout`, `AuthLayout`, or `SettingsLayout`)
- Breadcrumbs passed to the layout

- Page `Props` must **not** include `translations` (or any equivalent map of UI strings from the ViewModel). All copy uses Laravel `lang/` + frontend i18n + `t()` per §10.

### Naming

Page component names use PascalCase and match the resource + action:
- `BookingIndex`, `BookingShow`, `BookingForm`
- `CustomerIndex`, `CustomerEdit`
- `CompanyInformation`

---

## 4) Props contract

Page props are the **single source of data**. They come from the ViewModel's `toArray()` on the backend.

Rules:
- Props must be typed with an explicit TypeScript `interface Props` at the top of the file
- Props shape must match the ViewModel output exactly
- Pages must not fetch initial page data independently (no `fetch()`, `axios`, or `useEffect` for initial load). For in-page reactivity, see section 16 (JSON API calls).
- All user-facing text — page titles, labels, button text, descriptions, column headers, empty states, error messages — must come from **`t()`** backed by Laravel **`lang/*.json`** (see §10). ViewModels must not pass `translations` (or equivalent) for Inertia page copy, and pages must not read `props.translations` for labels.
- If a string is missing from `lang/` / the frontend dictionary, that is a **backend / asset pipeline bug**. The frontend must not invent or hardcode the string to compensate.

### Shared props

Shared data available on every page is typed in `types/index.ts` as `SharedData`:
- `auth` — current user
- `flash` — success/error flash messages
- `sidebarOpen` — sidebar state

Access shared data via `usePage<SharedData>()` only when needed.

### Select options and form metadata

ViewModels may provide additional props for form pages:
- `statusOptions` — array of `{ value, label }` for status dropdowns
- `serviceCategoryOptions` — array of `{ value, label }` for category selects
- `options` — grouped object with multiple select option sets

These must always originate from the backend ViewModel. The frontend must not hardcode option values.

```tsx
interface Props {
    campaign?: Campaign | null;
    statusOptions?: Array<{ value: number; label: string }>;
    serviceCategoryOptions?: Array<{ value: string; label: string }>;
}
```

---

## 5) Routing and navigation (Wayfinder)

All route references in the frontend must use Wayfinder-generated functions. Hand-written URL strings are forbidden.

### Import patterns

```tsx
import supplier from '@/routes/supplier';
import * as bookingsRoutes from '@/routes/supplier/bookings';
```

### Usage patterns

```tsx
// Dashboard link
supplier.dashboard().url

// Resource index
bookingsRoutes.index().url

// Resource show (UUID)
bookingsRoutes.show(item.uuid).url

// Resource create
bookingsRoutes.create().url

// Resource store (form submit target)
bookingsRoutes.store().url

// Resource update (with route model binding)
bookingsRoutes.update(item.uuid).url
// or with named parameter
customersRoutes.update({ customer: item.uuid }).url

// JSON API endpoint (see section 16)
supplier.matchingEngine.serviceTime.get.url()
```

### Navigation components

- Use `<Link href={...}>` from `@inertiajs/react` for navigation links
- Use `router.visit()`, `router.delete()`, etc. for programmatic navigation
- Never use `<a href="...">` with raw URLs for internal links

---

## 6) Forms

Forms use `useForm` from `@inertiajs/react`. No other form state management is allowed for Inertia page forms.

### Pattern

```tsx
const { data, setData, post, put, processing, errors } = useForm({
    field_name: initialValue ?? '',
});

const onSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (isEdit && item?.uuid) {
        put(itemRoutes.update(item.uuid).url);
    } else {
        post(itemRoutes.store().url);
    }
};
```

### Rules

- Form field names must match the backend Data class attributes (snake_case)
- Initial values come from the ViewModel props (pre-filled for edit, empty for create)
- Submit targets use Wayfinder route functions
- Validation errors from `errors` are displayed inline next to each field using `InputError`
- Flash messages (success/error) are displayed via `FlashAlert`
- `processing` state disables the submit button during submission
- Use `preserveScroll: true` when appropriate (edit forms)
- Add `autoComplete="off"` to inputs on business/company data forms (not auth forms) to prevent irrelevant browser suggestions

### Create vs Edit

A single form component may handle both create and edit:
- Determine mode: `const isEdit = Boolean(item?.uuid);`
- Pre-fill from props in edit mode, use empty defaults in create mode
- Submit to `store()` or `update()` accordingly

---

## 7) Layouts

Three layout families exist:

- `AppLayout` — main application shell with sidebar, used for all dashboard pages
- `AuthLayout` (and variants: split, simple, card) — authentication pages
- `SettingsLayout` — settings pages within the app shell

Every page must be wrapped in exactly one layout. Layouts receive:
- `breadcrumbs` — array of `BreadcrumbItem`
- `children` — the page content

Layout selection is determined by the page category, not by choice:
- Dashboard pages → `AppLayout`
- Auth pages → `AuthLayout` variant
- Settings pages → `SettingsLayout`

---

## 8) Shared components

### DataTable

`DataTable` from `@/components/dashboard/data-table` is the standard component for all index/listing pages.

Required props:
- `columns` — column definitions with key, label, sortable, filterable, filterType, render
- `data` — paginated data from the ViewModel
- `baseUrl` — Wayfinder route URL for the index page
- `viewHref` — function returning the show URL for a row
- `idKey` — set to `"uuid"` when list rows are identified by UUID in URLs (routes use `{model:uuid}`; see playbook § Route parameters)

### FlashAlert

`FlashAlert` from `@/components/flash-alert` displays success/error flash messages. Include it on form pages.

### InputError

`InputError` from `@/components/input-error` displays validation errors inline below form fields.

### Dashboard form components

The `@/components/dashboard/` directory contains shared form components that follow a consistent horizontal label / input layout used across edit and show pages:

| Component | Import from | When to use |
|---|---|---|
| `HorizontalInput` | `@/components/dashboard/horizontal-input` | Text, number, email, tel fields on form pages |
| `HorizontalSelect` | `@/components/dashboard/horizontal-select` | Native `<select>` dropdowns |
| `HorizontalDateInput` | `@/components/dashboard/horizontal-date-input` | Date fields (wraps `DateInput`) |
| `HorizontalTextArea` | `@/components/dashboard/horizontal-textarea` | Multi-line text fields |
| `HorizontalCheckbox` | `@/components/dashboard/horizontal-checkbox` | Boolean toggle fields |
| `SectionCard` | `@/components/dashboard/section-card` | Titled card with optional icon and action slot for show pages |
| `InfoRow` | `@/components/dashboard/info-row` | Read-only label / value pair inside `SectionCard` |
| `DashboardCard` | `@/components/dashboard/dashboard-card` | Coloured KPI card with optional action button |
| `CacheTooltip` | `@/components/dashboard/cache-tooltip` | Inline tooltip indicating cache duration |

Use these components instead of defining inline equivalents. The `Horizontal*` components include error display via `InputError`. The `ui/` primitives (`Input`, `Select`, `Label`) are lower-level building blocks — use dashboard components for standard form layouts.

### Breadcrumbs

Breadcrumbs are defined as a function or constant in each page and passed to the layout:

```tsx
const t = useI18n();

const breadcrumbs: BreadcrumbItem[] = [
    { title: t('Dashboard'), href: supplier.dashboard().url },
    { title: t('Items'), href: itemRoutes.index().url },
    { title: item ? `#${item.id}` : t('Details') },
];
```

The last breadcrumb item typically has no `href` (current page).

---

## 9) TypeScript conventions

### Type definitions

- Page-specific interfaces (`Props`, entity interfaces) live at the top of the page file
- Shared types live in `types/` and are re-exported from `types/index.ts`
- Use `interface` for object shapes, `type` for unions and aliases

### Naming

- Interfaces and types: PascalCase (`Booking`, `Props`, `BookingCustomer`)
- Variables and functions: camelCase (`pageTitle`, `getBreadcrumbs`)
- Constants: camelCase (not UPPER_SNAKE)
- Component names: PascalCase (`BookingIndex`, `SectionCard`)

### Strictness

- All props must be typed; `any` is discouraged
- Optional props use `?` only when the backend contract is genuinely optional (e.g. `item?: Item` on a create flow). Do not use `?` to hide missing required data — see `docs/07-anti-patterns.md` §28.
- Null safety: use nullish coalescing (`??`) and optional chaining (`?.`)

---

## 10) Translations and i18n

All user-facing text must originate from the **frontend i18n dictionary** that is **wired to Laravel `lang/*.json`**. The backend provides locale metadata and business data, not parallel ad hoc UI copy payloads on ViewModels (no `translations` maps for strings the React tree renders).

### The rule

Every string the user sees — page title, section heading, button label, column header, empty state message, placeholder, tooltip, confirmation text — must be resolved via the frontend translation function (`t()`).

### Key style (normative)

- Prefer **real English source strings** as lookup keys, e.g. `t('Customer first name')`, `t('Save changes')`, `t('No bookings yet')`.
- Do **not** use dotted slug keys such as `t('customer.first.name')` or `t('page.booking.index.title')` as the primary style — use readable English source strings as keys (see bullets above).
- Organize strings by **splitting Laravel lang into multiple JSON files** (e.g. by domain or feature) if needed — not by inventing artificial dot hierarchies for the same English phrase.

### Required pattern

Pages and shared components use **`t()`** and i18n, not backend `translations` props:

```tsx
const t = useI18n();

<Head title={t('Bookings')} />
<Button type="submit">{t('Save changes')}</Button>
```

### Laravel lang → frontend i18n

- **Authoritative source:** Laravel `lang/*.json` (e.g. `lang/sv.json` for default locale in this project). Add new user-visible strings there (or in additional `lang/**/*.json` files that your build merges).
- **Runtime dictionary:** Frontend modules under `resources/js/i18n/` (or the path your Vite pipeline loads) must stay **in sync** with those lang files — same keys, same default strings — via whatever import/sync/build step the project uses.
- Every string passed to `t('…')` in React must exist in that pipeline; missing keys are bugs to fix in `lang/`, not silent JSX fallbacks.

### Enum contract (backend labels)

- Backend Resources emit enum fields as **value + label pairs**:
  `'status' => $model->status->value`, `'status_label' => $model->status->label()`.
- Every domain enum defines `label(): string` wrapping `trans(...)` — see `stubs/core/domain/enums/enum.stub`. Enum labels are resolved on the backend, not via frontend `t()`.
- Select/filter option maps are built in ViewModels with `EnumMapper::keyValues(Enum::cases(), 'label')` (labrodev/php-enum-mapper).
- The frontend renders `*_label` for display and uses the raw `value` for logic/filters (TypeScript literal unions mirror enum values).
- Unknown enum values must show an explicit fallback (raw value), never silently fail.

### What this means in practice

- ViewModels must not pass `translations` (or equivalent) for Inertia page copy; pages use **`t()`** only for user-visible strings. Enum labels are the exception: they arrive pre-translated from the backend (`*_label`, `EnumMapper` option maps).
- If a key is missing, surface it visibly and fix the Laravel lang / i18n source.

---

## 11) Destructive actions

Destructive actions (delete, cancel) must follow the confirmation pattern defined in the playbook learning document:

- Use an AlertDialog (modal), not `window.confirm`
- Require the user to type the entity's unique business key to confirm
- Disable the confirm button until the typed value matches
- Submit via `router.delete()` to the Wayfinder route

---

## 12) Styling

All styling uses Tailwind CSS utility classes. See `docs/tailwindcss` (or the Tailwind skill) for version-specific guidance.

Rules:
- No inline `style` attributes unless absolutely necessary
- No CSS modules or separate CSS files per component
- Use `cn()` from `@/lib/utils` for conditional class merging
- Follow existing component patterns for spacing, colors, and borders
- Prefer existing UI components (`components/ui/*`) over custom implementations

---

## 13) Inertia v2 features

Inertia v2 provides advanced patterns beyond basic page rendering. These are available and should be used when appropriate. Always consult the Inertia v2 documentation via `search-docs` before implementing.

### Deferred props

Load non-critical data after the initial page render. The backend marks props as deferred; the frontend should display a skeleton or pulsing placeholder while they load.

### Polling

Automatically refresh page data at intervals. Useful for dashboards or status pages.

### Prefetching

Preload page data on hover or intent to navigate. Improves perceived performance.

### Merging props and infinite scroll

Append new data to existing page state (e.g. infinite scroll lists) using Inertia's merge behavior combined with `WhenVisible`.

### Once props

Props that are only sent on the first visit and not re-sent on subsequent partial reloads.

When using any of these features, follow the Inertia v2 documentation for the correct implementation. When using deferred props, always provide a skeleton/loading state.

---

## 14) Data exposure and security

Inertia serializes all page props into a JSON object embedded in the HTML `data-page` attribute. This means **every prop passed to an Inertia page is visible in the DOM** to anyone who inspects the page source. This is a fundamental security concern.

### The rule

Never pass raw Eloquent models, full database records, or unfiltered collections to Inertia pages. Every piece of data must pass through an explicit shaping layer — a Resource, a ViewModel, or a manually constructed array — that exposes only the fields the frontend needs.

### What must never appear in page props

- Internal database IDs when UUIDs are the public identifier (expose `uuid`, not `id`, unless `id` is explicitly needed for display)
- Password hashes, API tokens, secrets, or credentials
- Internal system fields (`created_by`, `updated_by`, internal flags) unless the page explicitly displays them
- Full related models loaded via eager loading that the page does not render
- Unfiltered `$model->toArray()` or `$model->attributesToArray()` output
- Pivot table data, internal metadata, or debug information
- Any field that exists in the database but has no corresponding UI element on the page

### How to enforce this

1. **Always use Resources** — Resources define an explicit allowlist of fields. The ViewModel passes models through Resources, never directly. This is already required by the stubs (`{Model}Resource::collection(...)` and `new {Model}Resource($this->model)->resolve()`).

2. **Review Resource output** — When adding a field to a Resource, ask: "Does the frontend need this? Does it display it?" If the answer is no, do not include it.

3. **Never pass `$model->toArray()` directly** — Even if it seems convenient, it exposes every attribute and loaded relation. Always go through a Resource.

4. **Audit eager-loaded relations** — If a ViewModel loads `$model->load(['relation'])`, the Resource for that relation must also filter its fields. Nested relations are a common source of leaks.

5. **Inspect the DOM** — During development, open the browser inspector and check the `data-page` attribute on the root element. Everything there is public. If you see data that should not be exposed, fix the Resource or ViewModel.

### ViewModel and Resource responsibility

The **ViewModel** decides *which* data to include (which model, which relations, which options).

The **Resource** decides *what fields* of that data to expose (the allowlist).

Together they form the security boundary between the database and the browser. Neither may be bypassed.

---

## 15) Anti-patterns (frontend)

### Hardcoded text and labels

Forbidden:
- Any user-facing string defined in `.tsx` files as the primary source (page titles, labels, buttons, column headers, placeholders, empty states, error messages, tooltips)
- Fallback strings (`?? 'Some text'`) shipped to production as a substitute for `t()`
- Swedish or English text written directly in JSX when `t()` is expected
- **Dotted slug keys** (`'customer.first.name'`, `'common.save'`) when a **readable English source string** would work as the key — use `t('Customer first name')` style per §10

If text is missing from i18n, fix the Laravel lang / frontend dictionary. Do not patch JSX with hardcoded strings.

### Faking, muting, and hiding backend problems

Forbidden:
- Hardcoding data, options, or logic in the frontend to make something "look like it works" when the backend does not yet provide it
- Hiding missing props behind fallback values or empty states instead of reporting the gap
- Implementing business rules, calculations, or conditional logic in React that belongs in Core (Actions, Services, Rules)
- Filtering, sorting, or transforming data in the frontend when it should be done in an IndexQuery, ViewModel, or Resource
- Catching errors silently and showing nothing instead of surfacing the problem

If the backend does not provide what the frontend needs, that is a **backend task**, not a frontend workaround. The frontend must make the gap visible (missing data, missing translation, missing prop) rather than hide it behind improvised code.

### Other forbidden patterns

- Fetching initial page data outside of Inertia props (no `fetch()`, `axios`, or SWR for page load data)
- Hand-written route URL strings (use Wayfinder)
- Hardcoded select option values in the frontend (use ViewModel-provided options)
- Business logic or authorization checks in frontend code
- Direct DOM manipulation
- Global mutable state outside of Inertia's page state
- Creating new UI primitives when an equivalent exists in `components/ui/`
- Using `window.confirm` for destructive actions
- Bypassing `@/lib/http` for JSON API calls (see section 16)

---

## 16) JSON API calls from Inertia pages (JsonControllers)

Some interactions require in-page reactivity without a full Inertia page visit: wizard-style flows, autocomplete/search, dynamic form fields, or real-time calculations. For these, the frontend makes JSON API calls to dedicated **JsonControllers** on the backend.

### When to use

- Multi-step flows where intermediate steps need server-side data (e.g. draft creation wizard)
- Search-as-you-type or autocomplete (e.g. finding external customers)
- Dynamic calculations that depend on server-side logic (e.g. matching engine time slots)
- Any interaction that needs a JSON response without replacing the current Inertia page

When the result is a full page navigation or form submission, use Inertia (`useForm`, `router.visit`, `<Link>`) instead.

### Backend: JsonControllers

JsonControllers live alongside regular Controllers in the layer structure:

```
App/Layer/<Layer>/{Domain}/
    Controllers/          ← Inertia controllers (return Inertia::render)
    JsonControllers/      ← JSON API controllers (return JsonResource)
    Exports/
    IndexQueries/
    ViewModels/
```

JsonControllers follow the same rules as regular controllers:
- Invokable (`__invoke()`)
- Single-responsibility (one endpoint per class)
- Thin (delegate to Core Actions, Services, or Orchestrators)
- Extend `BaseController` (or `SupplierController`) for `resolveSupplier()`
- **Authorize with class-level `#[Authorize(...)]`** (same as Inertia invokable controllers; see `docs/01-project-structure.md`)
- Return `JsonResource` or `AnonymousResourceCollection`, never raw arrays

**Naming:** `{Entity}{Verb}Controller` where verb describes the operation: `Get`, `Find`, `Calculate`, `Search`.

Examples:
- `ServiceGetController` — returns active services for a supplier
- `ExternalCustomerFindController` — searches external customers
- `ServiceTimeGetController` — calculates available time slots

### Backend: Routes

JsonController routes are nested under a `json/` prefix within the resource route group:

```php
Route::prefix('services')->name('services.')->group(function () {
    Route::get('/', ServiceIndexController::class)->name('index');
    Route::get('{service}', ServiceShowController::class)->name('show');
    Route::prefix('json')->name('json.')->group(function () {
        Route::get('get', ServiceGetController::class)->name('get');
    });
});
```

This keeps JSON endpoints:
- Under the same authentication and authorization middleware as the rest of the dashboard
- Within the `web` middleware group (session + CSRF)
- Discoverable by Wayfinder (so the frontend uses generated route functions)
- Namespaced to avoid collisions with Inertia routes

### Frontend: HTTP client (`@/lib/http`)

All JSON API calls from Inertia pages must use the helpers in `@/lib/http`. Direct use of `fetch()` or `axios` is forbidden.

The HTTP client handles CSRF and session authentication automatically:

```tsx
import { jsonPost } from '@/lib/http';

const response = await jsonPost(
    supplier.matchingEngine.serviceTime.get.url(),
    { service_uuid: selectedService, date: selectedDate }
);

if (!response.ok) {
    // handle error (see error handling below)
}

const data = await response.json();
```

### CSRF and session security

JSON API calls operate within the same Laravel session as the Inertia page. This is critical to understand:

**How it works:**

1. The supplier routes use the `web` middleware group (`['web', 'auth', 'verified', 'role:supplier']` in `bootstrap/app.php`)
2. Laravel's `web` group includes `VerifyCsrfToken` middleware and session handling
3. On every response, Laravel sets an `XSRF-TOKEN` cookie (encrypted, httpOnly=false so JS can read it)
4. `@/lib/http` reads this cookie, decodes it, and sends it back as `X-XSRF-TOKEN` header
5. Laravel validates the header against the session token

**Required headers for JSON calls:**

- `Accept: application/json` — tells Laravel to return JSON errors (not HTML redirects)
- `Content-Type: application/json` — for POST/PUT/DELETE with body
- `X-XSRF-TOKEN: {decoded cookie value}` — CSRF protection
- `credentials: 'include'` — ensures session cookies are sent with the request

All four are handled by `@/lib/http`. Do not implement CSRF handling manually elsewhere.

**Why 419 (CSRF mismatch) happens:**

- The XSRF-TOKEN cookie expired or was cleared (session timeout, browser cleanup)
- The cookie value was not URL-decoded before sending (the `%3D` padding issue)
- The `credentials: 'include'` flag was omitted (so cookies were not sent)
- The request was made from a context where the session cookie is not available

**Why 302 (redirect to login) happens:**

- The user's session expired while the Inertia page was still open
- The `Accept: application/json` header was missing (Laravel redirects HTML requests to login, but returns 401 JSON for API requests)

### Frontend: Error handling for JSON API calls

JSON API calls must handle session and CSRF errors gracefully. The user may have the page open for a long time, and the session can expire.

Required error handling pattern:

```tsx
async function callJsonApi<T>(url: string, body?: unknown): Promise<T> {
    const response = body !== undefined
        ? await jsonPost(url, body)
        : await jsonGet(url);

    if (response.status === 419) {
        // CSRF token expired — reload the page to get a fresh token
        window.location.reload();
        throw new Error('Session expired');
    }

    if (response.status === 401 || response.status === 403) {
        // Session expired or unauthorized — redirect to login
        window.location.href = '/login';
        throw new Error('Unauthorized');
    }

    if (!response.ok) {
        const error = await response.json().catch(() => ({}));
        throw new Error(error.message ?? `Request failed: ${response.status}`);
    }

    return response.json();
}
```

Key rules:
- **419** → reload the page (refreshes CSRF token and session state)
- **401/403** → redirect to login
- **422** → validation error, display errors from response body
- **500** → show a generic error message to the user
- Never silently swallow errors from JSON API calls

### What JsonControllers must NOT be used for

- Initial page data loading (use Inertia props / ViewModels)
- Form submissions that should navigate to another page (use `useForm` + Inertia)
- Replacing Inertia's built-in features (polling, deferred props, prefetching)
- Bypassing authorization (JsonControllers must authorize like any other controller)

---

## 17) Checklist for new pages

When creating a new Inertia page:

1. Backend ViewModel provides **all business data** and options; locale metadata is shared and UI text comes from frontend i18n (`t()` + Laravel `lang/`, see §10)
2. All page labels/titles/messages are resolved via `t('Readable English')` per §10 (no ViewModel `translations` or equivalent props for copy)
3. Page file placed at `pages/{inertia_render_path}.tsx`
4. Default-exported function component with typed `Props`
5. Wrapped in the correct layout with breadcrumbs
6. `<Head title={t('…')} />` — readable English key from `lang/`, not hardcoded JSX
7. **Zero hardcoded user-facing strings** in the component — all from `t()` / Laravel `lang/`
8. Routes via Wayfinder only (including JSON API endpoints)
9. Forms use `useForm` from `@inertiajs/react`
10. Index pages use `DataTable` with correct `idKey`
11. Validation errors shown inline with `InputError`
12. Flash messages shown with `FlashAlert`
13. No business logic, no frontend workarounds for missing backend functionality
14. No sensitive data in props — inspect `data-page` in the DOM to verify
15. JSON API calls (if any) use `@/lib/http` with proper error handling
16. `autoComplete="off"` on business data form inputs

---

## Closing note

The frontend is a rendering surface. It makes the ViewModel visible, resolves UI copy from frontend i18n, collects user input, and sends it back. It does not decide, calculate, authorize, or compensate for missing backend work.

If something is missing — a translation, a prop, an option list, a calculation — that is a backend task. The frontend must make the gap visible, not hide it.

When in doubt:
- data comes from the ViewModel
- **all text** comes from `t()` backed by Laravel `lang/` — zero hardcoded strings
- routes come from Wayfinder
- forms use `useForm`
- JSON API calls use `@/lib/http`
- structure follows existing pages
- if the backend doesn't provide it, fix the backend — don't fake it in React
