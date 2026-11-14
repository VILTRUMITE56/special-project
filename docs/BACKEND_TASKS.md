# Backend Implementation Plan

Updated November 14, 2026 after reviewing the source and running the existing checks. The target is the Express 5 + TypeScript JSON API under `/api` described in [SRS §6.2](SRS.md#62-software-and-api-interface), with authenticated, ownership-scoped MySQL persistence.

Frontend work (Staff/Child screens, TTS, drag-and-drop, timers and accessibility rendering) is outside this backend plan. API completion alone does not establish MVP acceptance.

## Current status and order

| Group | Status | Remaining dependency |
| --- | --- | --- |
| 1. Database foundation and API plumbing | Implemented in source | Live MySQL schema/seed verification remains unrecorded in this review |
| 2. Authentication and account | Pending | User data access, bcrypt, signed JWTs and real auth middleware |
| 3. Child profiles and activity library | Pending | Real authentication and ownership-scoped SQL queries |
| 4. Schedules and tasks | Route behavior implemented and tested with mocks | Replace both mock auth and in-memory storage; verify transactions and restart persistence |
| 5. Cards, progress, settings and hardening | Feature APIs pending; documentation updated | Real auth, child ownership and persistent schedule/task data |

Complete Groups **2 → 3 → Group 4 integration → 5** after verifying Group 1 against local MySQL. Group 4 landed ahead of Groups 2–3, so checked route tasks below indicate mock-backed behavior, not a finished persistent feature. Commit after each completed feature group (NFR-MAIN-03).

## Stack decisions

| Concern | Choice | State | SRS basis |
| --- | --- | --- | --- |
| Database driver | `mysql2/promise` pool; parameterize user-supplied values | Installed; used by health, schema and seeds | NFR-SEC-02 |
| Validation | `zod`; shared schemas in `packages/shared` | Installed; schedule/task schemas implemented | NFR-SEC-03 |
| Password hashing | `bcrypt` | Planned; dependency and implementation absent | FR-AUTH-03 |
| Sessions | `jsonwebtoken`, `JWT_SECRET`, 8-hour expiry | Planned; dependency, env configuration and implementation absent | FR-AUTH-04, NFR-SEC-01 |
| API tests | `vitest` + `supertest` | Installed; two mock-backed route test files | §9.2, NFR-SEC-06 |
| Config | `dotenv`, workspace `.env.example` files, ignored `.env` files | Implemented for current variables | NFR-SEC-05 |

See the [API reference](API.md) for current contracts and the [architecture guide](ARCHITECTURE.md) for the separate MySQL and mock data paths.

## Group 1 — Database foundation and API plumbing

- [x] **1.1 MySQL connection module** — Pool configured by `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`; required-value and port validation; startup connectivity check; public `GET /api/health` with 200/503 health objects; graceful shutdown. (NFR-SEC-02/05, NFR-REL-01)
- [x] **1.2 Schema script** — Seven §7.1 tables; unique user email, child/date and schedule/task-position keys; foreign-key cascades and nullable activity references; `npm run db:schema`. The database must already exist; `CREATE TABLE IF NOT EXISTS` is not a migration system. (§7.2)
- [x] **1.3 Seed script** — Eleven activities, ten default picture cards, a non-login demo caregiver, and Avery/Jordan/Riley with five tasks each for MySQL's current date; `npm run db:seed`. Re-runs reuse matching records but reset seeded task positions to pending and update their content/timing. This is repeatable demo setup, not preservation of demo progress. (FR-SCH-01, FR-COMM-02, §2.6)
- [x] **1.4 Error handling** — Central `{ "error": "message" }` feature errors; async wrapper; malformed/oversized JSON returns 400; unmatched routes return JSON 404. Health deliberately uses a separate typed response, including on 503.
- [x] **1.5 Validation and CORS** — Shared Zod schemas, parse-or-400 helpers and CORS response permissions for one configured frontend origin. Requests without an Origin header are accepted; CORS is not an authentication boundary. (NFR-SEC-03/04)

**Verification still required:** apply schema and seeds to a local development database, inspect keys/cascades and repeat-seed effects, confirm startup success/failure and health 200/503 behavior. Source review and passing route tests do not establish these checks.

## Group 2 — Authentication and account (AUTH)

- [ ] **2.1 User data access** — Parameterized insert/find-by-email/find-by-ID queries; keep hashes inside the authentication/data layer and out of API responses. Handle the null-hash SQL demo account without allowing password login. (§7.3, NFR-SEC-02)
- [ ] **2.2 Register endpoint** — `POST /api/auth/register`; validate name, email, password ≥8 characters and role; bcrypt hash; 409 on duplicate email. Add the dependency and shared auth schemas. (FR-AUTH-01/02/03)
- [ ] **2.3 Login endpoint** — `POST /api/auth/login`; verify credentials, sign an 8-hour JWT with a required secret, generic 401 for bad credentials. Add the dependency and `JWT_SECRET` configuration/template when the code consumes it. (FR-AUTH-04, NFR-SEC-01)
- [ ] **2.4 Auth middleware** — Verify Bearer JWTs; reject missing, invalid and expired tokens; attach the authenticated user. Replace the imports from `src/mocks/auth.ts` in schedule/task routes and apply real auth to every non-public route. Health remains public. (FR-AUTH-05/07)
- [ ] **2.5 Current-user endpoint** — `GET /api/auth/me` returning ID, name, email and role only. (FR-AUTH-05)
- [ ] **2.6 Auth tests** — Register/login/me flow; invalid input, duplicates, bad passwords, null-hash demo account, missing/invalid/expired tokens; adapt schedule/task fixtures to obtain real tokens. (§9.2)

**Group exit criteria:** registration → login → me works against MySQL; feature routes require signed JWTs. Logout is a frontend action that clears its stored token (FR-AUTH-06).

## Group 3 — Child profiles and activity library (CHILD)

- [ ] **3.1 Child data access** — Ownership-scoped SQL queries; every child read/write filters by authenticated `user_id`. (FR-CHILD-06, §7.3)
- [ ] **3.2 List and create** — `GET /api/children` and `POST /api/children`; name, age, delay domain, optional notes and predefined avatar. Confirm age validation bounds before publishing the contract. (FR-CHILD-01/02/05)
- [ ] **3.3 Read and update** — `GET, PUT /api/children/:id`; another user's child returns 404. (FR-CHILD-03/06)
- [ ] **3.4 Delete with cascade** — `DELETE /api/children/:id`; verify SQL cascades remove schedules, tasks and custom cards. The confirmation UI belongs to the frontend. (FR-CHILD-04)
- [ ] **3.5 Activity library endpoint** — `GET /api/activities` from MySQL; reconcile the 11 SQL seed activities with the 12 mock activities and their differing icons. Consolidate the predefined icon source. (FR-SCH-01)
- [ ] **3.6 Isolation tests** — User B cannot list, read, edit or delete user A's children; exercise actual database cascades. (NFR-SEC-06)

**Group exit criteria:** child CRUD and activity listing use MySQL, with ownership and cascade behavior verified.

## Group 4 — Schedules and tasks (SCH + TASK)

Current routes import `src/mocks/auth.ts` and `src/mocks/data.ts`. Fixed mock users own fixed mock children; schedules/tasks begin empty and are lost on restart. Group 1's MySQL pool is present, but these handlers do not use it.

- [x] **4.1 Schedule read (mock)** — `GET /api/schedules?childId=&date=`; tasks ordered by position; ownership through the child. (FR-SCH-02, FR-TASK-05)
- [x] **4.2 Schedule create and copy (mock)** — `POST /api/schedules`; 409 on duplicate child/date; optional `copyFrom` for the same child; copied tasks reset to pending with no completion time. (FR-SCH-02/09)
- [x] **4.3 Schedule update and delete (mock)** — `PUT, DELETE /api/schedules/:id`; change date with conflict detection or remove the schedule and tasks; ownership checked.
- [x] **4.4 Task add and edit (mock)** — `POST /api/tasks` for a library activity or custom name + predefined icon; `PUT /api/tasks/:id` for nullable `start_time`/`duration_min` and one-based position reordering. (FR-SCH-04/05/08)
- [x] **4.5 Task remove (mock)** — `DELETE /api/tasks/:id`; ownership checked and remaining positions resequenced. (FR-SCH-06)
- [x] **4.6 Complete and undo (mock)** — `PATCH /api/tasks/:id/complete` with `done`; first completion records a timestamp, repeating it preserves the timestamp, undo clears it. (FR-TASK-01–04)
- [x] **4.7 Mock route tests** — Create/copy/add/reorder/time/complete/undo/delete flows, invalid input, conflicts and cross-user access; 39 passing tests across two files. In-process re-reads are covered; restart persistence is not. (NFR-SEC-06)
- [ ] **4.8 MySQL data access and transactions** — Replace in-memory operations with ownership-scoped parameterized queries. Make copy, reorder and deletion atomic; respect unique `(schedule_id, position)` during moves and resequencing. Preserve response shapes and completion semantics. (NFR-REL-01)
- [ ] **4.9 Real-auth integration** — Use Group 2 middleware and Group 3 child/activity records; replace mock-specific test setup with database-backed users, children and tokens. Review signatures rather than assuming synchronous mock functions can remain unchanged.
- [ ] **4.10 Restart and database tests** — Replace `it.todo('state survives a server restart')` with an integration check that creates data, restarts the API and verifies order, timing and completion; verify ownership, conflicts and transaction rollback against MySQL. (NFR-REL-01, NFR-SEC-06)

**Group exit criteria:** the full lifecycle uses MySQL and real authentication, with persistence, ordering, timestamps and ownership verified. Checked mock route tasks alone do not meet this criterion.

## Group 5 — Cards, progress, settings and hardening

- [ ] **5.1 Card listing** — `GET /api/cards?childId=`; shared defaults plus custom cards for an owned child, grouped by category. (FR-COMM-01/02)
- [ ] **5.2 Custom card management** — `POST /api/cards` with child reference, label, category and predefined icon; `DELETE /api/cards/:id` for owned custom cards. Decide and implement hide semantics and any schema change; there is currently no visibility field. (FR-COMM-06/07)
- [ ] **5.3 Progress endpoint** — `GET /api/progress/:childId?range=7d`; today's done/total/percentage, seven-day daily completion and recent completion times. Define timezone, empty-day and zero-task behavior. (FR-PROG-01–03)
- [ ] **5.4 Settings and profile** — Persist `font_size`, `contrast`, `sound_on` and own profile name; confirm proposed `GET, PUT /api/settings` and `PUT /api/auth/me` contracts before implementation. (FR-SET-01/05)
- [ ] **5.5 Full API test suite** — Extend Vitest/Supertest across implemented §6.2 endpoints, including health, real auth, database isolation, validation and ownership; isolated test database setup/teardown. (§9.2, NFR-SEC-06)
- [x] **5.6 Current documentation pass** — README setup/configuration/commands, SRS implementation status, backend task status, API reference and architecture reflect reviewed source. Existing environment templates match variables currently consumed. Revisit when real auth and feature APIs land. (NFR-MAIN-02)
- [ ] **5.7 Backend acceptance support** — Verify API capabilities needed for SRS §9.1 items 1–6, record database-backed results, and hand off to frontend integration. Child screens, audio, phrase building and dashboard presentation require browser checks.

**Group exit criteria:** all agreed backend contracts are implemented, persisted and tested; documentation matches them. End-to-end MVP acceptance remains a separate application-level check.

## Verification record

On November 14, 2026:

- `npm run build`: passed.
- `npm run typecheck`: passed for all three workspaces.
- `npm run test -w @starter/api`: two files passed; **39 passed, one todo**.
- Live MySQL schema/seed execution, health failure handling and restart persistence: not verified in this review.
- Browser flows, TTS, responsiveness, accessibility, performance and usability: not verified; product UI is pending.

The API suite imports `app.ts` directly, resets the mock store, and does not start `server.ts` or exercise the MySQL connection. A passing suite therefore does not establish Group 1's live database checks or Group 4's persistence exit criteria.

## Frontend responsibilities

TTS and sentence bar (FR-COMM-03/04/05/08), countdown timers and sound cues (FR-TASK-06/07), Child view and exit control (FR-CHD-*), drag-and-drop and Add controls (FR-SCH-03), accessibility rendering (FR-SET-02/03/04), login/logout navigation and friendly error states require frontend implementation. The backend must provide authenticated, persistent data to support them.
