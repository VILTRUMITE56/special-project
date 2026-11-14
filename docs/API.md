# API Reference

Reviewed November 14, 2026. This document describes the routes currently mounted in [app.ts](../apps/api/src/app.ts). Planned routes remain in [SRS §6.2](SRS.md#62-software-and-api-interface); they are not available yet.

## Base URL and request handling

- Local API: `http://localhost:3001/api`.
- Development web requests: `/api`, forwarded by Vite to `API_PROXY_TARGET`.
- JSON request bodies: `Content-Type: application/json`; maximum size `100kb`.
- Success responses are resource objects, not wrapped in a `data` property.
- Schedule/task errors use `{ "error": "message" }`. Health has its own response shape.
- CORS enables browser access for the single configured `FRONTEND_ORIGIN` and accepts requests without an Origin header. It does not authenticate callers or prevent non-browser access.

## Temporary authentication and data

All schedule and task routes use the mock middleware in [mocks/auth.ts](../apps/api/src/mocks/auth.ts). Send one of these exact identities:

| Header | Mock user | Owned children |
| --- | --- | --- |
| `Authorization: Bearer 1` | Alice Caregiver (`caregiver`) | Child 1: Mia; child 2: Leo |
| `Authorization: Bearer 2` | Bob OT (`ot`) | Child 3: Ava |

A missing/malformed Bearer header returns 401 `Authentication required`; an unrecognized token returns 401 `Invalid or expired token`. These tokens are fixed mock IDs, with no JWT signature, password check or expiry. Registration, login and current-user endpoints are pending.

Each API process starts with these three children, a 12-activity library, and **no schedules or tasks**. All schedule/task writes are in memory and disappear on restart or watcher reload. The SQL seed script populates an independent set of demo records and does not populate this store. Resource ownership failures return 404, just like missing resources.

## Available routes

| Method | Path under `/api` | Success | Authentication |
| --- | --- | --- | --- |
| GET | `/health` | 200 health object | Public |
| GET | `/schedules?childId=1&date=2026-11-14` | 200 schedule with tasks | Mock Bearer |
| POST | `/schedules` | 201 schedule with tasks | Mock Bearer |
| PUT | `/schedules/:id` | 200 schedule with tasks | Mock Bearer |
| DELETE | `/schedules/:id` | 200 `{ "success": true }` | Mock Bearer |
| POST | `/tasks` | 201 task | Mock Bearer |
| PUT | `/tasks/:id` | 200 task | Mock Bearer |
| DELETE | `/tasks/:id` | 200 `{ "success": true }` | Mock Bearer |
| PATCH | `/tasks/:id/complete` | 200 task | Mock Bearer |

There is no standalone task-list or get-by-ID route. Read tasks through the schedule response. There is no child-list or activity-list endpoint yet.

## Health

`GET /api/health` runs `SELECT 1` through the MySQL pool for every request. HTTP 200:

```json
{
  "status": "ok",
  "database": "connected",
  "message": "API and database are available.",
  "timestamp": "2026-11-14T09:00:00.000Z"
}
```

If the database becomes unavailable, it returns HTTP 503:

```json
{
  "status": "error",
  "database": "disconnected",
  "message": "Database unavailable.",
  "timestamp": "2026-11-14T09:00:00.000Z"
}
```

This is a connectivity check, not a check that the tables exist. Normal server startup also requires a successful database connection before opening the listener.

## Schedules

### Read

`GET /api/schedules?childId=1&date=2026-11-14`

Both query parameters are required. `childId` is coerced to a positive integer; `date` must be a real calendar date in `YYYY-MM-DD` format. A missing schedule or a child outside the caller's ownership returns 404.

The response includes tasks sorted by ascending, one-based `position`:

```json
{
  "id": 1,
  "child_id": 1,
  "schedule_date": "2026-11-14",
  "created_at": "2026-11-14T09:00:00.000Z",
  "tasks": []
}
```

IDs and timestamps shown here are illustrative.

### Create and copy

`POST /api/schedules`

```json
{ "childId": 1, "date": "2026-11-14" }
```

`childId` must be a positive integer JSON number. The child must belong to the caller. There can be only one schedule per child/date; duplicates return 409.

To copy an existing schedule for the **same child**, supply an optional source date:

```json
{ "childId": 1, "date": "2026-11-15", "copyFrom": "2026-11-14" }
```

Copied tasks receive new IDs and retain their activity reference, name, icon, order, start time and duration. Their status resets to `pending` and `completed_at` to null. A missing source returns 404 and creates no target schedule.

### Change date

`PUT /api/schedules/:id`

```json
{ "date": "2026-11-16" }
```

Changes only the date; the child and tasks remain associated with the schedule. The same date is allowed; a date occupied by another schedule for that child returns 409.

### Delete

`DELETE /api/schedules/:id` removes the owned schedule and all its tasks from memory. Returns HTTP 200 with `{ "success": true }`.

## Tasks

### Add

`POST /api/tasks` accepts exactly one task source:

```json
{ "scheduleId": 1, "activityId": 2 }
```

or:

```json
{ "scheduleId": 1, "name": "Water the plants", "icon": "💧" }
```

- `scheduleId` and `activityId` must be positive integer JSON numbers.
- The schedule must belong to the caller; an unknown library activity returns 400.
- For a custom task, both `name` and `icon` are required. The name is trimmed and must contain 1–100 characters.
- `activityId` cannot be combined with `name` or `icon`.
- Custom icons must match [`PREDEFINED_ICONS`](../apps/api/src/mocks/data.ts) exactly.
- New tasks append to the schedule, with null time/duration, `pending` status and no completion timestamp. Set timing through PUT after creation.

The current mock activity IDs are:

| ID | Activity | ID | Activity |
| --- | --- | --- | --- |
| 1 | Wake up | 7 | Go to school |
| 2 | Brush teeth | 8 | Learning activity |
| 3 | Wash hands | 9 | Play |
| 4 | Eat breakfast | 10 | Bath |
| 5 | Get dressed | 11 | Get ready for bed |
| 6 | Toilet | 12 | Eat lunch |

These IDs refer to the mock library, not guaranteed SQL IDs. Example task response:

```json
{
  "id": 1,
  "schedule_id": 1,
  "activity_id": 2,
  "name": "Brush teeth",
  "icon": "🪥",
  "start_time": null,
  "duration_min": null,
  "position": 1,
  "status": "pending",
  "completed_at": null
}
```

Task responses do not currently include an activity category.

### Set timing and order

`PUT /api/tasks/:id`

```json
{ "start_time": "07:30", "duration_min": 10, "position": 1 }
```

At least one recognized update field is required. `start_time` must use zero-padded 24-hour `HH:MM`. `duration_min` must be an integer from 1 to 1440. Either may be null to clear it; omitted values are unchanged. `position` must be an integer from 1 to the current task count.

Moving a task shifts the other tasks and keeps all positions consecutive. Name, icon, activity reference, schedule and completion status are not editable through this route.

### Complete and undo

`PATCH /api/tasks/:id/complete`

```json
{ "done": true }
```

`done` must be a JSON boolean. `true` sets status to `done` and records an ISO UTC timestamp on the first completion. Repeating `true` preserves that timestamp. `false` returns the task to `pending` and clears `completed_at`.

### Delete

`DELETE /api/tasks/:id` removes the owned task and resequences remaining positions. Returns HTTP 200 with `{ "success": true }`.

## Validation and errors

Runtime schemas live in [shared/schemas.ts](../packages/shared/src/schemas.ts). Path IDs are coerced to positive integers. JSON body IDs must already be numbers. Zod object schemas strip unknown fields; use only the documented fields.

| Status | Current usage |
| --- | --- |
| 400 | Invalid input, invalid JSON, body over `100kb`, unknown activity, unsupported icon or out-of-range position |
| 401 | Missing or unknown mock credentials |
| 404 | Missing resource, inaccessible child/schedule/task, missing copy source, or unmatched route |
| 409 | Duplicate schedule for a child/date |
| 500 | Unexpected server failure; response hides internal error details |
| 503 | Health database check failure, using the health object above |

The error helper also supports 403, but current schedule/task handlers use 404 for ownership failures. Invalid requests return the first validation issue as `error`; clients should not depend on exact validation-library wording.

## PowerShell walkthrough

Start the API using the [README setup](../README.md#run-locally). Use a fresh API process or an unused date; repeating schedule creation for the same child/date returns 409. This walkthrough exercises the mock store, despite the startup database requirement.

```powershell
$apiBase = 'http://localhost:3001/api'
$mockHeaders = @{ Authorization = 'Bearer 1' }

Invoke-RestMethod "$apiBase/health"

$scheduleBody = @{ childId = 1; date = '2026-11-14' } | ConvertTo-Json
$schedule = Invoke-RestMethod "$apiBase/schedules" -Method Post -Headers $mockHeaders -ContentType 'application/json' -Body $scheduleBody

$taskBody = @{ scheduleId = $schedule.id; activityId = 2 } | ConvertTo-Json
$task = Invoke-RestMethod "$apiBase/tasks" -Method Post -Headers $mockHeaders -ContentType 'application/json' -Body $taskBody

$timingBody = @{ start_time = '07:30'; duration_min = 10 } | ConvertTo-Json
Invoke-RestMethod "$apiBase/tasks/$($task.id)" -Method Put -Headers $mockHeaders -ContentType 'application/json' -Body $timingBody

Invoke-RestMethod "$apiBase/tasks/$($task.id)/complete" -Method Patch -Headers $mockHeaders -ContentType 'application/json' -Body '{"done":true}'
Invoke-RestMethod "$apiBase/schedules?childId=1&date=2026-11-14" -Headers $mockHeaders
Invoke-RestMethod "$apiBase/tasks/$($task.id)/complete" -Method Patch -Headers $mockHeaders -ContentType 'application/json' -Body '{"done":false}'
```

Switching to `Bearer 2` for the final schedule read returns 404, because Mia belongs to mock user 1.

## Pending API areas

Authentication, child profile CRUD, activity listing, picture cards, progress summaries, settings and profile editing are not mounted. Their planned behavior and integration order are in the [SRS](SRS.md) and [backend task plan](BACKEND_TASKS.md). No JWT secret or token-issuing route exists yet.
