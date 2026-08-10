# Labrodev Inertia + React

Always-on law. The frontend (React 19, Inertia v2, TypeScript strict, Tailwind v4, Wayfinder, Vite) is a **delivery surface**, not a business layer: it renders data shaped by ViewModels and Resources, resolves UI copy from i18n, collects user input, and sends it back — it never decides, calculates, authorizes, or compensates for missing backend work. Anatomy and templates → the `labrodev-inertia-react` skill; per-file checks → `rules/frontend.md`.

## Musts

- **One Inertia page = one file** at `resources/js/pages/{render-path}.tsx`, where `{render-path}` is exactly the kebab-case string passed to `Inertia::render(...)` (`'bookings/index'` → `pages/bookings/index.tsx`).
- **Every page is a default-exported PascalCase function component** named `{Resource}{Action}`, with an explicit `interface Props` at the top of the file matching the ViewModel output exactly.
- **Every page** wraps its content in exactly one layout (`AppLayout` for dashboard, `AuthLayout` variants for auth, `SettingsLayout` for settings), passes `breadcrumbs`, and renders `<Head title={t('…')} />`.
- **All routes come from Wayfinder** — generated functions under `@/routes/` (named routes) and `@/actions/` (controller actions); navigate with `<Link>` and `router.visit()`/`router.delete()` from `@inertiajs/react`.
- **All forms use `useForm`** from `@inertiajs/react`, posting to the per-action routes (store/update/remove) via Wayfinder; field names are snake_case and match the backend Data class attributes (→ labrodev-data).
- **All user-facing text goes through `t()`** backed by Laravel `lang/*.json`; keys are readable English source strings (`t('Save changes')`), never dotted slugs (`t('common.save')`).
- **Enum display/typing follows the enum contract**: render `*_label`, use the raw value for logic (→ labrodev-enum).
- **In-page JSON calls** (autocomplete, wizards, live calculations) go only through the `@/lib/http` helpers, targeting JsonController routes via Wayfinder, with explicit 419/401/403/422 handling.
- **Destructive actions** use an AlertDialog confirmation requiring the user to type the entity's unique business key, then `router.delete()` to the Wayfinder route.
- **Index rows are identified by uuid** in URLs — routes bind `{model:uuid}`, so `viewHref` and form targets receive `item.uuid`, never a database id (→ labrodev-controller).

## Must-nots

- No hardcoded user-facing strings in `.tsx` — no JSX literals, no shipped `?? 'Fallback text'` substitutes for `t()`; a missing key is a `lang/` bug to fix, not something to patch in React.
- No `translations` prop (or any map of UI strings) in page `Props` — ViewModels do not ship page copy; `t()` does.
- No fetching of initial page data (`fetch()`, `axios`, SWR, `useEffect` loaders) — initial data comes exclusively from Inertia props.
- No hand-written URL strings, no `<a href="/...">` for internal links — Wayfinder only.
- No hardcoded select option values — options come from ViewModel props.
- No business rules, calculations, filtering, sorting, or authorization checks in React — that belongs in Core Actions/Rules, IndexQueries, ViewModels, or Policies on the backend.
- No `window.confirm` for destructive actions.
- No raw `fetch()`/`axios` for JSON calls — `@/lib/http` handles CSRF, session cookies, and headers.
- No editing files under `@/routes/` or `@/actions/` — they are Wayfinder-generated.
- No new UI primitives when an equivalent exists in `components/ui/`; no inline `style` attributes or per-component CSS files — Tailwind utilities with `cn()` from `@/lib/utils`.
- No faking or muting backend gaps — never hardcode data to "look like it works", never silently swallow errors, never hide a missing prop behind an empty state; make the gap visible and fix it on the backend.
