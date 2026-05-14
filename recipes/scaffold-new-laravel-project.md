# Recipe: Scaffold new Laravel project (Sail, Inertia React, Fortify, Spatie, Horizon, quality tools)

Greenfield baseline: **PHP 8.5** and **Laravel 13** only—do not scaffold on older PHP or Laravel majors. Stack includes **Laravel Sail** (PostgreSQL + Redis), **Inertia + React** via the official starter path, **Fortify**, selected **Spatie** packages, **Horizon**, and **Pint + Larastan** on PHP plus **ESLint + Prettier** on the React app. Within Laravel 13, exact `artisan` flags and installer UX can still shift—confirm details against current Laravel, Sail, Fortify, and starter-kit documentation.

**If the directory is empty:** create the application using Phase 2 (or `composer create-project laravel/laravel:^13.0 .` with PHP 8.5 active) so `composer.json` exists and requires **`laravel/framework` ^13.0** and **`php` ^8.5**, then run Phase 1 (Sail) and the remaining phases.

Attach:

- `docs/09-tooling.md`
- `docs/12-workflow.md`
- `docs/11-inertia-react.md` (after the UI shell exists)

## Prerequisites

- **PHP 8.5** on the host (or in Sail’s runtime image after you publish or customize Sail for 8.5—keep app and CI on 8.5).
- **Composer** and **Node.js** compatible with Laravel 13 and the chosen starter kit.
- **Docker** (or Docker Desktop) for Sail.
- A directory for the new application and write access there.
- After any install step, verify `composer.json` has `"php": "^8.5"` and `"laravel/framework": "^13.0"` (or equivalent `13.x` constraint).

## Phase 1 — Sail + PostgreSQL + Redis

1. Create or enter a Laravel application directory (see Phase 2 if you prefer creating the app and Sail together in one flow).
2. Require Sail if not already present: `composer require laravel/sail --dev`.
3. Run `php artisan sail:install` and select **PostgreSQL** and **Redis** (and any other services you need). This produces the Sail `docker-compose.yml` and stubs. Ensure the Sail PHP image matches **PHP 8.5** (publish/customize Sail if the default lags—this recipe assumes 8.5 everywhere).
4. Copy `.env.example` to `.env` and set at least:
   - `DB_CONNECTION=pgsql`
   - Database host/user/password/database aligned with Sail defaults (typically `pgsql` as `DB_HOST` when using Sail from inside containers; use `127.0.0.1` / published ports when running Artisan on the host without Sail—pick one model and stay consistent).
   - Redis host/port for Sail (`redis` hostname inside the compose network).
   - `QUEUE_CONNECTION=redis` so queues and Horizon use Redis.
5. Start the stack: `./vendor/bin/sail up -d` (or `sail up -d` if the shell alias is configured).
6. Verify PostgreSQL and Redis are reachable from the app container (e.g. `sail artisan migrate:status` or `sail redis-cli ping`).

## Phase 2 — Laravel + Inertia + React starter

1. Prefer the **official** path for **Laravel 13** + **PHP 8.5**, for example:
   - The Laravel **React** application / starter kit (see Laravel 13 documentation and the `laravel/react-starter-kit` GitHub repository), or
   - `laravel new` / installer options that target Laravel 13, PHP 8.5, Inertia + React + Vite when available.
2. After installation, confirm `composer.json` enforces PHP 8.5 and Laravel 13, and the tree includes **Inertia Laravel**, **Inertia React**, **Vite**, and a `resources/js` React entry.
3. Run `npm install` and `npm run build` (or `npm run dev`) once to confirm the frontend toolchain works.

## Phase 3 — Laravel Fortify

1. `composer require laravel/fortify`
2. If the starter kit **already** ships Fortify, align versions with `composer update` and skip duplicate publishes; otherwise run `php artisan fortify:install` and follow current Fortify docs for providers, routes, and views (headless Fortify is common with Inertia).
3. Configure Fortify features (registration, reset password, email verification, 2FA, etc.) per product needs.

## Phase 4 — Spatie packages

Install with Composer (use releases compatible with **Laravel 13**):

```bash
composer require spatie/laravel-data
composer require spatie/laravel-view-models
composer require spatie/laravel-permission
composer require spatie/laravel-query-builder
```

Notes:

- Package name is **`spatie/laravel-permission`** (singular), not `laravel-permissions`.
- Publish and migrate per package docs—**permission** ships migrations you must run (`sail artisan migrate` after publish).
- `laravel-data`, `laravel-view-models`, and `laravel-query-builder` may only need publish when you use config or stubs; follow each package’s installation page.

## Phase 5 — Laravel Horizon

1. `composer require laravel/horizon`
2. `php artisan horizon:install`
3. Ensure `QUEUE_CONNECTION=redis` and Redis credentials match Sail.
4. Register Horizon’s service provider if **Laravel 13** does not auto-discover it (see `bootstrap/providers.php` or application service providers for your skeleton).
5. Locally: `./vendor/bin/sail artisan horizon` (or queue worker) after `sail up`. Secure the Horizon dashboard in non-local environments (middleware / gate per Laravel docs).

## Phase 6 — Quality tooling (PHP + React)

**PHP (Laravel backend)**

- `composer require --dev laravel/pint larastan/larastan`
- Add or align `phpstan.neon` (paths, Larastan extension) and run `./vendor/bin/phpstan analyse` when the baseline is ready. See `docs/09-tooling.md` for project expectations around Pint and static analysis.

**React / TypeScript frontend**

- “Laravel + React” here means **backend + frontend** quality: use the **ESLint** and **Prettier** setup shipped or recommended by the starter kit (`package.json` scripts such as `lint`, `format`, or `lint:fix` if present).
- Do not duplicate config from scratch if the starter already provides it—extend only when needed.

## Post-install sanity

- `sail artisan migrate` (or `php artisan migrate` if not using Sail for Artisan).
- `npm install && npm run build`.
- Hit the app in the browser; confirm Fortify flows you enabled; confirm Horizon shows queues when jobs are dispatched (local only).
- Run Pint and PHPStan on a clean tree before adding domain code.

## Non-goals

This recipe does **not** establish the Labrodev **`src/`** layout, playbook **stubs**, or project-specific Composer packages (e.g. Wayfinder, Nightwatch, internal `labrodev/*` packages). Add those in a separate task or follow-up recipe once the vanilla scaffold is stable.
