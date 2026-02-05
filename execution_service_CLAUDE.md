# Execution Service — CLAUDE.md

Project conventions for Claude Code.

## Project Overview

Autonomous agent execution service with project board UI, policy engine, and external world watchers. See `docs/execution_service_factsheet_v3.md` for the full product spec and `docs/execution_service_implementation_blueprint.md` for the implementation plan.

## Project Structure

- Monorepo with pnpm workspaces
- `packages/shared/` — Shared types and Zod schemas (imported by both server and web)
- `packages/server/` — Fastify backend (API, WebSocket, agent runner, policy engine, watchers, sleep-time compute)
- `packages/web/` — Next.js frontend (board, chat, trace viewer, settings)
- `docs/` — Spec documents (factsheet, blueprint)

## Build, Test, and Development Commands

- Install deps: `pnpm install`
- Start dev (backend + frontend): `pnpm dev`
- Start backend only: `pnpm --filter server dev`
- Start frontend only: `pnpm --filter web dev`
- Build: `pnpm build`
- Lint + format: `pnpm check`
- Fix lint: `pnpm check --fix`
- Tests: `pnpm test`
- Test coverage: `pnpm test:coverage`
- DB migrations: `pnpm --filter server db:migrate`
- DB push (dev): `pnpm --filter server db:push`
- Type check: `pnpm typecheck`
- Docker (local infra): `docker compose up -d` (Postgres; Redis optional Phase 2+)

## Coding Style

- Language: TypeScript (ESM, strict mode)
- Formatting + linting: Biome
- Validation: Zod for all API inputs and config
- ORM: Drizzle (type-safe, SQL-like)
- No `any` — use strict typing throughout
- Keep files under ~500 LOC; split when it improves clarity
- Brief code comments for non-obvious logic only
- Error handling: validate at system boundaries (API inputs, external data), trust internal code

## Architecture Principles

- All agent events route through the Policy Engine before reaching the user
- WebSocket for all real-time updates (board, comments, traces, discussions, notifications)
- Injection queue (in-memory Phase 1, Redis Streams Phase 2+) for user -> agent communication
- Tool side effects pipeline: tools execute, then side effects update board/DB, events broadcast via WebSocket
- Plan-to-board projection: agent's plan is the source of truth; work items are derived
- Traces stored in Postgres (traces table)
- All agents run in a single shared Docker container; workspace isolation via /workspace/{task_id}/
- DB uses native UUID primary keys throughout

## Key Data Flow

```
User (chat/board) -> REST API -> creates task -> Agent Runner runs in shared container
Agent loop: LLM call -> tool call -> side effects -> trace -> next iteration
Side effects: update board, post comments, publish deliverables, route notifications
User comments -> injection queue -> agent sees on next iteration
Policy Engine: filters all notifications by priority/policy before delivery
```

## Naming Conventions

- Files: `kebab-case.ts`
- Types/Interfaces: `PascalCase`
- Functions/variables: `camelCase`
- DB tables: `snake_case`
- API routes: `/api/kebab-case`
- WebSocket events: `snake_case` type field

## Testing

- Framework: Vitest
- Unit tests: colocated `*.test.ts`
- Integration tests: `*.integration.test.ts` (requires DB)
- Test DB: use testcontainers or separate test database
- Run tests before pushing: `pnpm test`

## Git Conventions

- Commit messages: conventional commits (`feat:`, `fix:`, `refactor:`, `docs:`, `test:`)
- One feature/fix per commit
- Keep PRs focused; don't bundle unrelated changes
