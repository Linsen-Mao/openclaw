# Execution Service — Implementation Blueprint

**Purpose**: This document provides everything a fresh Claude Code session needs to implement the Execution Service from scratch. Read this document first, then reference `execution_service_factsheet_v3.md` for detailed data models, interfaces, and behavior specifications.

---

## 1. Project Identity

- **Name**: `execution-service` (repo name, package name)
- **Description**: Web-based autonomous agent platform with project board, policy engine, and external world watchers
- **Language**: TypeScript (ESM, strict mode)
- **Runtime**: Node.js 22+
- **Package Manager**: pnpm

---

## 2. Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| **Backend Framework** | Fastify | Fast, TypeScript-first, schema validation, WebSocket support via `@fastify/websocket` |
| **Database** | PostgreSQL 16 | JSONB for flexible schemas, solid relational model |
| **DB Client** | Drizzle ORM | Type-safe, SQL-like, lightweight, good migration story |
| **Queue / Pub-Sub** | In-memory (Phase 1), Redis Streams (Phase 2+, via `ioredis`) | Agent injection queue |
| **WebSocket** | `@fastify/websocket` | Integrated with Fastify, handles board updates + sync chat + trace streaming |
| **Container Runtime** | Docker Engine API (via `dockerode`) | Shared agent container (all tasks in one container) |
| **LLM Client** | Anthropic SDK (`@anthropic-ai/sdk`) + OpenAI SDK (`openai`) | Multi-provider LLM calls |
| **Frontend** | Next.js 15 (App Router) + React 19 | SSR, API route proxying, streaming |
| **UI Components** | shadcn/ui + Tailwind CSS 4 | Fast to build, accessible, customizable |
| **Board DnD** | `@dnd-kit/core` + `@dnd-kit/sortable` | Kanban drag-and-drop |
| **Client State** | TanStack Query v5 | Server state cache, optimistic updates |
| **Markdown** | `react-markdown` + `rehype-raw` | Render deliverables, comments, plans |
| **Scheduler** | `node-cron` | Sleep-time compute jobs, weekly review scheduling, watcher polling |
| **Testing** | Vitest | Fast, ESM-native, good TypeScript support |
| **Linting** | Biome | Fast, unified formatter + linter |
| **Validation** | Zod | Runtime validation for API inputs, policy configs |

---

## 3. Repository Structure

```
execution-service/
├── CLAUDE.md                          # Project conventions for Claude Code
├── package.json                       # Root workspace config
├── pnpm-workspace.yaml               # Workspace packages
├── tsconfig.json                      # Base TypeScript config
├── biome.json                         # Linter + formatter config
├── docker-compose.yml                 # Local dev: Postgres (Redis optional Phase 2+)
├── .env.example                       # Environment variable template
│
├── packages/
│   ├── shared/                        # Shared types + schemas
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── index.ts               # Re-exports all shared types
│   │       ├── types/
│   │       │   ├── task.ts            # Task, WorkItem, TaskStatus, etc.
│   │       │   ├── comment.ts         # Comment, CommentType
│   │       │   ├── deliverable.ts     # Deliverable, ReviewStatus
│   │       │   ├── discussion.ts      # DiscussionRequest, DiscussionSession, DiscussionMessage
│   │       │   ├── trace.ts           # TraceEntry, TraceEventType, TraceFilter
│   │       │   ├── policy.ts          # UserPolicy, AttentionPolicy, WIPPolicy, CostPolicy, etc.
│   │       │   ├── watcher.ts         # Watcher, SourcePlugin, NormalizedEvent, WatcherCondition
│   │       │   ├── agent.ts           # AgentLoopConfig, AgentLoopResult, AgentInjection
│   │       │   ├── user.ts            # User
│   │       │   └── ws-events.ts       # WebSocketEvent union type
│   │       └── schemas/
│   │           ├── source-config.ts   # Per-plugin Zod schemas (EmailImap, ApiPoll, etc.)
│   │           └── policy.ts          # Policy Zod schemas
│   │
│   ├── server/                        # Backend (Fastify)
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── src/
│   │   │   ├── index.ts               # Server entrypoint
│   │   │   ├── config.ts              # Environment config (Zod-validated)
│   │   │   │
│   │   │   ├── db/
│   │   │   │   ├── client.ts          # Drizzle client setup
│   │   │   │   ├── schema.ts          # Drizzle schema (all tables)
│   │   │   │   └── migrate.ts         # Migration runner
│   │   │   │
│   │   │   ├── api/
│   │   │   │   ├── routes.ts          # Route registration
│   │   │   │   ├── chat.ts            # POST /api/chat, /api/chat/confirm-plan, /api/chat/modify-plan
│   │   │   │   ├── tasks.ts           # CRUD /api/tasks, /api/tasks/:id
│   │   │   │   ├── work-items.ts      # GET /api/tasks/:id/items
│   │   │   │   ├── comments.ts        # CRUD /api/tasks/:id/items/:itemId/comments
│   │   │   │   ├── deliverables.ts    # CRUD + review /api/tasks/:id/deliverables
│   │   │   │   ├── discussions.ts     # Schedule, start, end discussions
│   │   │   │   ├── files.ts           # File browser: list, read, upload, edit, delete
│   │   │   │   ├── traces.ts          # GET /api/tasks/:id/trace
│   │   │   │   ├── digest.ts          # GET /api/digest
│   │   │   │   ├── improvements.ts    # CRUD /api/improvements
│   │   │   │   ├── watchers.ts        # CRUD /api/watchers
│   │   │   │   ├── policies.ts        # GET/PATCH /api/policies
│   │   │   │   └── review.ts          # Weekly review endpoints
│   │   │   │
│   │   │   ├── ws/
│   │   │   │   ├── server.ts          # WebSocket server setup
│   │   │   │   ├── events.ts          # Event type definitions
│   │   │   │   └── broadcast.ts       # Room-based broadcasting (per-task, per-user)
│   │   │   │
│   │   │   ├── agent/
│   │   │   │   ├── loop.ts            # Core agent loop (LLM-in-a-loop)
│   │   │   │   ├── runner.ts          # Agent Runner: container + loop lifecycle
│   │   │   │   ├── tools/
│   │   │   │   │   ├── index.ts       # Tool registry
│   │   │   │   │   ├── bash.ts
│   │   │   │   │   ├── read.ts
│   │   │   │   │   ├── write.ts
│   │   │   │   │   ├── edit.ts
│   │   │   │   │   ├── glob.ts
│   │   │   │   │   ├── grep.ts
│   │   │   │   │   ├── browser.ts
│   │   │   │   │   ├── web-search.ts
│   │   │   │   │   ├── web-fetch.ts
│   │   │   │   │   ├── update-plan.ts
│   │   │   │   │   ├── save-memo.ts
│   │   │   │   │   ├── search-memo.ts
│   │   │   │   │   ├── post-comment.ts
│   │   │   │   │   ├── request-discussion.ts
│   │   │   │   │   ├── spawn-agent.ts
│   │   │   │   │   ├── publish-deliverable.ts
│   │   │   │   │   ├── list-skills.ts
│   │   │   │   │   └── read-skill.ts
│   │   │   │   ├── side-effects.ts    # Tool side effects pipeline
│   │   │   │   ├── injection.ts       # Injection queue (in-memory Phase 1; Redis Phase 2+)
│   │   │   │   ├── compaction.ts      # Context compaction engine
│   │   │   │   ├── planning.ts        # Plan management, risk-first ordering
│   │   │   │   ├── risk.ts            # Risk classification, doom loop detection
│   │   │   │   ├── sub-agent.ts       # Sub-agent manager
│   │   │   │   └── self-assessment.ts # Self-assessment against success criteria
│   │   │   │
│   │   │   ├── board/
│   │   │   │   └── sync.ts            # Board Sync Engine: plan → work items projection
│   │   │   │
│   │   │   ├── chat/
│   │   │   │   ├── handler.ts         # Chat message handler (quick vs task mode)
│   │   │   │   ├── classify.ts        # Intent classification (quick/task)
│   │   │   │   └── goal-analysis.ts   # Goal clarity analysis, refinement questions
│   │   │   │
│   │   │   ├── discussion/
│   │   │   │   ├── scheduler.ts       # Discussion scheduling, availability
│   │   │   │   └── sync-chat.ts       # Real-time sync discussion handler
│   │   │   │
│   │   │   ├── policy/
│   │   │   │   ├── engine.ts          # PolicyEngine facade (single evaluate() entry point)
│   │   │   │   ├── attention.ts       # AttentionPolicy logic
│   │   │   │   ├── wip.ts             # WIPPolicy logic
│   │   │   │   ├── autonomy.ts        # AutonomyPolicy logic
│   │   │   │   ├── planning.ts        # PlanningPolicy logic
│   │   │   │   ├── cost.ts            # CostPolicy budget checking
│   │   │   │   └── defaults.ts        # Default policy values
│   │   │   │
│   │   │   ├── notification/
│   │   │   │   ├── service.ts         # Notification routing (priority → policy → delivery)
│   │   │   │   ├── priority.ts        # Priority classification (low/medium/high/blocker)
│   │   │   │   └── digest.ts          # Digest builder + delivery
│   │   │   │
│   │   │   ├── watcher/
│   │   │   │   ├── service.ts         # Watcher Service lifecycle manager
│   │   │   │   ├── condition.ts       # Condition evaluators (regex, threshold, llm_judge, compound)
│   │   │   │   ├── dedup.ts           # Deduplication state
│   │   │   │   ├── reconnect.ts       # Exponential backoff reconnect
│   │   │   │   └── plugins/
│   │   │   │       ├── types.ts       # SourcePlugin interface, NormalizedEvent
│   │   │   │       ├── email-imap.ts
│   │   │   │       ├── webhook.ts
│   │   │   │       ├── api-poll.ts
│   │   │   │       ├── rss.ts
│   │   │   │       ├── websocket.ts
│   │   │   │       └── cron.ts
│   │   │   │
│   │   │   ├── sleep-time/
│   │   │   │   ├── scheduler.ts       # Job scheduler (node-cron)
│   │   │   │   ├── daily-digest.ts
│   │   │   │   ├── memory-consolidation.ts
│   │   │   │   ├── failure-detection.ts
│   │   │   │   ├── stale-detection.ts
│   │   │   │   ├── skill-extraction.ts
│   │   │   │   └── workspace-cleanup.ts
│   │   │   │
│   │   │   ├── review/
│   │   │   │   ├── weekly.ts          # Weekly review briefing generator
│   │   │   │   └── calibration.ts     # Behavior feedback → policy suggestions
│   │   │   │
│   │   │   ├── trace/
│   │   │   │   ├── store.ts           # Trace storage (Postgres-backed)
│   │   │   │   ├── query.ts           # Trace query API
│   │   │   │   └── summary.ts         # Trace summarization for sleep-time compute
│   │   │   │
│   │   │   ├── improvement/
│   │   │   │   ├── proposals.ts       # Improvement proposal CRUD
│   │   │   │   └── apply.ts           # Apply/rollback changes
│   │   │   │
│   │   │   ├── container/
│   │   │   │   ├── manager.ts         # Shared Docker container lifecycle (singleton)
│   │   │   │   ├── workspace.ts       # Per-task workspace directory management
│   │   │   │   └── config.ts          # Container configuration
│   │   │   │
│   │   │   ├── context/
│   │   │   │   ├── service.ts         # Context Service (knowledge store)
│   │   │   │   └── memory.ts          # Tiered memory (working, memos, consolidated)
│   │   │   │
│   │   │   └── llm/
│   │   │       ├── client.ts          # Multi-provider LLM client
│   │   │       └── models.ts          # Model registry (thinking, fast, etc.)
│   │   │
│   │   └── drizzle/
│   │       └── migrations/            # SQL migration files
│   │
│   └── web/                           # Frontend (Next.js)
│       ├── package.json
│       ├── tsconfig.json
│       ├── next.config.ts
│       ├── tailwind.config.ts
│       ├── src/
│       │   ├── app/
│       │   │   ├── layout.tsx         # Root layout
│       │   │   ├── page.tsx           # Home: Chat + Board overview
│       │   │   ├── board/
│       │   │   │   └── page.tsx       # Full board view
│       │   │   ├── task/
│       │   │   │   └── [id]/
│       │   │   │       ├── page.tsx   # Task detail
│       │   │   │       ├── trace/
│       │   │   │       │   └── page.tsx  # Trace viewer
│       │   │   │       └── discuss/
│       │   │   │           └── [requestId]/
│       │   │   │               └── page.tsx  # Sync discussion
│       │   │   ├── digest/
│       │   │   │   └── page.tsx       # Daily digest
│       │   │   ├── improvements/
│       │   │   │   └── page.tsx       # Improvement proposals
│       │   │   ├── watchers/
│       │   │   │   └── page.tsx       # Watcher management
│       │   │   ├── review/
│       │   │   │   └── page.tsx       # Weekly review
│       │   │   └── settings/
│       │   │       └── page.tsx       # Settings + policy config
│       │   │
│       │   ├── components/
│       │   │   ├── ui/                # shadcn/ui components
│       │   │   ├── chat/
│       │   │   │   ├── chat-panel.tsx
│       │   │   │   ├── message.tsx
│       │   │   │   └── plan-proposal.tsx
│       │   │   ├── board/
│       │   │   │   ├── kanban-board.tsx
│       │   │   │   ├── task-card.tsx
│       │   │   │   ├── work-item-card.tsx
│       │   │   │   └── column.tsx
│       │   │   ├── task/
│       │   │   │   ├── task-detail.tsx
│       │   │   │   ├── comment-thread.tsx
│       │   │   │   ├── deliverable-viewer.tsx
│       │   │   │   └── success-criteria.tsx
│       │   │   ├── files/
│       │   │   │   ├── file-browser.tsx
│       │   │   │   └── file-viewer.tsx
│       │   │   ├── trace/
│       │   │   │   ├── trace-viewer.tsx
│       │   │   │   └── trace-timeline.tsx
│       │   │   ├── review/
│       │   │   │   ├── weekly-review.tsx
│       │   │   │   └── behavior-calibration.tsx
│       │   │   └── policy/
│       │   │       └── policy-editor.tsx
│       │   │
│       │   ├── hooks/
│       │   │   ├── use-websocket.ts
│       │   │   ├── use-tasks.ts
│       │   │   └── use-policy.ts
│       │   │
│       │   └── lib/
│       │       ├── api.ts             # API client
│       │       └── ws.ts              # WebSocket client
│       │
│       └── public/
│           └── ...
│
└── docs/
    ├── factsheet.md                        # Full spec (copy from openclaw repo)
    └── blueprint.md                        # Implementation plan (copy from openclaw repo)
```

---

## 4. Implementation Phases

Build in this order. Each phase produces a working (if incomplete) system. Phase 1 is split into four sub-phases to keep each step focused. Frontend is deferred to Phase 5 — build all backend APIs first.

### Phase 1a: Project Scaffold

**Goal**: Monorepo structure, configuration, local dev environment, shared types package.

Files to implement:
1. Root config: `package.json`, `pnpm-workspace.yaml`, `tsconfig.json`, `biome.json`, `docker-compose.yml`, `.env.example`, `CLAUDE.md`
2. `packages/shared/package.json` + `tsconfig.json` — shared types workspace package
3. `packages/shared/src/types/` — all shared TypeScript interfaces from factsheet (Task, WorkItem, Comment, Deliverable, Discussion, Trace, Policy, Watcher, Agent, User, WebSocket events)
4. `packages/shared/src/schemas/` — Zod schemas for SourceConfig per plugin, policy validation
5. `packages/shared/src/index.ts` — re-export barrel file
6. `packages/server/package.json` + `tsconfig.json` — server workspace package
7. `packages/server/src/config.ts` — env config with Zod (including INJECTION_QUEUE_BACKEND)

**Milestone**: `pnpm install` works, shared types compile, server package can import from `@execution-service/shared`.

### Phase 1b: Database Layer

**Goal**: Full database schema in Drizzle, migrations, client setup.

Files to implement:
1. `packages/server/src/db/schema.ts` — full Drizzle schema (20 tables from factsheet Section 15, including users, traces, discussion_messages, cost_usage)
2. `packages/server/src/db/client.ts` — Drizzle + Postgres connection
3. `packages/server/src/db/migrate.ts` — migration runner
4. `packages/server/drizzle/migrations/` — initial migration SQL

**Milestone**: `docker-compose up -d postgres` + migration runner creates all 20 tables. Can insert/query test data.

### Phase 1c: API Skeleton

**Goal**: Fastify server boots, REST endpoints exist (stubbed), WebSocket server runs.

Files to implement:
1. `packages/server/src/index.ts` — Fastify server bootstrap
2. `packages/server/src/api/routes.ts` — route registration
3. `packages/server/src/api/tasks.ts` — task CRUD (with DB queries)
4. `packages/server/src/api/work-items.ts` — work item listing
5. `packages/server/src/api/comments.ts` — comment CRUD + injection
6. `packages/server/src/api/deliverables.ts` — deliverable review
7. `packages/server/src/api/files.ts` — file browser (reads workspace dir)
8. `packages/server/src/api/traces.ts` — trace query (reads from Postgres)
9. `packages/server/src/api/chat.ts` — chat endpoints (stubbed handler)
10. `packages/server/src/ws/server.ts` — WebSocket server setup
11. `packages/server/src/ws/events.ts` — event type definitions (import from shared)
12. `packages/server/src/ws/broadcast.ts` — room-based broadcasting

**Milestone**: Server starts on port 3001, API endpoints respond, WebSocket connections accepted. Can create tasks via REST.

### Phase 1d: Agent Loop + Execution

**Goal**: A single agent can run in the shared Docker container, execute tools, and store traces in Postgres.

Files to implement:
1. `packages/server/src/llm/client.ts` — LLM client (Anthropic + OpenAI)
2. `packages/server/src/llm/models.ts` — model registry
3. `packages/server/src/container/manager.ts` — shared Docker container lifecycle (singleton)
4. `packages/server/src/container/workspace.ts` — per-task workspace directory management
5. `packages/server/src/container/config.ts` — container config
6. `packages/server/src/agent/tools/` — all tool implementations (bash, read, write, edit, glob, grep, update-plan, save-memo, search-memo, publish-deliverable, list-skills, read-skill)
7. `packages/server/src/agent/loop.ts` — core agent loop
8. `packages/server/src/agent/injection.ts` — injection queue (in-memory for Phase 1)
9. `packages/server/src/agent/compaction.ts` — compaction engine
10. `packages/server/src/agent/risk.ts` — risk classification
11. `packages/server/src/agent/runner.ts` — agent runner lifecycle
12. `packages/server/src/trace/store.ts` — Postgres-backed trace storage
13. `packages/server/src/board/sync.ts` — Board Sync Engine with event emitter pattern

**Milestone**: Can create a task via API, agent runs in shared Docker container, produces traces stored in Postgres, calls tools, plan syncs to board.

### Phase 2: Board + Comments + Collaboration (Backend Only)

**Goal**: Full collaboration backend. No frontend yet — testable via curl/Postman.

Files to implement:
1. `packages/server/src/agent/tools/post-comment.ts` — post_comment tool
2. `packages/server/src/agent/tools/request-discussion.ts` — request_discussion tool
3. `packages/server/src/agent/side-effects.ts` — tool side effects pipeline (uses event emitter for board sync)
4. `packages/server/src/discussion/scheduler.ts` — discussion scheduling, availability
5. `packages/server/src/discussion/sync-chat.ts` — real-time sync discussion handler
6. `packages/server/src/api/discussions.ts` — discussion endpoints

**Milestone**: Agent can post comments, request discussions. User comments inject into agent context. Sync discussions work via WebSocket. All backend-only.

### Phase 3: Chat + Quick Mode (Backend Only)

**Goal**: Chat-based task creation and Quick Mode work end-to-end.

Files to implement:
1. `packages/server/src/chat/classify.ts` — intent classification (quick/task)
2. `packages/server/src/chat/goal-analysis.ts` — goal clarity analysis
3. `packages/server/src/chat/handler.ts` — chat message handler (with escalation support)

**Milestone**: User sends chat message via API, agent classifies intent, refines requirements, proposes plan, user confirms via API, task created.

### Phase 4: Policy Engine + Notifications

**Goal**: Configurable policies govern agent-human interaction. Budget controls enforced.

Files to implement:
1. `packages/server/src/policy/defaults.ts` — default policy values (including CostPolicy)
2. `packages/server/src/policy/engine.ts` — PolicyEngine facade (evaluate entry point)
3. `packages/server/src/policy/attention.ts` — AttentionPolicy
4. `packages/server/src/policy/wip.ts` — WIPPolicy
5. `packages/server/src/policy/autonomy.ts` — AutonomyPolicy
6. `packages/server/src/policy/planning.ts` — PlanningPolicy
7. `packages/server/src/policy/cost.ts` — CostPolicy budget checking
8. `packages/server/src/notification/priority.ts` — priority classification
9. `packages/server/src/notification/service.ts` — notification routing
10. `packages/server/src/notification/digest.ts` — digest batching
11. `packages/server/src/api/policies.ts` — policy CRUD
12. `packages/server/src/agent/self-assessment.ts` — self-assessment pipeline
13. `packages/server/src/agent/planning.ts` — risk-first ordering, success criteria

**Milestone**: Notifications route through PolicyEngine.evaluate(). WIP limits enforced. Cost budgets tracked. Self-assessment works on deliverables.

### Phase 5: Frontend (Next.js)

**Goal**: Full web UI. All backend features become user-accessible.

Files to implement:
1. `packages/web/` — scaffold Next.js app
2. All pages: Home (chat + board), Board, Task Detail, Sync Chat, Trace Viewer, Digest, Improvements, Watchers, Weekly Review, Settings
3. All components: chat panel, kanban board, task cards, comment threads, deliverable viewer, file browser, trace timeline, policy editor, behavior calibration
4. Hooks: use-websocket, use-tasks, use-policy
5. API client + WebSocket client

**Milestone**: Full working web UI backed by the existing API/WebSocket server.

### Phase 6: External World Watchers

**Goal**: Background service monitors external sources and triggers agent actions.

Files to implement:
1. `packages/server/src/watcher/plugins/types.ts` — SourcePlugin interface
2. `packages/server/src/watcher/plugins/email-imap.ts`
3. `packages/server/src/watcher/plugins/webhook.ts`
4. `packages/server/src/watcher/plugins/api-poll.ts`
5. `packages/server/src/watcher/plugins/rss.ts`
6. `packages/server/src/watcher/plugins/websocket.ts`
7. `packages/server/src/watcher/plugins/cron.ts`
8. `packages/server/src/watcher/condition.ts` — condition evaluators
9. `packages/server/src/watcher/dedup.ts` — deduplication
10. `packages/server/src/watcher/reconnect.ts` — reconnect with backoff
11. `packages/server/src/watcher/service.ts` — watcher lifecycle manager
12. `packages/server/src/api/watchers.ts` — watcher CRUD

**Milestone**: Can create watchers that monitor email/APIs/webhooks and trigger task creation or injection.

### Phase 7: Sleep-Time Compute + Self-Improvement

**Goal**: Nightly background jobs analyze traces and propose improvements.

Files to implement:
1. `packages/server/src/sleep-time/scheduler.ts` — cron job scheduler
2. `packages/server/src/sleep-time/daily-digest.ts`
3. `packages/server/src/sleep-time/memory-consolidation.ts`
4. `packages/server/src/sleep-time/failure-detection.ts`
5. `packages/server/src/sleep-time/stale-detection.ts`
6. `packages/server/src/sleep-time/skill-extraction.ts`
7. `packages/server/src/sleep-time/workspace-cleanup.ts`
8. `packages/server/src/improvement/proposals.ts` — proposal CRUD
9. `packages/server/src/improvement/apply.ts` — apply/rollback
10. `packages/server/src/trace/query.ts` — trace query (Postgres)
11. `packages/server/src/trace/summary.ts` — trace summarization
12. `packages/server/src/api/digest.ts` — digest endpoints
13. `packages/server/src/api/improvements.ts` — improvement endpoints

**Milestone**: Nightly jobs run, produce digest, detect failures, propose improvements. User reviews and approves.

### Phase 8: Weekly Review + Behavior Calibration

**Goal**: System-scheduled weekly review with briefing, action queue, and behavior calibration.

Files to implement:
1. `packages/server/src/review/weekly.ts` — briefing generator
2. `packages/server/src/review/calibration.ts` — behavior feedback -> policy suggestions
3. `packages/server/src/api/review.ts` — weekly review endpoints

**Milestone**: Weekly review generates briefing, user processes action queue, calibrates agent behavior, system suggests policy changes.

### Phase 9: Sub-Agents + Skills + Browser

**Goal**: Agent can spawn sub-agents, use skills, and browse the web.

Files to implement:
1. `packages/server/src/agent/sub-agent.ts` — sub-agent manager
2. `packages/server/src/agent/tools/spawn-agent.ts`
3. `packages/server/src/agent/tools/browser.ts` — browser tool (Playwright)
4. `packages/server/src/agent/tools/web-search.ts`
5. `packages/server/src/agent/tools/web-fetch.ts`
6. `packages/server/src/context/service.ts` — context service
7. `packages/server/src/context/memory.ts` — tiered memory

**Milestone**: Full agent capability set. Sub-agents work. Skills system works.

---

## 5. Key Dependencies

```json
{
  "packages/shared": {
    "dependencies": {
      "zod": "^3.24"
    },
    "devDependencies": {
      "typescript": "^5.7"
    }
  },
  "packages/server": {
    "dependencies": {
      "@execution-service/shared": "workspace:*",
      "fastify": "^5",
      "@fastify/websocket": "^11",
      "@fastify/cors": "^10",
      "@fastify/multipart": "^9",
      "drizzle-orm": "^0.38",
      "postgres": "^3.4",
      "dockerode": "^4",
      "@anthropic-ai/sdk": "^0.39",
      "openai": "^4",
      "zod": "^3.24",
      "node-cron": "^3",
      "pino": "^9"
    },
    "optionalDependencies": {
      "ioredis": "^5"
    },
    "devDependencies": {
      "drizzle-kit": "^0.30",
      "vitest": "^3",
      "typescript": "^5.7",
      "@types/node": "^22",
      "@types/dockerode": "^3"
    }
  },
  "packages/web": {
    "dependencies": {
      "@execution-service/shared": "workspace:*",
      "next": "^15",
      "react": "^19",
      "react-dom": "^19",
      "@tanstack/react-query": "^5",
      "@dnd-kit/core": "^6",
      "@dnd-kit/sortable": "^10",
      "react-markdown": "^9",
      "rehype-raw": "^7",
      "tailwindcss": "^4",
      "class-variance-authority": "^0.7",
      "lucide-react": "^0.469"
    }
  }
}
```

Note: `ioredis` is optional — only needed when `INJECTION_QUEUE_BACKEND=redis` (Phase 2+).

---

## 6. docker-compose.yml (Local Development)

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: execution_service
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  # Redis (optional — needed only when INJECTION_QUEUE_BACKEND=redis, Phase 2+)
  # redis:
  #   image: redis:7-alpine
  #   ports:
  #     - "6379:6379"

volumes:
  pgdata:
```

---

## 7. Environment Variables (.env.example)

```bash
# Database
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/execution_service

# Redis (optional — Phase 2+ only)
# REDIS_URL=redis://localhost:6379

# Injection Queue
INJECTION_QUEUE_BACKEND=memory          # memory | postgres | redis

# LLM Providers
ANTHROPIC_API_KEY=
OPENAI_API_KEY=

# Models
THINKING_MODEL=claude-sonnet-4-20250514
FAST_MODEL=claude-haiku-4-20250414

# Server
PORT=3001
HOST=0.0.0.0

# Frontend
NEXT_PUBLIC_API_URL=http://localhost:3001
NEXT_PUBLIC_WS_URL=ws://localhost:3001

# Docker (shared container)
AGENT_IMAGE=execution-service-agent:latest
WORKSPACE_BASE_PATH=/tmp/execution-service/workspaces
```

---

## 8. Database Schema Summary

Full SQL is in the factsheet (Section 15). Tables:

| Table | Purpose |
|-------|---------|
| `users` | User accounts with auth provider info |
| `tasks` | Top-level tasks with goal, status, plan, success criteria |
| `work_items` | Plan step projections on the board |
| `comments` | Agent/user/system comments on work items (with supersession) |
| `deliverables` | Published outputs with review status |
| `discussion_requests` | Sync discussion requests |
| `discussion_sessions` | Discussion session metadata (summary, confirmation) |
| `discussion_messages` | Individual messages within a discussion session |
| `improvement_proposals` | Self-improvement proposals from sleep-time compute |
| `applied_changes` | Applied improvement changes (for rollback) |
| `daily_digests` | Generated daily digests |
| `traces` | Full trace entries stored in Postgres |
| `watchers` | External world watcher configs |
| `watcher_history` | Watcher trigger history |
| `chat_messages` | Quick mode conversation history (with escalation link) |
| `user_policies` | Per-user policy configuration (includes cost policy) |
| `weekly_reviews` | Weekly review sessions |
| `notification_queue` | Notification batching queue |
| `behavior_feedback` | Human behavior calibration feedback |
| `cost_usage` | Token/cost tracking for budget enforcement |

---

## 9. API Summary

Full endpoint list is in the factsheet (Section 12.3). Key groups:

| Group | Base Path | Count |
|-------|-----------|-------|
| Chat | `/api/chat` | 3 endpoints |
| Tasks | `/api/tasks` | 5 endpoints |
| Work Items | `/api/tasks/:id/items` | 1 endpoint |
| Comments | `/api/tasks/:id/.../comments` | 3 endpoints |
| Deliverables | `/api/tasks/:id/deliverables` | 3 endpoints |
| Discussions | `/api/tasks/:id/discussions` | 4 endpoints |
| Files | `/api/tasks/:id/files` | 5 endpoints |
| Traces | `/api/tasks/:id/trace` | 2 endpoints |
| Digest | `/api/digest` | 2 endpoints |
| Improvements | `/api/improvements` | 3 endpoints |
| Watchers | `/api/watchers` | 7 endpoints |
| Policies | `/api/policies` | 2 endpoints |
| Weekly Review | `/api/review` | 4 endpoints |

---

## 10. Agent System Prompt

The full system prompt template is in the factsheet (Section 17). Key sections to include:

1. **Collaboration** — how to use `post_comment`, `request_discussion`, `publish_deliverable`
2. **Question supersession** — when/how to supersede old questions
3. **Planning** — risk-first ordering, success criteria, stretch items
4. **Autonomy** — micro-request handling, self-assessment, proactive risk reporting
5. **Manage-up behavior** — surface risks early, propose alternatives, note assumptions

---

## 11. Testing Strategy

| Type | What | Tool |
|------|------|------|
| Unit | Individual modules (policy engine, condition evaluators, compaction) | Vitest |
| Integration | API routes + DB (Postgres test container) | Vitest + testcontainers |
| Agent loop | Mock LLM responses, verify tool calls and side effects | Vitest |
| WebSocket | Event broadcasting, room management | Vitest + ws client |
| Frontend | Component rendering, interaction | Vitest + React Testing Library |
| E2E | Full flow: chat → task → agent runs → deliverable → review | Playwright |

---

## 12. Files to Copy to New Repo

Copy these files from the openclaw repo into the new `execution-service/` directory:

| Source (openclaw repo) | Destination (new repo) | Purpose |
|------------------------|----------------------|---------|
| `execution_service_factsheet_v3.md` | `docs/factsheet.md` | Complete product spec: all data models, interfaces, behaviors, DB schema (20 tables with UUID PKs), API, agent loop, compaction, tools, policy engine with cost budgets, system prompt |
| `execution_service_implementation_blueprint.md` | `docs/blueprint.md` | Implementation plan: repo structure (with packages/shared), phases (1a-1d, 2-9), tech stack, dependencies |
| `execution_service_CLAUDE.md` | `CLAUDE.md` | Project conventions for Claude Code |

The factsheet is self-contained — it includes both the high-level product features (Sections 1-12) and the core agent architecture (Section 16: agent loop, compaction, planning, tools, sub-agents, skills, risk, shared Docker container, config, error handling).

---

## 13. Instructions for the New Claude Code Session

When starting the new session, provide these instructions:

```
I'm building a new project from scratch: an autonomous agent execution service
with a project board UI, policy engine, and external world watchers.

Reference documents (in docs/):
- factsheet.md — the complete product spec. Sections 1-12 cover product features
  (board, chat, collaboration, watchers, policy engine, sleep-time compute).
  Section 16 covers core agent architecture (agent loop, compaction, planning tool,
  tools, sub-agents, skills, risk, shared Docker container, config, error handling).
  Section 15 has the full DB schema (20 tables, UUID PKs). Section 17 has the system prompt.
- blueprint.md — implementation plan (repo structure, phases, tech stack, dependencies)

Start with Phase 1a from the blueprint. Build the foundation:
1a: repo scaffold + packages/shared types
1b: database schema (Drizzle, all 20 tables including users, traces, discussion_messages, cost_usage)
1c: API skeleton (Fastify routes, WebSocket server)
1d: agent loop + tools + shared Docker container + Postgres trace storage

All agents run in a single shared Docker container (workspace isolation via /workspace/{task_id}/ dirs).
Use in-memory injection queue for Phase 1 (no Redis required).
Follow the repo structure exactly as specified in the blueprint Section 3.
```

---

*End of Implementation Blueprint*
