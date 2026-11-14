# Visual Schedule, Task Management and Communication System

A web application being developed to help caregivers, occupational therapists and SPED teachers organize children's daily routines and support picture-based communication. This repository is an npm workspace using TypeScript, React 19, Vite 7, Express 5, MySQL and shared Zod schemas. Workspace names retain the `@starter/*` prefix.

## Current implementation

Reviewed against the code on November 14, 2026. The application is a partial implementation of the [software requirements](docs/SRS.md).

| Area | Current behavior |
| --- | --- |
| Web app | Starter page that requests and displays API health; Staff and Child interfaces are pending |
| Database foundation | MySQL pool, startup connectivity check, seven-table schema and demo seed script |
| Health | Public `GET /api/health`, including a live database connectivity check |
| Schedules and tasks | Create, read, copy, date changes, reordering, timing, completion, undo and deletion using an in-memory store |
| Authentication | Temporary `Bearer 1` / `Bearer 2` identities for schedule/task routes; registration, login and JWT verification are pending |
| Remaining features | Child CRUD, activity listing, cards, progress and settings endpoints are pending |

Schedule/task changes are lost when the API restarts or its development watcher reloads. These routes do not read the SQL seed data. Mock identities are for local development; real authentication and database-backed feature routes are required before live use.

## Documentation

- [SRS](docs/SRS.md): intended product requirements, acceptance criteria and implementation status.
- [Backend tasks](docs/BACKEND_TASKS.md): completed foundations, mock-backed work and remaining integration tasks.
- [API reference](docs/API.md): routes available today, request validation, responses and a PowerShell walkthrough.
- [Architecture](docs/ARCHITECTURE.md): workspace layout, runtime flow, database behavior and integration limits.

## Run locally

Prerequisites: Node.js **22.12.0 or newer**, npm, and a running MySQL server. Run the following commands from the repository root. On PowerShell systems that block `npm.ps1`, use `npm.cmd` in place of `npm`.

1. Install the locked workspace dependencies:

   ```sh
   npm ci
   ```

2. Create the database using a MySQL client or database administration tool. For example, open `mysql -u root -p` and run:

   ```sql
   CREATE DATABASE IF NOT EXISTS special_project
     CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   ```

3. Copy the API environment template. In PowerShell:

   ```powershell
   Copy-Item apps/api/.env.example apps/api/.env
   ```

   Edit `apps/api/.env` to match your MySQL credentials and database name. An existing `.env` should be edited rather than overwritten.

4. Apply the schema, optionally add SQL demo data, then start both apps:

   ```sh
   npm run db:schema
   npm run db:seed
   npm run dev
   ```

Open <http://localhost:5173>. The API listens on <http://localhost:3001>. It checks MySQL before listening; it will exit if the connection fails. Vite forwards `/api` requests to the API during development.

The schema command creates tables in an existing database; it does not create the database or migrate existing tables. Seeding is optional for the current mock routes. It adds 11 activities, 10 shared picture cards, and Avery, Jordan and Riley with five tasks each for MySQL's current date. The SQL demo caregiver (`demo@special-project.invalid`) has a null password hash and no login flow. Re-running the seed resets completion for its demo task positions; use a development database. See [database behavior](docs/ARCHITECTURE.md#database-foundation).

## Environment variables

The API and database scripts load `apps/api/.env` when invoked through the workspace scripts.

| API variable | Template value / default | Purpose |
| --- | --- | --- |
| `PORT` | `3001` | Listener port; defaults to 3001 when unset; must be an integer from 1 to 65535 |
| `DB_HOST` | `localhost` | Required MySQL host |
| `DB_PORT` | `3306` | Required MySQL port; integer from 1 to 65535 |
| `DB_USER` | `root` | Required MySQL username |
| `DB_PASSWORD` | Empty | Must be defined; an empty value is accepted |
| `DB_NAME` | `special_project` | Required existing database name |
| `FRONTEND_ORIGIN` | `http://localhost:5173` | Single allowed browser origin; also the fallback when empty or unset |

Copy `apps/web/.env.example` to `apps/web/.env` only when customizing the web configuration:

| Web variable | Default | Purpose |
| --- | --- | --- |
| `API_PROXY_TARGET` | `http://localhost:3001` | Development proxy target; match the API port if it changes |
| `VITE_API_BASE_URL` | Empty | Public API origin used by the browser client; empty uses the page's origin |

Restart development processes after changing environment values. `VITE_API_BASE_URL` is embedded at build time. Keep credentials in the API environment; `VITE_` variables are public. `JWT_SECRET` is not consumed by the current implementation.

## Commands and verification

| Command | Purpose |
| --- | --- |
| `npm run dev` | Run API and web watchers; the API's `predev` builds the shared package |
| `npm run typecheck` | Type-check all workspaces |
| `npm run build` | Build shared package, API and web app in order |
| `npm run test -w @starter/api` | Build shared package, then run the Vitest/Supertest route suite |
| `npm run db:schema` | Apply the seven `CREATE TABLE IF NOT EXISTS` statements |
| `npm run db:seed` | Seed SQL library, cards and demo schedules |
| `npm start` | Start the compiled API after building; does not serve the web app |
| `npm run preview -w @starter/web` | Preview the built web app; has no development API proxy |
| `npm run hooks:install` | Install the optional Conventional Commits message hook |

There is no root `npm test` script. The API suite imports the Express app directly and tests in-memory schedule/task behavior without MySQL or starting `server.ts`. The current result is **39 passing tests and one todo** for restart persistence. It does not verify SQL schema/seed execution, JWT authentication, health connectivity or browser flows.

Validation on November 14, 2026: build, workspace type-checks and the API suite passed. Live MySQL setup and browser acceptance checks were not performed in this review.

## Development conventions

Add routes under `apps/api/src/routes`, frontend features under `apps/web/src`, and shared API contracts and validation schemas under `packages/shared/src`. See the [architecture guide](docs/ARCHITECTURE.md#workspace-layout) for the complete layout. Shared runtime exports come from `packages/shared/dist`, so rebuild that package after schema changes; the root build and API predev/pretest scripts do so automatically.

The optional commit-message hook accepts `type(scope): description` or `type: description`, with a subject of at most 72 characters. Example: `docs: align project documentation with current implementation`. The installer writes to `.git/hooks` and is invoked explicitly, not by `npm ci`.

## Deployment plan

After real authentication and MySQL persistence are integrated, build with `npm run build`, serve `apps/web/dist` from a static host, and run the API with `npm start`. The shared workspace package and its compiled output must be available to the API runtime. Supply backend environment variables and apply the schema to the target database separately.

For a shared origin, configure the host to forward `/api` to the API. For separate origins, set `VITE_API_BASE_URL` before building the web app and set `FRONTEND_ORIGIN` to that web origin. Use HTTPS for live deployment. No hosting configuration or automated deployment pipeline is included yet.

## Troubleshooting

- **API exits before listening:** confirm MySQL is running, the database exists, and every required `DB_*` value is set. `DB_PASSWORD=` is valid; a missing variable is not.
- **Frontend says the API is unavailable:** check the API process and `GET /api/health`, then confirm the proxy target or built API base URL. A lost database connection produces HTTP 503 from health.
- **Preview cannot reach `/api`:** the development proxy is unavailable in preview. Use a reverse proxy or rebuild with an explicit API base URL and configure the matching CORS origin.
- **SQL demo children do not appear in schedule requests:** schedule routes currently use Mia, Leo and Ava from the independent mock store. See [mock identities](docs/API.md#temporary-authentication-and-data).
- **Runtime cannot import shared schemas:** run `npm run build -w @starter/shared` before starting the API directly.
