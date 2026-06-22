# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Setup

1. Clone the repository and navigate to the root.
2. Install Node.js (>=20) and pnpm (or npm/yarn).
3. Install dependencies:
   ```bash
   pnpm install   # installs all workspace packages
   ```
4. Copy `.env.example` to `.env` and fill in required values (database URL, JWT secret, etc.).
5. Run the development environment:
   ```bash
   pnpm dev   # starts frontend, backend, and private dashboard concurrently
   ```

## Common Commands

| Purpose                              | Command                                 |
|--------------------------------------|-----------------------------------------|
| Install dependencies                 | `pnpm install`                          |
| Start all services (dev)             | `pnpm dev`                              |
| Build frontend for production        | `pnpm -r build` (or `pnpm build --filter client`) |
| Build backend (transpile TS)         | `pnpm -r build` (or `pnpm build --filter server`) |
| Run linting (ESLint + Prettier)      | `pnpm lint`                             |
| Run unit tests                       | `pnpm test`                             |
| Run a single test file               | `pnpm test -- src/server/__tests__/user.test.ts` |
| Start only frontend dev server       | `pnpm dev --filter client`              |
| Start only backend dev server        | `pnpm dev --filter server`              |
| Start only private dashboard         | `pnpm dev --filter dashboard`           |
| Run database migrations              | `pnpm prisma migrate dev`               |
| Generate Prisma client               | `pnpm prisma generate`                  |
| Run end-to-end tests (Playwright)    | `pnpm e2e`                              |
| Docker compose up (local)            | `docker compose up --build`             |
| Docker compose down                  | `docker compose down`                   |

## High‑Level Architecture

The repository is a **monorepo** (managed by pnpm) composed of three main applications:

1. **client** – Public‑facing portfolio website (SPA).  
   - Built with React 18 + Vite + TypeScript.  
   - Uses Framer Motion for smooth animations and page transitions.  
   - Pages: `Home` (hero + about), `Projects` (grid with filters), `Contact` (optional).  
   - Consumes REST/GraphQL API from the `server`.  
   - Deployed to a CDN (e.g., Vercel, Netlify, or AWS S3+CloudFront).

2. **server** – Backend API and auth service.  
   - Node.js 20 + Express + TypeScript.  
   - RESTful endpoints (`/api/projects`, `/api/about`, `/api/contact`) and optional GraphQL wrapper.  
   - Auth via JWT (HttpOnly cookie) for protected dashboard routes.  
   - Data layer: Prisma ORM with PostgreSQL (supports SQLite for local dev).  
   - Background jobs: optional BullMQ queue for email notifications or webhook processing.  
   - Exposes health (`/health`) and metrics (`/metrics` via Prometheus client).

3. **dashboard** – Private, Jira‑like progress tracker (accessible only to the owner).  
   - Same tech stack as `client` but isolated under `/dashboard` route.  
   - Features:  
     - Daily consistency calendar (heatmap).  
     - Pending tasks list (manually entered or synced via Jira API).  
     - Burndown chart and velocity metrics.  
     - Settings to connect personal Jira instance (OAuth2 / API token).  
   - Stores its own data in the same PostgreSQL schema (`dashboard_*` tables).  
   - Protected by middleware that checks for a valid session cookie; redirects to login if missing.

### Shared Packages

- `tsconfig` – Base TypeScript config.
- `eslint-config` – Shared ESLint rules (includes Prettier).
- `ui` – Component library (buttons, inputs, modal) used by both `client` and `dashboard`.
- `api-types` – Generated TypeScript types for API endpoints (using `openapi-typescript` or manual sync).

### Deployment Overview

- **Development**: `pnpm dev` runs all three apps concurrently via `concurrently` or `turbo`.
- **Production**:  
  - `client` built to static assets → served via CDN.  
  - `server` deployed as a Docker container (or serverless) behind an ALB/API Gateway.  
  - `dashboard` can be deployed alongside the server (same container) or as a separate internal service behind VPN/auth‑only subdomain.
- Container images are built with a multi‑stage Dockerfile (see `docker-compose.yml`).

## Code Organization (selected directories)

```
/client          – Vite/React SPA
  /src
    /components  – Reusable UI (buttons, cards, animations)
    /pages       – Route‑based pages (Home, Projects, …)
    /hooks       – Custom React hooks (api, auth)
    /utils       – Helpers (date formatting, analytics)
/server
  /src
    /controllers – Request handlers
    /services    – Business logic (Prisma wrappers)
    /middleware  – Auth, error handling, logging
    /routes      – API route definitions
    /utils       – Validators, email helpers
    /prisma      – Prisma schema & migrations
/dashboard       – Similar structure to client, scoped to `/dashboard`
/packages
  /ui            – Shared component library (styled with Tailwind or CSS modules)
  /api-types     – Generated API contracts
  /tsconfig      – Base tsconfig.json
  /eslint-config – Shared ESLint configuration
docker-compose.yml   – Defines server, db, optional adminer, redis (for queues)
```

## Technology Stack (suggested)

- **Language**: TypeScript (strict mode)
- **Frontend**: React 18, Vite, Framer Motion, Tailwind CSS (or CSS‑modules)
- **Backend**: Node.js 20, Express, Prisma ORM
- **Database**: PostgreSQL (local dev via Docker; production managed AWS RDS or Supabase)
- **Auth**: JWT (access token in HttpOnly cookie), optional refresh token rotation
- **State Management**: React Query / TanStack Query (client & dashboard)
- **Testing**:
  - Unit: Vitest + React Testing Library (frontend), Vitest + Supertest (backend)
  - E2E: Playwright (chromium)
- **Linting/Formatting**: ESLint (with @typescript-eslint, eslint-plugin-react, prettier), Prettier
- **CI/CD**: GitHub Actions (lint, test, build, Docker push, deploy)
- **Observability**:
  - Backend metrics exposed via `prom-client` -> scraped by Prometheus
  - Frontend performance via `web-vitals` library
  - Logs via pino (structured JSON) -> forwarded to Loki or CloudWatch

## Private Dashboard Specifics

- Accessible only at `/dashboard` (or subdomain `dash.<domain>`).  
- Middleware checks for a valid session (`req.session.userId`).  
- Routes:
  - `GET /dashboard` – Overview (calendar heatmap, quick stats)  
  - `GET /dashboard/tasks` – Pending tasks list (supports filtering, drag‑drop reorder)  
  - `POST /dashboard/tasks` – Create/update task  
  - `GET /dashboard/connect-jira` – OAuth2 flow to sync Jira projects (stores encrypted credentials)  
  - `GET /dashboard/settings` – User preferences, export/backup data  
- Data model includes tables: `DashboardTask`, `DashboardSyncLog`, `UserPreference`.
- All dashboard routes are **SSR‑disabled** (pure SPA) to keep bundle small; the server just serves the static bundle and injects the user context via a cookie‑read endpoint.

## Testing Guidelines

- **Unit tests** live beside the file they test (`*.test.ts`).  
- Aim for:  
  - 80%+ line coverage on backend services/controllers.  
  - 70%+ coverage on frontend hooks and utilities (UI components tested via RTL).  
- **E2E tests** cover critical user flows:  
  - Portfolio site loads, projects list filters, contact form submission.  
  - Dashboard login, task creation, Jira sync simulation (mocked).  
- Run tests in CI on every push; require passing before merge.

## Code Quality & Style

- Use **pre‑commit hooks** via `lefthook` or `husky` to run `lint` and `test` on staged files.  
- Commit messages: follow Conventional Commits (`feat:`, `fix:`, `docs:`, etc.).  
- PR template: include checklist for testing, lint, and documentation updates.  
- Avoid `any` TypeScript type; prefer generics or `unknown` with runtime validation (zod).  
- Keep bundle size under 200 KB gzipped for the client (monitor via `vite build` report).  

## Environment Variables (example `.env`)

```
# Server
PORT=4000
DATABASE_URL="postgresql://postgres:password@localhost:5432/portfolio?schema=public"
JWT_ACCESS_SECRET="supersecret"
JWT_REFRESH_SECRET="supersecretrefresh"
NODE_ENV=development

# Dashboard (same server)
DASHBOARD_ENABLE_JIRA_SYNC=true

# Client (exposed at build time)
VITE_API_URL="http://localhost:4000/api"
VITE_GA_ID="UA-XXXXX-Y"

# Optional
REDIS_URL="redis://localhost:6379"
```

> **Note**: Never commit real secrets. Use GitHub Secrets or AWS Parameter Store for production.

## Docker & Docker Compose (quick start)

```yaml
# docker-compose.yml (excerpt)
services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: portfolio
    volumes:
      - pgdata:/var/lib/postgresql/data
  server:
    build: ./server
    env_file: .env
    depends_on: [db]
    ports: ["4000:4000"]
  # optional adminer for DB inspection
  adminer:
    image: adminer
    ports: ["8080:8080"]
volumes:
  pgdata:
```

Run with `docker compose up --build`. The `client` and `dashboard` can be served via the server’s static build or a separate CDN in production.

## When Adding New Features

1. **Backend**:  
   - Add Prisma model (if needed) → run `prisma migrate dev --name <feature>`.  
   - Create service functions in `src/services/`.  
   - Expose via controller in `src/controllers/` and register route in `src/routes/`.  
   - Write unit tests for service and controller.  
   - Update OpenAPI spec (if used) and regenerate `api-types`.

2. **Frontend / Dashboard**:  
   - Add new route under `src/pages/`.  
   - Create component(s) in `src/components/`.  
   - Use React Query to fetch/mutate via the API client (generated from `api-types`).  
   - Add unit tests for hooks/components.  
   - Add E2E scenario covering the new flow.

3. **Shared UI / Packages**:  
   - Follow the same pattern; ensure TS builds (`pnpm build --filter <pkg>`).  
   - Update `tsconfig` references if necessary.

## Documentation

- Keep `README.md` at the repo root with a quick start guide (as above).  
- Each package (`client`, `server`, `dashboard`, `ui`) should have its own `README.md` describing purpose and any special scripts.  
- Architecture diagrams (if any) live in `/docs/architecture.png` or as Mermaid in `/docs/architecture.md`.  
- Update CHANGELOG.md using conventional-changelog-cli when releasing.
