# Apex Manufacturing ERP - Production-Grade Enterprise Platform

Apex Manufacturing ERP is a full-stack, enterprise-grade Manufacturing Execution System (MES) and Enterprise Resource Planning (ERP) platform designed for discrete and process manufacturing operations.

---

## 🏗️ Architecture Overview

```
                                  +------------------------------------+
                                  |    Next.js 14 Frontend Client      |
                                  | (App Router, Tailwind, Lucide, UI) |
                                  +-----------------+------------------+
                                                    |
                                          HTTP / REST API (JWT)
                                                    |
                                                    v
+---------------------------------------------------------------------------------------------------+
|                                NestJS / TypeScript Backend Engine                                 |
|                                                                                                   |
|  [RequestIdMiddleware] -> [JwtAuthGuard] -> [RolesGuard (RBAC)] -> [LoggingInterceptor]          |
|                                                                                                   |
|   +--------------------------------------- Modules -------------------------------------------+   |
|   |  - AuthModule (JWT, Bcrypt, Refresh)        - InventoryModule (Stock movements, Warehousing)  |   |
|   |  - UsersModule (RBAC administration)       - BOMModule (Multi-level Bill of Materials)       |   |
|   |  - WorkOrdersModule (Shop floor planning)  - QualityModule (Inspection & Defect tracking)    |   |
|   |  - MachinesModule (Workstation telemetry)                                                     |   |
|   +-------------------------------------------------------------------------------------------+   |
|                                                                                                   |
|   [TransformResponseInterceptor] Centralized wrapper -> { success, data, meta }                  |
|   [GlobalExceptionFilter] Centralized error handler  -> { success: false, errorCode, message }    |
+------------------------------------+---------------------------------------+----------------------+
                                     |                                       |
                         Prisma Client (ORM / Pool)                 ioredis (Cache / Queue)
                                     |                                       |
                                     v                                       v
                     +-------------------------------+       +-------------------------------+
                     |     PostgreSQL 16 Alpine      |       |        Redis 7 Alpine         |
                     |  (Persistent WAL, Healthcheck)|       |  (AOF Persistence, LRU Cache) |
                     +-------------------------------+       +-------------------------------+
```

---

## 🚀 Key Deliverables Implemented

### 1. Docker Compose for Infrastructure (`docker-compose.yml`)
- **PostgreSQL 16 Alpine**:
  - Auto-initialization scripts (`docker/postgres/init.sql`) enabling `uuid-ossp` and `pgcrypto`.
  - Tuned PostgreSQL performance flags (`shared_buffers=256MB`, `work_mem=16MB`, `max_connections=150`).
  - Native Docker healthchecks (`pg_isready`) and volume persistence.
- **Redis 7 Alpine**:
  - Password authentication (`--requirepass`).
  - Append-only file (`--appendonly yes`) persistence for transactional cache reliability.
  - LRU memory eviction policy (`allkeys-lru`).
- **Developer Inspection Profiles**:
  - Optional pgAdmin 4 and Redis Commander services via `docker compose --profile tools up -d`.

### 2. Complete Modular Folder Separation
- **Backend (`backend/src/`)**:
  - `database/`: Dedicated `PrismaService` lifecycle manager, `RedisService`, connection pooling, transactions, and migration seeds.
  - `common/`: Centralized interfaces, DTOs, Enums, Exceptions, Guards (`JwtAuthGuard`, `RolesGuard`), Interceptors, and Middlewares.
  - `modules/auth/`: JWT access/refresh token generation, password hashing via `bcrypt`, and user authentication.
  - `modules/users/`: User management with strict role-based access.
  - `modules/work-orders/`: Production order scheduling, finite capacity tracking, and status state machine.
  - `modules/inventory/`: Raw materials, finished goods, stock movements, and ledger tracking.
  - `modules/bom/`: Multi-level Bill of Materials definition and scrap factor calculations.
  - `modules/quality/`: Quality assurance inspections, sample sizes, and defect tracking.
  - `modules/machines/`: Machine workstation status, hourly rates, and capacity tracking.
- **Frontend (`frontend/src/`)**:
  - Next.js 14 App Router layout structure (`(dashboard)` and `(auth)` route groups).
  - Modern Tailwind CSS styling with Lucide icons.
  - Strongly typed API client consuming the backend's uniform response structure.

### 3. Centralized API Response Wrapper & Global Error Handler Middleware
- **Uniform Success Structure (`TransformResponseInterceptor`)**:
  ```json
  {
    "success": true,
    "statusCode": 200,
    "message": "Work orders retrieved successfully",
    "data": [ ... ],
    "meta": {
      "timestamp": "2026-10-01T08:00:00.000Z",
      "path": "/api/v1/work-orders",
      "requestId": "6cf6dfbe-a9a3-4876-b9b5-f8ea55530ec3",
      "durationMs": 12,
      "pagination": {
        "page": 1,
        "limit": 20,
        "totalItems": 4,
        "totalPages": 1,
        "hasNextPage": false,
        "hasPreviousPage": false
      },
      "version": "v1"
    }
  }
  ```
- **Uniform Error Structure (`GlobalExceptionFilter`)**:
  - Handles NestJS `HttpException`, Prisma database exceptions (`P2002` uniqueness, `P2025` missing record, `P2003` foreign key), class-validator arrays, and unexpected runtime errors.
  - Protects production systems by stripping internal stack traces while preserving error correlation via `requestId`.
  ```json
  {
    "success": false,
    "statusCode": 409,
    "errorCode": "DB_UNIQUE_VIOLATION",
    "message": "A record with this sku already exists in the system.",
    "details": { "fields": ["sku"] },
    "meta": {
      "timestamp": "2026-10-01T08:00:00.000Z",
      "path": "/api/v1/inventory",
      "requestId": "6cf6dfbe-a9a3-4876-b9b5-f8ea55530ec3",
      "version": "v1"
    }
  }
  ```

---

## 🔒 Role-Based Access Control (RBAC) Hierarchy

| Role | Responsibilities | Permissions |
|---|---|---|
| `SUPER_ADMIN` | System Governance | Full unrestricted platform access |
| `PLANT_MANAGER` | Plant Management | Manage plant schedules, users, work orders, BOMs, machines |
| `PRODUCTION_SUPERVISOR` | Production Line Lead | Create/release work orders, allocate machines, issue inventory |
| `INVENTORY_MANAGER` | Warehouse Lead | Stock counts, inventory receipts, scrap movements, reorder thresholds |
| `QUALITY_INSPECTOR` | QA / Metrology Lead | Record sample inspections, pass/fail status, quarantine items |
| `OPERATOR` | Shop Floor Technician | View assigned work orders, report progress, record completions |
| `MAINTENANCE_TECH` | Reliability Engineer | Machine workstation maintenance, downtime logging |

---

## 🛠️ Quick Start Guide

### 1. Start Database & Redis via Docker
```bash
docker compose up -d
```
*(Optional) Access pgAdmin at `http://localhost:5050` and Redis Commander at `http://localhost:8081` by running `docker compose --profile tools up -d`.*

### 2. Configure Backend & Seed Database
```bash
cd backend
npm install
npx prisma generate
npx prisma migrate dev --name init
npm run prisma:seed
npm run start:dev
```
- Backend runs on `http://localhost:4000/api/v1`
- Swagger OpenAPI documentation at `http://localhost:4000/api/docs`

### 3. Start Next.js Frontend
```bash
cd ../frontend
npm install
npm run dev
```
- Frontend application runs on `http://localhost:3000`

### Default Seed Credentials
- **Super Admin**: `admin@mfg-erp.local` / `Admin@ERP2026!`
- **Plant Manager**: `manager@mfg-erp.local` / `Admin@ERP2026!`
- **Production Supervisor**: `supervisor@mfg-erp.local` / `Admin@ERP2026!`
- **Quality Inspector**: `qa@mfg-erp.local` / `Admin@ERP2026!`
