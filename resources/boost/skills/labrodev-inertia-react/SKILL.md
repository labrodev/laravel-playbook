---
name: labrodev-inertia-react
description: "Use when creating or reviewing React/Inertia frontend code in a Labrodev Laravel project — Inertia page components under resources/js/pages, typed Props interfaces, useForm form submissions, Wayfinder route usage, t()/lang/*.json i18n wiring, flash/toast consumption, destructive-action confirmation dialogs, or in-page JSON API calls via @/lib/http."
license: MIT
metadata:
  author: labrodev
---

# Inertia + React Frontend

Part of the Labrodev playbook. **The law for this component lives in the always-on `labrodev-inertia-react` guideline** (musts, must-nots); the per-file checklist is `rules/frontend.md`. This skill holds the craft: anatomy, canonical templates, and edge cases.

The frontend is a **delivery surface**, not a business layer. It renders data shaped by ViewModels and Resources, resolves UI copy from i18n, collects user input, and sends it back. It never decides, calculates, authorizes, or compensates for missing backend work. Backend prop assembly (ViewModels, Resources, field allowlisting) → see the labrodev-viewmodel-resource skill.

Stack: React 19, Inertia.js v2 (React adapter), TypeScript (strict), Tailwind CSS v4, Wayfinder (generated route functions), Vite.

## Directory structure

```
resources/js/
    pages/          — Inertia page components (one file per route, kebab-case path = render string)
    layouts/        — AppLayout, AuthLayout variants, SettingsLayout
    components/     — shared reusable components
    components/ui/  — primitives (button, input, card, …)
    components/dashboard/ — composed dashboard components (DataTable, HorizontalInput, SectionCard, …)
    hooks/          — custom React hooks (use-i18n, …)
    types/          — shared TypeScript types (SharedData, BreadcrumbItem, PaginatedData, …)
    lib/            — utilities (cn(), http.ts)
@/routes/, @/actions/ — Wayfinder-generated; never edit manually
```

A helper component used by only one domain's pages may be co-located inside that domain's `pages/` subfolder (e.g. `pages/bookings/status-badge.tsx`) — it must not be default-exported (default export is reserved for page components). Once it is used by a second domain, move it to `components/`.

## Index page template

Naming pattern: file `pages/{resource-plural}/index.tsx`, component `{Resource}Index`. Worked example for a Booking domain:

```tsx
import { Head } from '@inertiajs/react';
import AppLayout from '@/layouts/app-layout';
import { DataTable } from '@/components/dashboard/data-table';
import { useI18n } from '@/hooks/use-i18n';
import dashboard from '@/routes/dashboard';
import * as bookingRoutes from '@/routes/bookings';
import type { BreadcrumbItem, PaginatedData } from '@/types';

// Literal union mirroring the backend enum's backing values (BookingStatus)
type BookingStatusValue = 1 | 2 | 3;

interface Booking {
    uuid: string;
    reference: string;
    status: BookingStatusValue;
    status_label: string; // pre-translated on the backend
    booked_at: string;
}

interface Props {
    bookings: PaginatedData<Booking>;
    statusOptions: Array<{ value: BookingStatusValue; label: string }>;
}

export default function BookingIndex({ bookings, statusOptions }: Props) {
    const t = useI18n();

    const breadcrumbs: BreadcrumbItem[] = [
        { title: t('Dashboard'), href: dashboard.index().url },
        { title: t('Bookings') },
    ];

    return (
        <AppLayout breadcrumbs={breadcrumbs}>
            <Head title={t('Bookings')} />
            <DataTable
                columns={[
                    { key: 'reference', label: t('Reference'), sortable: true },
                    {
                        key: 'status',
                        label: t('Status'),
                        filterable: true,
                        filterType: 'select',
                        filterOptions: statusOptions,
                        render: (row: Booking) => row.status_label ?? String(row.status),
                    },
                    { key: 'booked_at', label: t('Booked at'), sortable: true },
                ]}
                data={bookings}
                baseUrl={bookingRoutes.index().url}
                viewHref={(row: Booking) => bookingRoutes.show(row.uuid).url}
                idKey="uuid"
            />
        </AppLayout>
    );
}
```

Notes on the template:

- `Props` mirrors the ViewModel `toArray()` output exactly. Which fields exist there is decided on the backend → see the labrodev-viewmodel-resource skill.
- An unknown enum value renders an explicit fallback (the raw value), never nothing.
- `idKey="uuid"` because show/update routes bind `{booking:uuid}`.

## Form template (create + edit in one component)

Naming pattern: file `pages/{resource-plural}/form.tsx`, component `{Resource}Form`. Field names are snake_case and mirror the Data class attributes.

```tsx
import { Head } from '@inertiajs/react';
import { useForm } from '@inertiajs/react';
import AppLayout from '@/layouts/app-layout';
import FlashAlert from '@/components/flash-alert';
import { HorizontalInput } from '@/components/dashboard/horizontal-input';
import { HorizontalSelect } from '@/components/dashboard/horizontal-select';
import { Button } from '@/components/ui/button';
import { useI18n } from '@/hooks/use-i18n';
import * as bookingRoutes from '@/routes/bookings';

interface Props {
    booking?: { uuid: string; reference: string; status: number } | null;
    statusOptions: Array<{ value: number; label: string }>;
}

export default function BookingForm({ booking, statusOptions }: Props) {
    const t = useI18n();
    const isEdit = Boolean(booking?.uuid);

    const { data, setData, post, put, processing, errors } = useForm({
        reference: booking?.reference ?? '',
        status: booking?.status ?? statusOptions[0]?.value ?? '',
    });

    const onSubmit = (e: React.FormEvent) => {
        e.preventDefault();
        if (isEdit && booking?.uuid) {
            put(bookingRoutes.update(booking.uuid).url, { preserveScroll: true });
        } else {
            post(bookingRoutes.store().url);
        }
    };

    return (
        <AppLayout breadcrumbs={[]}>
            <Head title={isEdit ? t('Edit booking') : t('Create booking')} />
            <FlashAlert />
            <form onSubmit={onSubmit}>
                <HorizontalInput
                    label={t('Reference')}
                    value={data.reference}
                    onChange={(v: string) => setData('reference', v)}
                    error={errors.reference}
                    autoComplete="off"
                />
                <HorizontalSelect
                    label={t('Status')}
                    value={data.status}
                    options={statusOptions}
                    onChange={(v: string) => setData('status', Number(v))}
                    error={errors.status}
                />
                <Button type="submit" disabled={processing}>
                    {t('Save changes')}
                </Button>
            </form>
        </AppLayout>
    );
}
```

Form rules recap: initial values come from props (pre-filled for edit, empty defaults for create); validation errors from `errors` display inline (the `Horizontal*` dashboard components embed `InputError`; use `InputError` directly with `ui/` primitives); `processing` disables submit; `preserveScroll: true` on edit forms; `autoComplete="off"` on business-data inputs (not on auth forms). The backend controller flashes a toast with `Inertia::flash('toast', ...)` and redirects with `to_route()` → see the labrodev-controller skill.

## TypeScript conventions

- `interface` for object shapes, `type` for unions and aliases. Page-specific interfaces live at the top of the page file; shared types in `types/` re-exported from `types/index.ts`.
- PascalCase for interfaces, types, and components; camelCase for variables, functions, and constants (no UPPER_SNAKE).
- `any` is discouraged; prefer `unknown` plus narrowing. Use `??` and `?.` for null safety.
- Optional props (`?`) only when the backend contract is genuinely optional (e.g. `booking?: Booking | null` on a shared create/edit form). Never use `?` to hide missing required data.
- Shared per-page data (`auth`, `flash`, `sidebarOpen`) is typed as `SharedData` in `types/index.ts` and accessed via `usePage<SharedData>()` only when needed.

## i18n contract

- Authoritative source: Laravel `lang/*.json`. The runtime frontend dictionary (modules the Vite pipeline loads) must stay in sync — same keys, same source strings.
- Every string passed to `t('…')` must exist in that pipeline. A missing key is a backend/asset-pipeline bug; surface it visibly, never invent a JSX fallback.
- Keys are readable English source strings. Split `lang/` into multiple JSON files by domain if volume demands — do not invent dot hierarchies.
- The single exception to "all copy via `t()`": **enum labels**, which follow the enum contract → see the labrodev-enum skill.

## Flash / toast consumption

The backend flashes with `Inertia::flash('toast', ...)` before `to_route()` (→ see the labrodev-controller skill). On the frontend:

- Flash data arrives in shared props (`usePage<SharedData>().props.flash`).
- Form and mutation pages render `FlashAlert` (`@/components/flash-alert`) to display success/error flashes; the app shell's toast listener consumes the `toast` flash for transient notifications.
- Never re-implement flash display ad hoc per page; use the shared components.

## Destructive-action confirmation

Delete/cancel actions must:

1. Open an AlertDialog (modal from `components/ui/`), never `window.confirm`.
2. Require the user to type the entity's unique business key (e.g. the booking reference) into an input.
3. Keep the confirm button disabled until the typed value matches exactly.
4. On confirm, submit `router.delete(bookingRoutes.remove(booking.uuid).url)` (Wayfinder route; the backend exposes deletion as POST/DELETE per its route conventions → see the labrodev-controller skill).

## JSON API calls from pages

For in-page reactivity (autocomplete, multi-step wizards, server-side calculations) call JsonController endpoints. Backend anatomy, naming, and the `json/` route prefix → see the labrodev-controller skill. Frontend rules:

- Use `jsonGet` / `jsonPost` from `@/lib/http` only — it sends `Accept: application/json`, `Content-Type: application/json`, the decoded `X-XSRF-TOKEN` header, and `credentials: 'include'`. Never hand-roll CSRF handling.
- Target the endpoint via its Wayfinder function.
- Required status handling: **419** → `window.location.reload()` (refresh CSRF token); **401/403** → redirect to login; **422** → display validation errors from the response body; **500/other** → generic user-visible error. Never silently swallow a failed JSON call.

```tsx
import { jsonPost } from '@/lib/http';
import * as bookingSlotRoutes from '@/routes/bookings/json';

const response = await jsonPost(bookingSlotRoutes.slots().url, {
    service_uuid: selectedService,
    date: selectedDate,
});
if (response.status === 419) { window.location.reload(); return; }
if (response.status === 401 || response.status === 403) { window.location.href = '/login'; return; }
if (!response.ok) { /* surface the error to the user */ }
const slots = await response.json();
```

Never use JSON calls for initial page data (Inertia props), for form submissions that navigate (useForm), or to replace Inertia v2 built-ins (polling, deferred props, prefetching).

## Edge cases

- **Inertia v2 features** (deferred props, polling, prefetching, merge/infinite scroll with `WhenVisible`, once props) are allowed; consult the Inertia v2 docs before implementing. Deferred props always get a skeleton/loading placeholder.
- **Data exposure**: everything in page props is serialized into the DOM `data-page` attribute and is public. During development, inspect it — if you see internal ids, secrets, or fields the page does not render, the fix is in the Resource/ViewModel allowlist → see the labrodev-viewmodel-resource skill.
- **Show pages** compose `SectionCard` + `InfoRow` from `@/components/dashboard/` for read-only label/value layouts; KPI tiles use `DashboardCard`.
- **Breadcrumbs**: array of `BreadcrumbItem` per page; the last item carries no `href` (current page); titles via `t()`.
- **Missing backend data** (prop, option list, translation, calculation): render a visible gap or error, file it as a backend task. Do not fill it with frontend logic.
