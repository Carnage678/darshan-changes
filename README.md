# miniproject

Student Performance Tracker — a web app with a React + Vite + TypeScript frontend and a
Node + Express + TypeScript backend (Prisma + SQLite to be added in the next track).

## How to run

- Node.js >= 18.18
- npm >= 9 (required for npm workspaces)

### Setup

```bash
npm install
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
npm run db:setup    # create SQLite DB and seed the real DSCE cohort data
```

### Run commands

From the repo root:

```bash
npm run dev              # run frontend (5173) and backend (4000) in parallel
npm run dev:frontend     # frontend only — Vite dev server on http://localhost:5173
npm run dev:backend      # backend only — Express on http://localhost:4000
npm run typecheck        # typecheck shared, backend, frontend
npm run build            # production frontend build (vite build)
npm run start:backend    # run backend with tsx (production-style)
```

The Vite dev server proxies `/api/*` to the backend (`VITE_API_PROXY_TARGET`,
default `http://localhost:4000`). Verify the backend is up by visiting
`http://localhost:4000/api/health`.

## Workspace layout

```
backend/    Express API (TypeScript, run via tsx)
frontend/   React + Vite app (TypeScript)
shared/     Shared TypeScript types (@miniproj/shared)
```

## Environment variables

Backend (`backend/.env`):

| Var            | Default                       | Purpose                              |
| -------------- | ----------------------------- | ------------------------------------ |
| `PORT`         | `4000`                        | Express listen port                  |
| `CORS_ORIGIN`  | `http://localhost:5173`       | Allowed origin for CORS              |
| `DATABASE_URL` | `file:./prisma/dev.db`        | Prisma SQLite database (next track)  |

Frontend (`frontend/.env`):

| Var                       | Default                  | Purpose                              |
| ------------------------- | ------------------------ | ------------------------------------ |
| `VITE_API_BASE_URL`       | `/api`                   | Base path used by the API client     |
| `VITE_API_PROXY_TARGET`   | `http://localhost:4000`  | Vite dev-server proxy upstream       |

## Agent Mistake Log

- 2026-04-30: Initial worktree merge left frontend without a complete `src/main.tsx`, causing Vite build failure. Fixed by wiring a proper app entry with routing/auth shell and re-running `npm run typecheck && npm run build`.
