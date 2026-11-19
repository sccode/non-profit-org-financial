# Non-Profit Financial App

A web app for a non-profit treasurer: expense and donation intake, review and receipts, bank reconciliation, and a yearly project dashboard.

- Design: [docs/DESIGN.md](docs/DESIGN.md)
- Task list: [docs/TASKS.md](docs/TASKS.md) (each task is a GitHub issue)

## Run locally

Requires Node 22 or newer.

```bash
npm install
npm run dev
```

Open http://localhost:3000. The health check is at http://localhost:3000/api/health.

## Scripts

| Command | What it does |
|---|---|
| `npm run dev` | Start the dev server |
| `npm run build` | Production build |
| `npm run start` | Run the production build |
| `npm run typecheck` | TypeScript check |

## Configuration

Copy `.env.example` to `.env.local`. Later tasks add the database URL, Google OAuth credentials, and session secret.
