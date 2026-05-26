# Palm-Oil Field → Management Platform

> A multi-tenant estate-management platform that replaces a legacy PHP/OpenCart admin panel with a modern **Next.js + Go + MSSQL** stack — covering harvesting, grading, attendance, work orders, inspections, weighbridge, weather, and 17 operational reports across multiple palm-oil estates.

---

## Table of Contents

1. [Business Problem Solved](#1-business-problem-solved)
2. [Stack](#2-stack)
3. [Architecture Overview](#3-architecture-overview)
4. [Repository Structure](#4-repository-structure)
5. [Screenshots](#5-screenshots)
6. [Deployment Approach](#6-deployment-approach)
7. [Notes & Status](#7-notes--status)

---

## 1. Business Problem Solved

Palm-oil estates run on tightly coupled daily workflows — **harvesting, grading, gang attendance, field inspections, work-order payroll, weighbridge intake, and weather logging** — that all roll up to a manager dashboard and a stack of statutory reports.

The original system ("Estate-Admin") was a **fork of OpenCart 4.x** in PHP/Twig, repurposed so that the e-commerce surface is inert and the real value lives in custom modules (`field/`, `master/`, `collection/`, `upkeep/`, `work_order/`, `mill/`, `attendance/`, `audit/`, `asset/`, plus `extension/irga/` dashboards and reports). It was difficult to maintain, hard to extend, and tied to legacy frontend libraries (jQuery, Bootstrap 5, Flot, dropzone, gmaps, etc.).

**What this platform delivers:**

- **Multi-tenant by estate** — one estate is chosen at login; every subsequent query, dashboard, and report is scoped to that estate's MSSQL database (e.g. `ESTATE A`, `ESTATE B`).
- **Operational coverage** for the six primary personas: Estate Manager, Field Supervisor, Gang Leader (mobile companion), Mill Operator, HR/Payroll Clerk, and System Admin.
- **Full domain modeling** of harvesting (LPG cooperative vs. TPS collection-station variants), grading, attendance + muster chits, work orders (daily-rate / piece-rate / overtime / harvesting), field inspections (per-palm), weighbridge tickets, weather logs, and a complete master-data catalog (estates, divisions, blocks, block tasks, platforms, rows, employees, gangs, vehicles, equipment, products, UoM, job descriptions, attendance rates, etc.).
- **Reports** — 17 print-friendly reports (harvest by block / employee / platform, muster chit, monthly pocket checkroll, attendance, field-inspection by inspector, harvest-round month, reception-block month, upkeep gang attendance, WO daily-rate / piece-rate / overtime by employee or job description, etc.) exported via the browser's *Save as PDF*.
- **Read-only weighbridge integration** — weighbridge tickets are owned by the third-party scale software and surfaced as a read-only view.
- **Map view** — Leaflet-based aggregator with grouped GPS pins (one marker per `(lat, lng)`) and detail drill-down per pin.
- **Mobile companion sync target** — the Go API exposes a `/api/v1/mobile/*` namespace for the field-side mobile app (gang leaders log harvesting from tablets).

The result: the same business workflows the legacy panel served, on a stack that's faster to develop against, easier to deploy, and friendly to both desktop operators and tablet-based field users.

---

## 2. Stack

### Frontend — `iplants-next/`
| Layer | Choice |
| --- | --- |
| Framework | **Next.js 13** (App Router) |
| Language | **TypeScript 5.2** |
| Styling | **Tailwind CSS 3** + **shadcn/ui** (Radix primitives) |
| Forms | **React Hook Form** + **Zod** validation |
| State | **React Context API** |
| Charts | **Recharts** |
| Maps | **Leaflet** |
| Auth (client) | Cookie-based session, JWT bearer to API |
| Themes | **next-themes** (light / dark, system-aware) |
| Icons | **lucide-react** |

### Backend — `iplant-go-api/`
| Layer | Choice |
| --- | --- |
| Language | **Go 1.26** |
| HTTP | `net/http` + `http.ServeMux` (stdlib) |
| gRPC | `google.golang.org/grpc` (attendance service) |
| DB driver | `github.com/microsoft/go-mssqldb` |
| Auth | **JWT** (`golang-jwt/jwt/v5`) + bcrypt (`golang.org/x/crypto`) |
| Config | `github.com/joho/godotenv` |
| Storage | Local file store for uploads (random 16-byte URLs) |

### Database
- **Microsoft SQL Server** (unchanged from the legacy system to preserve operational continuity).
- Three schemas inside `FIELD_PLUS_V1`:
  - `dbo.*` — operational data (harvesting, inspections, gradings, attendance, audit, weighbridge, weather, …)
  - `General.*` — masters (Estate, Division, BlockMaster, block_task, block_task_platform, block_task_row)
  - `Checkroll.*` — labor (CREmployee, GangMaster, GangEmployeeSetup)
- Per-estate database connection registry on the Go side (in-memory `estatereg.Registry`).

### Legacy reference — `Estate-Admin/`
Kept in-tree as the spec source: PHP / OpenCart 4.x / Twig / jQuery / Bootstrap 5, ~200 controllers, ~120 models, ~440 Twig templates, plus the 522-file `extension/irga/` dashboards & reports module.

---

## 3. Architecture Overview

```
                                  ┌────────────────────────────────────┐
                                  │      Operators & Managers          │
                                  │  (desktop browsers, tablets)       │
                                  └──────────────┬─────────────────────┘
                                                 │ HTTPS
                                                 ▼
                              ┌──────────────────────────────────┐
                              │      Next.js 13 (App Router)     │
                              │  iplants-next  — TS, Tailwind,   │
                              │  shadcn/ui, Recharts, Leaflet    │
                              │                                  │
                              │  • Login + estate selection      │
                              │  • Dashboard KPIs & charts       │
                              │  • CRUD on master/operational    │
                              │  • 17 printable reports          │
                              │  • Map aggregator                │
                              └──────────────┬───────────────────┘
                                             │ JWT (Authorization: Bearer …)
                                             ▼
              ┌────────────────────────────────────────────────────────────┐
              │                    iplant-go-api  (Go)                     │
              │                                                            │
              │  HTTP :8080                       gRPC :9090               │
              │  ───────────                       ───────────              │
              │  /api/v1/dashboard/*               attendance.*             │
              │  /api/v1/mobile/*                                           │
              │                                                            │
              │  Middleware:  RequireJWT → RequireEstate → handler         │
              │                                                            │
              │  Domains: auth, dashboard aggregates, attendances,         │
              │           work-orders, weather, inspections, weighbridge,  │
              │           map, history, settings, uploads, users,          │
              │           master data (estates/divisions/blocks/employees/ │
              │           gangs/vehicles/platforms/...), reports (17)      │
              └──────────────┬─────────────────────────────────────────────┘
                             │ database/sql + go-mssqldb
                             ▼
              ┌────────────────────────────────────────────────────────────┐
              │            Microsoft SQL Server  (FIELD_PLUS_V1)           │
              │   dbo.*          General.*          Checkroll.*            │
              │   operational    masters            labor / payroll        │
              └──────────────┬─────────────────────────────────────────────┘
                             │  per-estate DB / config
                             ▼
              ┌────────────────────────────────────────────────────────────┐
              │     Estate A db   │   Estate B db   │   Estate N db …      │
              └────────────────────────────────────────────────────────────┘

  (sync target)                                      (legacy reference)
  Mobile field app   ───── /api/v1/mobile ──┐        Estate-Admin (PHP/OpenCart)
  (gang leaders)                            │        kept in-tree as PRD source
                                            ▼
                                  iplant-go-api
```

### Request lifecycle (authenticated dashboard request)

1. Browser sends `Authorization: Bearer <JWT>` with the request.
2. `RequireJWT` middleware validates signature, expiry, and extracts claims (user id, estate code).
3. `RequireEstate` middleware looks up the estate's DB pool via the in-memory `estatereg.Registry` and injects an `*estatereg.Entry` into the request context.
4. The handler reads `*sql.DB` from the entry, executes scoped SQL, and returns JSON.
5. Reports use the same path but the frontend renders the JSON into a print-friendly HTML page; operators export via *Save as PDF*.

### Multi-tenancy model

- One JWT carries an estate claim.
- One `estatereg.Entry` per estate holds the `*sql.DB` pool + metadata.
- Login picks the DB from the request body's `estateCode`; every other route gets it from the JWT.
- The `/api/v1/dashboard/estates` endpoint is intentionally unauthenticated so the login dropdown can render before any token exists.

---

## 4. Repository Structure

The working monorepo lives at `~/Documents/GitHub/IRGA/WORKING-DASHBOARD/` and contains:

```
WORKING-DASHBOARD/
├── PRD.md                     # ~830-line product spec (mapping legacy → new)
├── .prd-notes/                # controller / view / model inventories
├── iplants-next/              # ✅ NEW frontend (Next.js 13 App Router)
│   ├── app/
│   │   ├── login/
│   │   ├── dashboard/
│   │   ├── attendances/
│   │   ├── work-orders/       (daily-rate / piece-rate / overtime / harvesting)
│   │   ├── field-inspections/
│   │   ├── master/            (25+ master pages: estates, divisions, blocks,
│   │   │                       block-tasks, block-task-rows, block-task-platforms,
│   │   │                       employees, gangs, gang-employees, vehicles,
│   │   │                       equipment, products, platforms, station,
│   │   │                       harvest-round, work-types, work-order-activities,
│   │   │                       job-categories, job-descriptions, employee-roles,
│   │   │                       attendance-rates, unit-of-measurements, …)
│   │   ├── map/
│   │   ├── reports/
│   │   ├── system/
│   │   ├── weather-logs/
│   │   └── weightbridge/
│   ├── components/
│   │   ├── auth/  dashboard/  forms/  inspection/  layout/
│   │   ├── map/   master/     providers/  reports/  ui/  work-orders/
│   ├── contexts/   hooks/   lib/   types/
│   └── middleware.ts          # route protection
│
├── iplant-go-api/             # ✅ NEW backend (Go 1.26)
│   ├── cmd/
│   │   ├── api/               # HTTP + gRPC entrypoint
│   │   └── schema/            # schema inspection tool
│   ├── api/attendance/        # gRPC proto contracts
│   ├── internal/
│   │   ├── auth/              # JWT + bcrypt
│   │   ├── config/            # env loading
│   │   ├── db/  dbutil/       # MSSQL pool + helpers
│   │   ├── estatereg/         # per-estate DB registry
│   │   ├── httpapi/           # HTTP router, middleware, v1 handlers
│   │   ├── grpcapi/  grpcserver/   # gRPC services (attendance)
│   │   ├── dashboard/  reports/    # aggregate queries
│   │   ├── attendance/  workorder/ inspection/  weighbridge/
│   │   ├── weather/    history/   mapdata/    irgauser/
│   │   ├── master/     settings/  uploads/
│   └── migrations/            # master/lookup table SQL
│
└── Estate-Admin/              # 📜 LEGACY reference (PHP/OpenCart 4.x)
    ├── spanel/version 3/      # controller / model / view (Twig) / language
    ├── extension/version 3/   # irga/, opencart/, dev/, modification/{lpg,tps}/
    └── project/v3/            # bootstrap, per-estate configs, database.config
```

### Key surface-area counts

- **Go HTTP routes:** ~80 (auth + dashboard + master + reports + uploads + settings)
- **Master-data CRUD modules:** 25+
- **Reports:** 17 print-friendly endpoints
- **Frontend pages:** 30+ (App Router segments)
- **Legacy PHP controllers replicated against:** ~200

---

## 5. Screenshots

> Drop screenshots into `./screenshots/` as `1.jpeg`, `2.jpeg`, … — the references below resolve automatically.

### Login & Estate Selection
![Login](./screenshots/1.jpeg)
![Estate selection](./screenshots/2.jpeg)

### Operational Dashboard
KPI cards, harvest charts (monthly / yearly / by block), quality bars, weather chart.

![Dashboard overview](./screenshots/3.jpeg)
![Harvest charts](./screenshots/4.jpeg)
![Quality & weather widgets](./screenshots/5.jpeg)

### Attendances
Per-employee flattened view (one row per `attendance_list` entry inside a muster chit), filterable by gang / employee / date range.

![Attendance list](./screenshots/6.jpeg)
![Attendance detail](./screenshots/7.jpeg)

### Work Orders
Daily-rate, piece-rate, overtime, and harvesting flows.

![Work orders index](./screenshots/8.jpeg)
![Daily rate](./screenshots/9.jpeg)
![Piece rate](./screenshots/10.jpeg)
![Overtime](./screenshots/11.jpeg)
![Harvesting](./screenshots/12.jpeg)

### Field Inspections
Per-palm inspections with image uploads.

![Inspections list](./screenshots/13.jpeg)
![Inspection detail](./screenshots/14.jpeg)

### Map View
Leaflet aggregator — one pin per `(lat, lng)` with detail drill-down.

![Map markers](./screenshots/15.jpeg)
![Pin detail](./screenshots/16.jpeg)

### Master Data
Estates, divisions, blocks, block tasks (+ rows / platforms), employees, gangs, vehicles, equipment, products, job descriptions, attendance rates, and more.

![Master estates](./screenshots/17.jpeg)
![Master blocks](./screenshots/18.jpeg)
![Master employees / gangs](./screenshots/19.jpeg)
![Master block-task hierarchy](./screenshots/20.jpeg)

### Reports
17 print-friendly reports rendered from JSON, exported via the browser's *Save as PDF*.

![Reports dropdown](./screenshots/21.jpeg)
![Harvest by block](./screenshots/22.jpeg)
![Muster chit](./screenshots/23.jpeg)
![Monthly pocket checkroll](./screenshots/24.jpeg)

### Weather & Weighbridge
![Weather log](./screenshots/25.jpeg)
![Weighbridge tickets](./screenshots/26.jpeg)

### System / Users
![User management](./screenshots/27.jpeg)

---

## 6. Deployment Approach

### Frontend — `iplants-next`

Configured for **static export** (`next.config.js → output: 'export'`, `images: { unoptimized: true }`), so the build emits a fully static bundle that can be hosted on any static host or CDN.

```bash
npm install
npm run dev        # local dev → http://localhost:3000
npm run build      # produces ./out/ for static deploy
npm run start      # serve the production build
```

Environment (`.env`):
```
NEXTAUTH_SECRET=...
NEXTAUTH_URL=http://localhost:3000
NEXT_PUBLIC_API_BASE=https://api.your-domain/api/v1/dashboard
```

**Targets:** any static host (S3 + CloudFront, nginx, Caddy, Vercel/Netlify static, internal IIS, etc.). Because everything that needs data goes through the Go API, the frontend has no server-side runtime dependency.

### Backend — `iplant-go-api`

Single Go binary, HTTP on `:8080` and gRPC on `:9090` by default.

```bash
go run ./cmd/api
# or build a static binary
go build -o iplant-api ./cmd/api
./iplant-api
```

Environment (`.env`):
```
DB_HOST=...          DB_PORT=1433       DB_USER=...       DB_PASSWORD=...
DB_NAME=FIELD_PLUS_V1
JWT_SECRET=...       JWT_ISSUER=iplant-go-api      JWT_TTL_MINUTES=60
PORT=8080            GRPC_PORT=9090
```

**Targets:**
- Bare metal / VM behind nginx (the existing infra for the legacy panel).
- Docker container (`FROM golang:1.26 AS build … FROM gcr.io/distroless/static`).
- Systemd unit on the on-prem estate server.

**Database:** Microsoft SQL Server already in production for each estate — no migration required. The Go API connects via `microsoft/go-mssqldb` and registers one connection pool per estate in `estatereg.Registry` at boot.

**Uploads:** local `FileStore` writes to a configurable directory served at a public URL prefix with random 16-byte filenames (effectively unlisted). Swap for S3 by re-implementing the `FileStore` interface.

### Suggested topology

```
            Internet / VPN
                  │
                  ▼
            ┌───────────────┐
            │  nginx / ALB  │  TLS termination
            └───┬───────┬───┘
                │       │
       /        │       │  /api/v1/*
                ▼       ▼
        ┌────────────┐ ┌────────────────┐
        │ Static     │ │ iplant-go-api  │
        │ Next.js    │ │ (Go binary,    │
        │ bundle     │ │  HTTP + gRPC)  │
        └────────────┘ └───────┬────────┘
                               │
                               ▼
                      ┌──────────────────┐
                      │ MSSQL FIELD_PLUS │
                      │ (per estate)     │
                      └──────────────────┘
```

### CI/CD ideas

- **Frontend:** GitHub Actions → `npm ci && npm run lint && npm run build` → upload `./out/` to the static host.
- **Backend:** GitHub Actions → `go vet ./... && go test ./... && go build ./cmd/api` → push container or scp the binary.
- **DB migrations:** plain SQL files in `iplant-go-api/migrations/`, applied via `sqlcmd` or a thin runner; only additive lookup-table changes have been needed since the operational schema is reused as-is.

---

## 7. Notes & Status

- The **legacy `Estate-Admin/` tree is kept in-repo** as the canonical PRD source — every controller / model / view that the rebuild must cover is referenced in `PRD.md` (~830 lines) and the `.prd-notes/` files.
- The **frontend currently boots with mock data fallback** for some flows so screens are demoable without a live MSSQL connection; production builds point `NEXT_PUBLIC_API_BASE` at the Go API.
- **Light & dark themes** are wired through `next-themes` with class-based Tailwind dark mode; toggle lives in the header.
- The Go API includes a small **gRPC surface** for the attendance domain (used by the mobile companion) alongside the broader HTTP/JSON surface used by the dashboard.
- **Security posture:** JWT-bearer auth, bcrypt password hashing, CORS via wrapper, per-estate DB scoping enforced at middleware level, upload URLs are random 16-byte ids served unauthenticated (toggleable to scoped).

---

*Portfolio doc — last updated 2026-05-27.*
