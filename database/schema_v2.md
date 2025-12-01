# LeanOps — Supabase Schema v2 (Draft)

*Lightweight MES schema for stations, jobs, assignments, performance tracking, users, and events.*

This version builds on **v1** and adds:
- User table (`app_users`)
- Event log (`events_log`)
- Extra timestamps and status fields for better analytics

---

## 1️⃣ Table: stations

Represents every workstation on the production floor.

| Column     | Type       | Notes                                          |
|-----------|------------|------------------------------------------------|
| id        | text (PK)  | Example: `"ST-01"`                             |
| name      | text       | Human-readable station name                    |
| line      | text       | Line name, e.g. `Line A`, `Line B`, etc.       |
| status    | text       | `running` / `idle` / `blocked`                 |
| is_active | boolean    | Whether station is in use                      |
| created_at| timestamptz| default: `now()`                               |
| updated_at| timestamptz| optional; track last status/name/line change   |

**Notes**

- `status` drives the supervisor view badges.
- `updated_at` can be maintained via trigger later if you want.

---

## 2️⃣ Table: jobs

Every job that exists in the system: past, present, or waiting in queue.

| Column       | Type        | Notes                                                 |
|-------------|-------------|-------------------------------------------------------|
| id          | text (PK)   | Example: `"JOB-1004"`                                 |
| part_name   | text        | Part / product name                                   |
| qty_total   | integer     | Total units required                                  |
| due_date    | date        | Due date                                              |
| priority    | text        | `normal` / `high` (can extend later)                  |
| status      | text        | `waiting` / `running` / `completed` / `cancelled`    |
| line_hint   | text        | Optional hint: which line this job *should* run on   |
| created_at  | timestamptz | default: `now()`                                      |
| completed_at| timestamptz | When job actually completed (nullable)                |

**Notes**

- `status` is what the Supervisor dashboard updates when you click **Mark complete**.
- `completed_at` will let you do lead-time + on-time performance later.

---

## 3️⃣ Table: station_assignments

Tracks which station is working on which job *right now* (or last assignment).

| Column      | Type        | Notes                                                       |
|------------|-------------|-------------------------------------------------------------|
| id         | uuid (PK)   | default `uuid_generate_v4()`                                |
| station_id | text (FK)   | → `stations.id`                                             |
| job_id     | text (FK)   | → `jobs.id`                                                 |
| qty_done   | integer     | Units completed so far on this station for this job         |
| is_active  | boolean     | `true` = currently assigned; `false` = finished / archived  |
| started_at | timestamptz | When the job started on this station                        |
| ended_at   | timestamptz | When this station finished this job (nullable)              |
| notes      | text        | Freeform notes (optional)                                   |

**Notes**

- The dashboard reads this to know each station’s `currentJob`.
- For v2, the Supervisor actions:
  - **Mark complete** can set `qty_done = qty_total`, `is_active = false`, `ended_at = now()` (future enhancement).
  - **Idle / Blocked** still lives on `stations`, but can also be logged in `events_log` for history.

---

## 4️⃣ Table: performance_logs

Daily performance snapshots by station and job (or just station).

| Column            | Type        | Notes                                             |
|------------------|-------------|---------------------------------------------------|
| id               | uuid (PK)   | default `uuid_generate_v4()`                      |
| station_id       | text (FK)   | → `stations.id`                                   |
| job_id           | text (FK)   | → `jobs.id` (nullable if station-only log)        |
| log_date         | date        | The date this log refers to                       |
| shift            | text        | e.g. `day` / `night`                              |
| mirrors_completed| integer     | Units completed in this period                    |
| target_units     | integer     | Planned units (for efficiency calc)               |
| efficiency       | numeric(5,2)| Optional pre-computed efficiency %                |
| created_at       | timestamptz | default: `now()`                                  |

**Notes**

- You can derive efficiency in SQL/BI, but keeping a stored value is handy for quick charts.
- `log_date + shift` lets you build your day vs night dashboards later.

---

## 5️⃣ Table: app_users

Application-level users who interact with dashboards (supervisors, operators, managers).

| Column        | Type        | Notes                                            |
|--------------|-------------|--------------------------------------------------|
| id           | uuid (PK)   | Should match Supabase Auth `auth.users.id`      |
| email        | text        | Unique email address                             |
| display_name | text        | Name shown in UI                                 |
| role         | text        | `supervisor` / `operator` / `manager` / `admin`  |
| is_active    | boolean     | default: `true`                                  |
| created_at   | timestamptz | default: `now()`                                 |
| last_login_at| timestamptz | Updated on login (optional)                      |

**Notes**

- v2 doesn’t *require* this table for the Supervisor dashboard, but it unlocks:
  - “Who changed this status?”
  - Per-user filters (e.g., show only stations on `Line A` for a specific supervisor).

---

## 6️⃣ Table: events_log

Append-only log of meaningful events (status changes, material calls, reassignments, etc.).

| Column     | Type        | Notes                                                         |
|-----------|-------------|---------------------------------------------------------------|
| id        | uuid (PK)   | default `uuid_generate_v4()`                                  |
| event_type| text        | e.g. `status_change`, `material_request`, `job_completed`     |
| station_id| text (FK)   | → `stations.id` (nullable)                                    |
| job_id    | text (FK)   | → `jobs.id` (nullable)                                        |
| user_id   | uuid (FK)   | → `app_users.id` (nullable, for who performed the action)     |
| payload   | jsonb       | Arbitrary details: old/new status, notes, quantities, etc.    |
| created_at| timestamptz | default: `now()`                                              |

**Notes**

- This becomes your **“activity feed”** for later:
  - “Show me everything that happened on ST-01 today.”
  - “Show all job reassignment events this week.”
- For now, you can log just key actions like:
  - Mark job complete  
  - Mark station blocked / idle  
  - Simulate material request

---

## 7️⃣ Relationships & High-Level Flow

- **stations**  
  - Core physical assets. Supervisor view reads `stations.status` to color badges.

- **jobs**  
  - Work to be done. `status` tracks its lifecycle; `priority` decides queue ordering.

- **station_assignments**  
  - Real-time mapping of which job is on which station, including partial completions via `qty_done`.

- **performance_logs**  
  - Historical rolls-ups for daily/shift KPIs (efficiency, throughput).

- **app_users**  
  - Who is using the system; important once you share dashboards and build permissions.

- **events_log**  
  - Time-stamped story of what’s happening across the floor.

---

## 8️⃣ v1 → v2 Summary

**What stayed the same**

- Core tables: `stations`, `jobs`, `station_assignments`, `performance_logs`.
- Basic columns (ids, names, qtys, due dates) remain compatible.

**What’s new / extended**

- Added timestamps (`updated_at`, `completed_at`, `started_at`, `ended_at`).
- Added `app_users` for future auth & accountability.
- Added `events_log` for an activity trail.
- Clarified allowed values for `status`, `priority`, and `shift`.

This schema is designed so your current Supervisor dashboard keeps working, while giving you room to:

- Add realtime logs,  
- Build job history and KPIs,  
- Expand into multi-user, multi-shift, multi-line analytics.

