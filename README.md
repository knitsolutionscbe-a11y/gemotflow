# OT Flow — Backend API

Node.js + Express + PostgreSQL backend for the OT Flow patient-tracking app
(Ward → Pre-Op → Theatre → Recovery → Ward).

This has been built and smoke-tested end-to-end (login, full patient pipeline,
department permissions, cancel-case, reports, audit log) against a real
PostgreSQL 16 instance.

## 1. Requirements

- Node.js 18+
- PostgreSQL 14+

## 2. Setup

```bash
npm install
```

Create a database and user:

```sql
CREATE USER otflow WITH PASSWORD 'choose-a-strong-password';
CREATE DATABASE otflow OWNER otflow;
```

Apply the schema and the feature migrations, in order:

```bash
psql -h localhost -U otflow -d otflow -f src/schema.sql
psql -h localhost -U otflow -d otflow -f src/migration_002.sql
psql -h localhost -U otflow -d otflow -f src/migration_003.sql
```

Copy `.env.example` to `.env` and fill in your real values:

```bash
cp .env.example .env
```

Seed the default Super Admin account and starter masters (wards, 7 theatres,
7 pre-op bays, 2 recovery beds, default anaesthesia types):

```bash
node src/seed.js
```

This creates username `superadmin` / password `admin123` (or whatever you set
`SEED_SUPERADMIN_PASSWORD` to in `.env`). **Change this password immediately
after first login** — there's no UI restriction stopping you, just use the
"Edit user" flow once the frontend is wired up, or `PUT /api/users/:id`.

## 3. Run

```bash
node src/server.js
```

Server listens on `PORT` (default 4000). Check it's alive:

```bash
curl http://localhost:4000/health
```

For production, run this behind a process manager (pm2, systemd) and a
reverse proxy (nginx) with HTTPS termination — the API itself does not
serve TLS.

## 4. Authentication

`POST /api/auth/login` with `{ "username": "...", "password": "..." }`
returns `{ token, user }`. Send the token on every other request as:

```
Authorization: Bearer <token>
```

Tokens expire after 12 hours (`JWT_SECRET` in `.env` — change this to a long
random value before deploying anywhere real).

## 5. Permission model

Mirrors the original frontend exactly:

- **Roles**: `superadmin`, `admin`, `consultant`, `manager`, `user`
- **Departments**: `all`, `ward`, `preop`, `ot`, `recovery`, `otstore`
- Super Admin and Admin bypass department checks entirely — they can act
  anywhere, always, regardless of what their `department` field says.
- **Resource scoping (new):** a `ward` or `ot` department user can additionally
  be locked to one specific ward/theatre via `assigned_ward_id`/`assigned_ot_id`
  on their user record. Leaving it blank makes them a "manager" with access to
  every ward/theatre in that department — this is enforced on the actual
  patient-pipeline endpoints (admit, send/receive, log-step, move-recovery),
  not just in the UI.
- Other departments (Pre-Op, Recovery, OT Store) remain department-wide, not
  resource-scoped.
- Only `superadmin` can cancel a case or reverse/correct a patient's stage.
- Only `admin`/`consultant`/`superadmin` can upload/import an OT booking
  list or clear it.
- Only `superadmin`/`admin` can manage users, wards, theatres, beds,
  pre-op bays, masters, the item catalog, and surgery packages.
- Only `superadmin` can view or edit the HMS billing integration config.

## 6. API reference (summary)

All routes below are prefixed with `/api` and require `Authorization: Bearer <token>`
except `/api/auth/login`.

| Method | Path | Notes |
|---|---|---|
| POST | `/auth/login` | Public |
| GET | `/auth/me` | Current user |
| GET/POST/PUT/DELETE | `/users` | Super Admin / Admin; body accepts `assignedOtId`/`assignedWardId` |
| GET/POST/PUT/DELETE | `/wards` | + `/wards/:wardId/rooms` |
| GET/POST/PUT/DELETE | `/preop-bays` | |
| GET/POST/PUT/DELETE | `/ots` | + `PATCH /ots/:id/mark-ready` |
| GET/POST/PUT/DELETE | `/beds` | |
| GET | `/masters` | All 4 simple master lists |
| POST | `/masters/:type` | type = anesthesiaTypes / anesthetists / drugs / otItems |
| DELETE | `/masters/:type/:value` | |
| GET/POST/PUT/DELETE | `/item-catalog` | Drugs/OT items/implants master, matches S No/Item Code/Description/UOM/Item Group/Item Type |
| POST | `/item-catalog/bulk` | CSV-style bulk import, `{rows:[...]}`, skips existing codes |
| GET/POST/PUT/DELETE | `/surgery-packages` | Bundles of catalog items with quantities |
| POST | `/surgery-packages/bulk` | Bulk import `{rows:[{packageName,itemCode,qty}]}`, groups by package name |
| GET | `/patients` | `?status=` `?q=` filters; each patient includes `itemOrders` |
| GET | `/patients/:id` | |
| POST | `/patients` | Admit — ward dept, ward-scoped if assigned |
| POST | `/patients/emergency-admit` | Pre-op dept; admits straight to Pre-Op bypassing ward |
| POST | `/patients/:id/fast-track-preop` | Pre-op dept; moves an existing ward patient straight to Pre-Op |
| POST | `/patients/:id/send-preop` | ward dept |
| POST | `/patients/:id/receive-preop` | preop dept |
| POST | `/patients/:id/send-ot` | preop dept |
| POST | `/patients/:id/receive-ot` | ot dept, OT-scoped |
| POST | `/patients/:id/log-step` | `{step, time}`, ot dept, OT-scoped |
| PATCH | `/patients/:id/case-items` | drugs / OT items, ot dept, OT-scoped |
| POST | `/patients/:id/move-recovery` | ot dept, OT-scoped |
| PATCH | `/patients/:id/recovery-notes` | recovery dept |
| POST | `/patients/:id/vitals/preop` | `{bp, spo2}`, preop dept |
| POST | `/patients/:id/vitals/recovery` | `{bp, spo2}`, recovery dept |
| POST | `/patients/:id/send-ward` | recovery dept |
| POST | `/patients/:id/receive-ward` | ward dept, ward-scoped if assigned |
| POST | `/patients/:id/cancel-case` | Super Admin only, `{reason}` |
| POST | `/patients/:id/reverse` | Super Admin only, `{action, reason, bedId?}` |
| POST | `/patients/:id/item-orders` | Order a catalog item, ot dept |
| POST | `/patients/:id/item-orders/from-package` | Add every item in a surgery package at once |
| PATCH | `/patients/:id/item-orders/:orderId/issue` | otstore dept |
| PATCH | `/patients/:id/item-orders/:orderId/receive` | ot dept — confirms physical receipt from store |
| PATCH | `/patients/:id/item-orders/:orderId/use` | ot dept |
| PATCH | `/patients/:id/item-orders/:orderId/return` | ot dept — marks unused, not billed |
| PATCH | `/patients/:id/item-orders/:orderId/acknowledge-return` | otstore dept — confirms the unused item has been physically handed back |
| POST | `/patients/:id/item-orders/submit-billing` | Sends "Used" items to the configured HMS endpoint |
| POST | `/patients/:id/item-orders/retry-billing` | Resets failed items to Used and resubmits |
| GET | `/ot-store/pending` | Cross-patient queue of items still "ordered", for store staff |
| GET | `/ot-store/returns` | Items marked "returned" (unused) but not yet acknowledged as physically back in the store |
| GET/PUT | `/integration` | Super Admin only — HMS billing endpoint/key config |
| POST | `/integration/test` | Sends a test payload to the configured HMS endpoint |
| GET | `/bookings` | |
| POST | `/bookings/parse-pdf` | multipart `file`, returns preview rows only |
| POST | `/bookings/import` | commits reviewed rows, optional immediate admit |
| DELETE | `/bookings` | Clear whole list |
| GET | `/audit-log` | Super Admin / Admin |
| GET | `/reports?from=&to=&workingHours=` | Turnaround + utilization |

## 7. What's NOT built yet

- The frontend (the HTML app) still talks to client-side storage, not this
  API. Wiring it up to call these endpoints instead is the next step.
- No rate limiting, no HTTPS (add a reverse proxy), no automated DB backups —
  add these before this holds real patient data.
- Password reset flow doesn't exist; an admin resets a password directly.
- Storing the HMS API key in the database is not a secrets vault — for a real
  production billing integration, consider a proper secrets manager.
