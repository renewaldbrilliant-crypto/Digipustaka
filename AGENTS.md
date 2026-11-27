# AGENTS.md

## Project and specs
- Despite the repo name, the target app is the **temporary anonymous chat room** specified in `agent/prd.md`; UI rules are in `agent/design.md`. Read both before feature work. The root `README.md` is the stock Laravel readme, not project documentation.
- Follow `agent/task-instruction.md`: work through its 10 categories and update `.agents/tasklist.md` after each task (`[x] ✅`, overall progress, files changed). Create the tasklist from those categories if it is missing. The specs are in `agent/` (singular); keep them there — only the tasklist goes in `.agents/`.
- The PRD asks for Laravel 13/PHP 8.4, but the repo is **Laravel 9.52 on PHP 8.3**. Treat `composer.json` as authoritative and use Laravel 9 conventions (`app/Console/Kernel.php`, `app/Http/Kernel.php`, `$casts` property, `mix()`); don't start a framework upgrade or assume Laravel 11+ APIs unless the user asks.

## Setup and verification
- `.env` points at MySQL `Digipustaka` (127.0.0.1:3306, root, empty password), but the database does not exist in a fresh checkout. Create it before migrating:
  `mysql -u root -e "CREATE DATABASE IF NOT EXISTS Digipustaka CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci"`, then `php artisan migrate`.
- `composer install` (vendor is present), `npm install` (no `node_modules` yet), `npm run dev` or `npm run watch`, `php artisan serve`. There is no `package-lock.json`; use `npm install`, not `npm ci`.
- `php artisan test` runs both suites. Single file: `php artisan test tests/Feature/ExampleTest.php`; filter: `php artisan test --filter=ExampleTest`.
- `phpunit.xml` leaves the SQLite overrides commented out, so DB-backed tests use the real MySQL database from `.env` — a `RefreshDatabase` test can wipe the `Digipustaka` database. Enable the `:memory:` SQLite lines there before adding such tests, or use a separate test database.
- Room-expiry cleanup belongs in `app/Console/Kernel.php`; verify with `php artisan schedule:list` and run locally with `php artisan schedule:work`.

## Architecture gotchas
- There is no auth scaffolding. The PRD is session-based and CSRF-protected, so keep page and AJAX routes in the `web` middleware group (`routes/web.php`); do not build a token-auth SPA. The default axios setup sends the `XSRF-TOKEN` cookie.
- The default `users` table/model is Laravel auth (`email`, `password`). The PRD's participant `users` schema (`room_id`, `name`, `last_seen`, `typing_at`) conflicts with it: repurpose the existing table/model or use a separate table (e.g. `room_users`), never create a second `users` table. If repurposing, update the default factory/seeder too.
- Realtime is AJAX polling every 2 seconds, not websockets/broadcasting. Keep `GET /room/{code}/typing/status` read-only; only `POST .../typing` updates `typing_at`, after a true 300 ms debounce. Online means `last_seen` within 30 seconds.
- Frontend is Laravel Mix, not Vite: `resources/js/app.js` and `resources/css/app.css` compile via `webpack.mix.js` to `public/js` / `public/css`; use `mix()` in Blade, never `@vite`. Tailwind is not installed/configured; add it through Mix's PostCSS pipeline (the PRD requires it).
- `agent/design.md` fonts (JetBrains Mono, Inter) are not bundled; load them explicitly.

## UI and security constraints
- Follow `agent/design.md` strictly: `#161614` background, `#1D1D1B` surface, `#E5E4DF` text, `#7B7A74` muted, sole accent `#5FB5D6`; JetBrains Mono display/labels + Inter body; 3/6/10 px radii; 8/16/32 px spacing; no gradients; one accent action per screen. Keep the PRD's English UI copy ("Create Room", "joined the room", "Copy Link").
- The PRD's HTMLPurifier/Laravel Purifier is not installed. Add a sanitizer or sanitize input and escape output (Blade `{{ }}`, JS `textContent`, not `innerHTML`); enforce 30-char names, 500-char messages, 1 message/sec/user, and CSRF on all AJAX.
- App timezone is UTC (`config/app.php`); change it or convert explicitly if timestamps/countdowns should display in local time.
- Style: Laravel preset (`.styleci.yml`), 4-space indent, LF, UTF-8, final newline (`.editorconfig`).
