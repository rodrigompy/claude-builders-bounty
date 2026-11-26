# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview
- **Framework:** Next.js 15 (App Router, React 19, TypeScript strict mode)
- **Database:** SQLite (`better-sqlite3`) with Drizzle ORM
- **Styling:** Tailwind CSS + `clsx` / `tailwind-merge`
- **Validation:** Zod
- **Testing:** Vitest (unit/integration) + Playwright (E2E)

## Common Commands
- `npm run dev` — Start local development server on `http://localhost:3000`
- `npm run build` — Create production build (also validates TypeScript and route types)
- `npm run lint` — Run ESLint across `src/`
- `npm run typecheck` — Run `tsc --noEmit`
- `npm run test` — Run Vitest test suite
- `npm run db:generate` — Generate SQLite migration files from Drizzle schema changes
- `npm run db:migrate` — Apply pending migrations to local SQLite database (`./data/app.db`)
- `npm run db:studio` — Open Drizzle Studio to inspect local SQLite tables

## Directory Structure
```text
src/
├── app/                  # Next.js App Router pages, layouts, and API route handlers
│   ├── api/              # REST endpoints (validate inputs with Zod, return NextResponse.json)
│   └── (dashboard)/      # Route groups for authenticated/app views
├── components/           # Reusable UI components (keep pure/presentational where possible)
├── db/
│   ├── index.ts          # Singleton SQLite connection & Drizzle client instance
│   ├── schema.ts         # Drizzle SQLite table definitions and inferred TS types
│   └── migrations/       # Generated SQL migration files (never edit manually)
├── lib/
│   ├── queries/          # Server-only database query functions (grouped by domain)
│   ├── actions/          # Next.js Server Actions ('use server')
│   └── validations/      # Shared Zod schemas for forms, actions, and API routes
└── tests/                # Vitest unit/integration tests using an in-memory SQLite DB
```

## Architecture & Coding Conventions

### Next.js App Router Rules
- Default to **React Server Components (RSC)**. Only add `'use client'` at the leaf component level when browser APIs, event listeners, or React hooks (`useState`, `useEffect`) are required.
- Perform data fetching directly in Server Components via `src/lib/queries/*` rather than calling internal `/api/*` routes from the server.
- Use **Server Actions** (`src/lib/actions/*`) for form submissions and UI mutations; always call `revalidatePath()` or `revalidateTag()` after mutating data.
- Keep `src/app/api/*` route handlers reserved for external webhooks, third-party callbacks, or client-side polling.

### SQLite & Drizzle ORM Rules
- **Single Connection:** Always import the shared `db` client from `src/db/index.ts`. Never instantiate `new Database()` inside routes, actions, or components.
- **SQLite Pragmas:** Ensure `WAL` mode (`PRAGMA journal_mode = WAL;`) and foreign key enforcement (`PRAGMA foreign_keys = ON;`) remain enabled in `src/db/index.ts`.
- **Synchronous Driver:** `better-sqlite3` executes synchronously—do not wrap `db.select()` or `db.insert()` calls in unnecessary `async` overhead unless required by an async Server Action or route signature.
- **Transactions:** Wrap multi-step writes in `db.transaction((tx) => { ... })` so partial writes roll back cleanly on error.
- **Schema Changes:** Edit `src/db/schema.ts`, run `npm run db:generate`, inspect the generated SQL in `src/db/migrations/`, and apply with `npm run db:migrate`. Never alter the SQLite file manually.
- **Git Hygiene:** Never commit `*.db`, `*.db-wal`, or `*.db-shm` files. Keep `data/` in `.gitignore`.

### TypeScript & Validation
- Strict TypeScript is enforced (`"strict": true`). Never use `any`; use `unknown` and narrow types safely.
- Infer database types directly from Drizzle (`typeof users.$inferSelect` and `typeof users.$inferInsert`) instead of duplicating interfaces.
- Validate all external inputs (Server Action `FormData`, API `request.json()`, and `searchParams`) with Zod schemas from `src/lib/validations/` before touching the database.

## Testing Guidelines
- Configure Vitest tests to run against `:memory:` SQLite databases with migrations applied in `beforeEach` so tests stay fast, isolated, and deterministic.
- Run `npm run typecheck`, `npm run lint`, and `npm run test` before finishing any task.
