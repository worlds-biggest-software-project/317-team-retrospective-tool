# Team Retrospective Tool — Phased Development Plan

> Project: 317-team-retrospective-tool · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three `data-model-suggestion-*.md` files into a concrete, phased implementation specification for an AI-native, self-hostable, open-source retrospective platform.

The product's heart is a **real-time collaborative retrospective board** that turns sprint feedback into tracked, measurable improvement. Differentiators (from research): cross-sprint trend analytics, reliable action-item carry-forward, AI sentiment/clustering/effectiveness scoring, cross-team pattern detection, and an MCP server for AI-agent access.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (strict) across frontend, backend, and CLI | Real-time collaborative UI is the core surface; a single language removes the FE/BE serialization seam and lets board state types be shared verbatim between server, WebSocket payloads, and React. Parabol (the only open-source incumbent, our reference) is TS/Node. |
| App framework | Next.js 16 (App Router, React Server Components) | Server-rendered board shell for fast first paint + SEO of marketing pages, with client components for the interactive board. Route handlers host the REST API and webhook receivers. Mobile-responsive requirement (features.md MVP) is met with one codebase. |
| Real-time transport | WebSocket (RFC 6455) via a dedicated Node `ws` gateway; SSE (WHATWG) fallback for read-only viewers | Boards require bidirectional messaging (card add, vote, phase change) — WebSocket per standards.md. SSE is the lighter path for dashboards/large read-only audiences. |
| Realtime fan-out | Redis Pub/Sub | Lets multiple WebSocket gateway instances broadcast board events to all connected clients in a session room without sticky sessions. |
| Database | PostgreSQL 16 | Strong relational integrity for the session→card→vote→action pipeline, native `JSONB` + GIN for variable board formats, `tsvector` full-text search for card search, `TIMESTAMPTZ` (ISO 8601). Single store covers OLTP + analytics at this scale. |
| Data model | **Hybrid Relational + JSONB** (data-model-suggestion-2) as the primary schema, with a lightweight `session_events` append log borrowed from suggestion-3 for facilitator timelines and AI interaction analysis | The Hybrid model loads a full board in one query with no joins — exactly what the WebSocket layer serves — and absorbs exotic formats (Sailboat, DAKI) with zero migrations. The pure event-sourced model (suggestion-3) is over-engineered for MVP read paths; we keep its best idea (an event log) as an additive audit/AI feed rather than the source of truth. The fully-normalized model (suggestion-1) forces a 6-table join for the hottest query. |
| ORM / query layer | Drizzle ORM | Type-safe schema-as-TS, first-class `jsonb` typing, generated SQL migrations, no heavy runtime. Schema types feed directly into shared board types. |
| Migrations | drizzle-kit | Versioned, reviewable SQL migrations checked into the repo. |
| Background jobs / queue | BullMQ (Redis-backed) | Async work: AI analysis calls, integration syncs, email/Slack notifications, carry-forward proposal generation, overdue-action sweeps. Reuses the Redis already present for pub/sub. |
| AI layer | Vercel AI SDK targeting Anthropic Claude (`claude-opus-4` for analysis, `claude-haiku` for cheap per-card sentiment); pluggable provider via env | Structured-output (`generateObject` + Zod) gives schema-validated theme clusters, sentiment, and effectiveness scores. Provider abstraction keeps the tool open-source-friendly (swap to local/OpenAI). |
| Auth (own) | Auth.js (NextAuth v5) with email magic-link + OIDC (OpenID Connect Core 1.0) | OIDC enables corporate SSO (Google Workspace, Entra ID, Okta) per standards.md. JWT sessions (RFC 7519). |
| Auth (integrations) | OAuth 2.0 (RFC 6749) authorization-code flow for Jira, GitHub, Slack, Teams | Delegated, scoped, credential-less access per standards.md. |
| Enterprise provisioning | SCIM 2.0 (RFC 7644) endpoint (v1.1, not MVP) | Auto user provisioning from Okta/Entra. |
| Frontend board interactions | dnd-kit (drag-and-drop), accessible by design | dnd-kit supports full keyboard DnD, addressing the WCAG 2.2 AA accessibility gap called out in features.md/standards.md. |
| UI components | shadcn/ui + Tailwind CSS | Accessible primitives (Radix), themeable, fast to build a polished board + dashboards. |
| Charts | Recharts | Trend dashboards (sentiment, action completion, health scores) per features.md v1.1. |
| API contract | OpenAPI 3.1 generated from Zod via `zod-to-openapi` | Machine-readable public API per standards.md; powers SDK generation + interactive docs. JSON Schema 2020-12 for payload validation. |
| MCP server | `@modelcontextprotocol/sdk` (TypeScript) | AI-agent access to trigger retros, query actions/trends — the "AI-native from the outset" positioning in standards.md/README. |
| Validation | Zod | One schema source reused for runtime validation, TS types, OpenAPI, and AI structured output. |
| Email | Nodemailer + React Email templates | Action reminders / notifications (features.md v1.1). Pluggable SMTP for self-hosters. |
| Export | `@react-pdf/renderer` (PDF), `csv-stringify` (CSV) | MVP export requirement. |
| Testing | Vitest (unit/integration), Playwright (E2E + accessibility via `@axe-core/playwright`) | Vitest is fast and native to the Vite/TS ecosystem; Playwright drives the real-time board and verifies WCAG. |
| Linting / formatting | ESLint (typescript-eslint) + Prettier | Standard TS quality gate. |
| Type checking | `tsc --noEmit` in CI | Strict mode enforced. |
| Package manager | pnpm (workspace monorepo) | Efficient, workspace-aware; cleanly separates `web`, `realtime`, `mcp`, `shared`, `worker`. |
| Containerisation | Docker + docker-compose (Postgres, Redis, web, realtime, worker) | Self-hosted deployment is a stated product pillar (README). One `docker-compose up` brings up the full stack. |
| CI | GitHub Actions | Lint, typecheck, unit+integration, migration check, Playwright, Docker build. |

### Project Structure

A pnpm monorepo. Code grouped by concern, additive across phases.

```
team-retrospective-tool/
├── package.json                      # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json                        # task pipeline (build/test/lint)
├── tsconfig.base.json
├── .env.example
├── docker-compose.yml                # postgres, redis, web, realtime, worker
├── Dockerfile                        # multi-stage; targets per app
├── .github/workflows/ci.yml
├── packages/
│   ├── shared/                       # cross-cutting, no runtime deps on apps
│   │   ├── src/
│   │   │   ├── types/                # Board, Card, Vote, ActionItem, Session, Format
│   │   │   ├── schemas/              # Zod schemas (source of truth for types+OpenAPI+AI)
│   │   │   ├── events/               # WebSocket message envelope + event payload types
│   │   │   ├── formats/              # built-in retro format definitions (SSC, 4Ls, etc.)
│   │   │   └── constants.ts
│   │   └── package.json
│   ├── db/                           # Drizzle schema, migrations, query helpers
│   │   ├── src/
│   │   │   ├── schema/               # one file per table group
│   │   │   ├── migrations/           # generated SQL
│   │   │   ├── client.ts
│   │   │   └── queries/              # reusable typed queries (boardBySession, etc.)
│   │   └── drizzle.config.ts
│   ├── ai/                           # AI service: sentiment, clustering, effectiveness
│   │   └── src/
│   │       ├── prompts/              # versioned prompt templates
│   │       ├── sentiment.ts
│   │       ├── clustering.ts
│   │       ├── effectiveness.ts
│   │       └── crossTeam.ts
│   └── integrations/                 # Jira / GitHub / Slack / Teams clients
│       └── src/{jira,github,slack,teams,oauth}.ts
├── apps/
│   ├── web/                          # Next.js 16 app (UI + REST API + webhooks)
│   │   ├── src/app/
│   │   │   ├── (marketing)/
│   │   │   ├── (app)/                # authed app shell
│   │   │   │   ├── teams/[teamId]/...
│   │   │   │   ├── retro/[sessionId]/board/   # the live board
│   │   │   │   └── analytics/
│   │   │   └── api/                  # route handlers: /api/v1/*, /api/webhooks/*, /api/auth/*
│   │   ├── src/components/board/     # Column, Card, VoteButton, ThemeGroup, PhaseBar
│   │   ├── src/components/analytics/
│   │   ├── src/lib/ws-client.ts      # WebSocket client + reconnect/replay
│   │   └── ...
│   ├── realtime/                     # standalone Node WebSocket gateway (ws + Redis)
│   │   └── src/{server.ts,rooms.ts,auth.ts,handlers/}
│   ├── worker/                       # BullMQ job processors
│   │   └── src/jobs/{aiAnalyze,integrationSync,notify,carryForward,overdueSweep}.ts
│   └── mcp/                          # MCP server exposing retro tools
│       └── src/{server.ts,tools/}
├── tests/
│   ├── e2e/                          # Playwright specs + fixtures
│   └── fixtures/                     # sample sessions, boards, sprint payloads
└── docs/
    └── openapi.json                  # generated
```

---

## Phase 1: Foundation — Monorepo, Database, Shared Types

### Purpose
Establish the buildable, testable skeleton: monorepo tooling, the PostgreSQL schema (Hybrid model + event log), shared Zod schemas/types, and the Docker dev stack. After this phase the app builds, the database migrates cleanly, and every later phase imports types from `packages/shared` and queries from `packages/db` without restructuring.

### Tasks

#### 1.1 — Monorepo & toolchain bootstrap

**What**: Create the pnpm/turbo workspace with TypeScript strict config, ESLint, Prettier, and Vitest wired across packages.

**Design**:
- `pnpm-workspace.yaml` lists `packages/*` and `apps/*`.
- `tsconfig.base.json`: `strict: true`, `noUncheckedIndexedAccess: true`, `target: ES2022`, `moduleResolution: bundler`, path alias `@retro/*` → `packages/*/src`.
- `turbo.json` pipeline: `build` depends on `^build`; `test`, `lint`, `typecheck` defined; `test` depends on `^build`.
- Root scripts: `dev`, `build`, `test`, `lint`, `typecheck`, `db:migrate`, `db:generate`.
- `.env.example` with: `DATABASE_URL`, `REDIS_URL`, `AUTH_SECRET`, `ANTHROPIC_API_KEY`, `AI_PROVIDER`, `APP_URL`, `WS_URL`.

**Testing**:
- `Unit: pnpm -r typecheck → exits 0 on empty packages`
- `Unit: pnpm lint → exits 0`
- `Integration: pnpm build → all packages build with no missing-reference errors`

#### 1.2 — Database schema (Drizzle) — Hybrid model

**What**: Implement the Hybrid Relational + JSONB schema from data-model-suggestion-2 as Drizzle table definitions, plus an additive `session_events` log.

**Design**:
Tables (`packages/db/src/schema/`), matching suggestion-2 with the noted additions:

- `workspaces(id, name, slug UNIQUE, settings JSONB, created_at, updated_at)` — `settings` holds `billing_plan`, `features[]`, `branding`, `integrations.{jira,slack,github,teams}`, `scim`.
- `users(id, workspace_id FK, email UNIQUE, name, role CHECK(admin|facilitator|member), avatar_url, timezone, is_active, …)`.
- `teams(id, workspace_id FK, name, slug, description, members JSONB, …, UNIQUE(workspace_id, slug))` — `members: [{user_id, role, joined_at}]`; GIN index on `members`.
- `retro_sessions(id, team_id FK, facilitator_id FK, title, sprint_name, status CHECK(draft|collecting|voting|discussing|actions|completed), format JSONB, settings JSONB, themes JSONB, ai JSONB, scheduled_at, started_at, completed_at, …)`.
- `retro_cards(id, session_id FK, author_id FK, column_key, content, theme_id TEXT, sort_order, votes JSONB, ai JSONB, …)` — GIN on `votes`, GIN tsvector on `content`.
- `action_items(id, session_id FK, team_id FK, card_id FK?, title, description, owner_id FK?, status CHECK(pending|in_progress|completed|cancelled|carried_forward), priority CHECK(high|medium|low), due_date, carry_forward JSONB, external JSONB, completed_at, …)`.
- `health_checks(id, session_id FK?, team_id FK, template JSONB, is_anonymous, responses JSONB, summary JSONB, created_at)`.
- **Added** `session_events(id, session_id FK, event_type, actor_id, phase, data JSONB, occurred_at)` — append-only feed for facilitator timeline + AI interaction analysis; index `(session_id, occurred_at)`. Not the source of truth.
- **Added** `api_keys(id, workspace_id FK, user_id FK, name, key_hash UNIQUE, key_prefix, scopes TEXT[], is_active, created_at)` — for public REST/MCP auth.
- **Added** `notifications_log(id, workspace_id, channel, payload JSONB, status, sent_at)` — dedupe + audit of outbound notifications.

Drizzle JSONB columns are typed via `.$type<FormatDoc>()` etc., importing types from `packages/shared`.

**Testing**:
- `Integration (real Postgres via docker-compose): drizzle migrate → all tables created, all CHECK constraints present`
- `Integration: insert workspace+user+team → FK cascade on workspace delete removes children`
- `Unit: invalid role value insert → DB rejects with check_violation`
- `Integration: GIN index on retro_cards.content → full-text query returns matching card`

#### 1.3 — Shared schemas & types

**What**: Define Zod schemas in `packages/shared` for every domain entity and derive TS types; define the WebSocket event envelope.

**Design**:
```ts
// packages/shared/src/schemas/format.ts
export const ColumnSchema = z.object({
  key: z.string(), label: z.string(), prompt: z.string().optional(),
  color: z.string().regex(/^#[0-9A-Fa-f]{6}$/).optional(),
  zone: z.enum(['positive','negative','risk','goal','neutral']).default('neutral'),
});
export const FormatSchema = z.object({
  name: z.string(), columns: z.array(ColumnSchema).min(1),
  icon: z.string().optional(), layout: z.enum(['columns','quadrant']).default('columns'),
});
export type Format = z.infer<typeof FormatSchema>;

// packages/shared/src/schemas/session.ts
export const SessionStatus = z.enum(['draft','collecting','voting','discussing','actions','completed']);
export const SessionSettingsSchema = z.object({
  isAnonymous: z.boolean().default(true),
  maxVotesPerUser: z.number().int().min(0).default(5),
  timerSeconds: z.number().int().positive().optional(),
  allowMultipleVotesPerCard: z.boolean().default(false),
  showAuthorAfterReveal: z.boolean().default(false),
  cardCharacterLimit: z.number().int().positive().default(500),
});

// packages/shared/src/events/envelope.ts
export const WsEnvelopeSchema = z.object({
  v: z.literal(1),
  type: z.string(),              // e.g. 'card.added'
  sessionId: z.string().uuid(),
  actorId: z.string().uuid().nullable(),
  ts: z.string().datetime(),     // ISO 8601
  payload: z.unknown(),
});
export type WsEnvelope = z.infer<typeof WsEnvelopeSchema>;
```
Built-in formats live in `packages/shared/src/formats/index.ts` as `Format` objects: Start/Stop/Continue, 4Ls (Liked/Learned/Lacked/Longed-for), Glad/Sad/Mad, Sailboat (quadrant), DAKI (Drop/Add/Keep/Improve).

**Testing**:
- `Unit: parse valid Start/Stop/Continue format → ok, 3 columns`
- `Unit: Sailboat format → layout 'quadrant', 4 columns with zones`
- `Unit: SessionSettings with no fields → defaults applied (anonymous true, 5 votes)`
- `Unit: column color 'red' → ZodError (must be hex)`

#### 1.4 — Docker dev stack

**What**: `docker-compose.yml` bringing up Postgres 16, Redis 7, and placeholders for web/realtime/worker; a `Dockerfile` multi-stage build.

**Design**:
- Services: `postgres` (volume, healthcheck `pg_isready`), `redis`, `web` (port 3000), `realtime` (port 3001), `worker`.
- `Dockerfile` stages: `deps` (pnpm install) → `build` (turbo build) → per-app runtime targets.
- `web`/`realtime`/`worker` `depends_on` postgres+redis healthchecks.
- Entrypoint for `web` runs `drizzle migrate` then starts Next.

**Testing**:
- `Integration: docker-compose up postgres redis → both healthy within 30s`
- `E2E (CI): docker build --target web → image builds`
- `Integration: web container starts → migrations applied, /api/health returns 200`

---

## Phase 2: Auth, Workspaces, Teams

### Purpose
Stand up identity and tenancy so every later resource is scoped to a workspace/team with role-based access (OWASP A01). After this phase a user can sign in, a workspace exists, teams can be created, and membership/roles are enforced.

### Tasks

#### 2.1 — Authentication (Auth.js)

**What**: Email magic-link + OIDC sign-in producing JWT sessions; user provisioned into a workspace on first login.

**Design**:
- Auth.js v5 in `apps/web`, providers: `Email` (magic link via Nodemailer) and a generic `OIDC` provider configurable by env (`OIDC_ISSUER`, `OIDC_CLIENT_ID/SECRET`).
- Session strategy: JWT (RFC 7519), 7-day expiry, contains `userId`, `workspaceId`, `role`.
- First sign-in flow: if email domain matches an existing workspace's claimed domain → join as `member`; else create a new workspace + `admin` user.
- `getServerSession()` helper returns typed `{ userId, workspaceId, role }`.

**Testing**:
- `Integration (mocked SMTP): request magic link → email enqueued with signed token`
- `Integration: valid token callback → session JWT issued, user row created`
- `Unit: expired token → 401, no session`
- `Integration (mocked OIDC): id_token with email → user mapped to workspace`

#### 2.2 — RBAC & tenancy middleware

**What**: Authorization layer enforcing workspace isolation and role checks on every API route and server action.

**Design**:
```ts
type Principal = { userId: string; workspaceId: string; role: 'admin'|'facilitator'|'member' };
function authorize(p: Principal, resource: { workspaceId: string }, need: Permission): void; // throws 403
// Permissions: 'team.create','session.facilitate','session.view','action.edit','workspace.admin'
```
- Every query helper in `packages/db/queries` takes `workspaceId` and includes it in the `WHERE` clause — no cross-tenant read possible.
- Team-level role resolved from `teams.members` JSONB.
- Maps to OWASP A01 (broken access control) — tested explicitly.

**Testing**:
- `Unit: member requests workspace.admin → throws 403`
- `Integration: user from workspace A queries session in workspace B → 404 (not 403, to avoid existence leak)`
- `Unit: facilitator on team X has session.facilitate; member does not`

#### 2.3 — Workspace & team management API + UI

**What**: REST endpoints and UI for creating/editing workspaces, teams, and managing members.

**Design**:
REST (`/api/v1`), all Zod-validated, all OpenAPI-documented:
- `POST /workspaces` (admin bootstrap), `PATCH /workspaces/:id` (branding, features).
- `POST /teams` → `{name, slug, description}`; `GET /teams`; `PATCH /teams/:id`.
- `POST /teams/:id/members` → `{userId, role}`; `DELETE /teams/:id/members/:userId`.
UI: `/teams` list, team settings page, member table with role dropdowns.

**Testing**:
- `Integration: POST /teams duplicate slug in workspace → 409`
- `Integration: add member → teams.members JSONB appended, GIN-queryable`
- `E2E: admin creates team, adds member, member appears in roster`

---

## Phase 3: The Retrospective Board (core value, single-user)

### Purpose
Build the heart of the product: a session with a chosen format, columns, cards, voting, and facilitation phases — first as standard request/response (no real-time yet) so the data model and rules are proven before layering WebSockets. After this phase a facilitator can run a complete retro solo.

### Tasks

#### 3.1 — Session lifecycle & format selection

**What**: Create a session from a built-in or custom format and advance it through phases.

**Design**:
- `POST /teams/:id/sessions` → `{title?, sprintName?, formatRef | format, settings?}`. `formatRef` is a built-in key; otherwise inline `Format`. The chosen `format` is **copied** into `retro_sessions.format` (suggestion-2 decision #1: format is preserved even if template later changes).
- Phase state machine on `retro_sessions.status`:
  `draft → collecting → voting → discussing → actions → completed` (forward-only; facilitator may step back one phase except out of `completed`).
- `POST /sessions/:id/advance` and `/back`; each transition writes a `session.phase_changed` row to `session_events`.
- Scrum Guide alignment: optional `timerSeconds` per phase; default suggested timebox surfaced in UI.

**Testing**:
- `Unit: advance from collecting → voting; advancing from completed → error`
- `Unit: create session with formatRef 'sailboat' → format JSONB has 4 quadrant columns`
- `Integration: each advance writes a session_events row`

#### 3.2 — Cards CRUD with column & anonymity rules

**What**: Add/edit/delete/move cards within a session's columns, honoring the character limit and anonymity setting.

**Design**:
- `POST /sessions/:id/cards` → `{columnKey, content}`; rejects if `columnKey` not in `format.columns` (400) or `content` exceeds `settings.cardCharacterLimit`.
- `PATCH /cards/:id` (author or facilitator only), `DELETE /cards/:id`, `POST /cards/:id/move` → `{columnKey, sortOrder}`.
- **Anonymity (GDPR Art. 25 + suggestion-2 decision):** `author_id` is always stored, but the **serializer** strips it from API responses when `settings.isAnonymous && !settings.showAuthorAfterReveal`. Anonymity is a projection concern, never a storage one — preserves vote-limit enforcement and right-to-deletion.
- Each mutation appends a `card.*` row to `session_events`.

**Testing**:
- `Unit: card to unknown column → 400`
- `Unit: content over limit → 400 with limit in message`
- `Integration: anonymous session → GET board omits author_id; non-anonymous → includes it`
- `Integration: member edits another member's card → 403`
- `Integration: card content indexed → full-text search finds it`

#### 3.3 — Voting with limits

**What**: Cast/remove votes with per-user limits enforced server-side.

**Design**:
- `POST /cards/:id/vote`, `DELETE /cards/:id/vote`. Vote stored as entry in `retro_cards.votes` JSONB `[{userId, votedAt}]`.
- Enforcement (application layer, per suggestion-2 trade-off): on cast, count user's votes across the session's cards; reject if `>= maxVotesPerUser` (400 `VOTE_LIMIT_REACHED`). If `!allowMultipleVotesPerCard`, reject a second vote on the same card.
- Vote actions only allowed in `voting` phase (or `collecting` if `settings` permits early voting — default no).
- `voteCount` derived as `votes.length` in serializer.

**Testing**:
- `Unit: 6th vote with limit 5 → 400 VOTE_LIMIT_REACHED`
- `Unit: double vote on same card, multiple disallowed → 400`
- `Integration: vote then remove → count decrements, budget restored`
- `Unit: vote during collecting phase (early voting off) → 400`

#### 3.4 — Themes (grouping)

**What**: Facilitator creates themes and assigns cards to them during discussion.

**Design**:
- Themes are entries in `retro_sessions.themes` JSONB `[{id, title, color, sortOrder}]`. Cards reference via `retro_cards.theme_id` (string).
- `POST /sessions/:id/themes`, `PATCH`, `DELETE` (delete ungroups its cards), `POST /cards/:id/group` → `{themeId|null}`.
- Facilitator-only.

**Testing**:
- `Unit: delete theme → member cards' theme_id cleared`
- `Integration: group card → board groups cards under theme, ordered by sortOrder`
- `Unit: member attempts create theme → 403`

#### 3.5 — Board read + facilitation UI

**What**: The single-query board read and the React board UI (columns, cards, votes, phases) with accessible drag-and-drop.

**Design**:
- `GET /sessions/:id/board` → one query (suggestion-2 "full board" query): session `format/themes/settings/ai` + `json_agg` of cards (author stripped per anonymity), ordered by column then vote count.
- UI components: `PhaseBar` (current phase + advance), `Column`, `Card`, `VoteButton`, `ThemeGroup`, `Timer`. dnd-kit for card move/group with **full keyboard support** (WCAG 2.2: arrow-key move, space to pick up/drop) and `@axe-core` checks.
- Responsive: columns stack on mobile (features.md MVP requirement).

**Testing**:
- `Integration: board query for 50-card session → single SQL round-trip`
- `E2E (Playwright): create session → add cards → vote → advance phases → see results`
- `E2E (axe): board page → no WCAG 2.2 AA violations`
- `E2E (keyboard): move a card between columns using only keyboard`

---

## Phase 4: Real-Time Collaboration

### Purpose
Make the board multi-user and live. The WebSocket gateway broadcasts every mutation so concurrent participants see cards, votes, and phase changes instantly, with replay for late joiners. This is the capability that turns a CRUD app into a retrospective tool. Depends on Phase 3.

### Tasks

#### 4.1 — WebSocket gateway

**What**: Standalone `apps/realtime` Node service authenticating connections and managing per-session rooms via Redis pub/sub.

**Design**:
- `ws` server on port 3001. Client connects to `wss://…/session/:id?token=<jwt>`.
- Auth: verify the Auth.js JWT; reject if user lacks `session.view` on the session's team (401/403). Maps to OWASP A01/A07.
- Each session is a Redis pub/sub channel `room:session:<id>`. Gateway instances subscribe; any instance can publish, so no sticky sessions needed.
- Message envelope = `WsEnvelopeSchema` (Phase 1.3). Server validates inbound, applies via the same service functions as the REST layer (single source of mutation logic), persists, then publishes the resulting event to the room.
- Presence: `presence.joined`/`presence.left` with anonymized display (or name if non-anonymous).

**Testing**:
- `Integration (mocked JWT): connect with valid token → 101 upgrade; invalid → closed with 4401`
- `Integration: two clients in a room; client A adds card → client B receives card.added within 200ms`
- `Integration: connection without session.view → rejected`
- `Integration: publish via instance 1 → subscriber on instance 2 receives it (Redis fan-out)`

#### 4.2 — Shared mutation service & event broadcast

**What**: Extract all board mutations (card/vote/theme/phase) into `packages/db`-backed service functions invoked by both REST and WebSocket, each emitting a typed event.

**Design**:
- `applyMutation(principal, command): { state, event }`. Commands mirror REST bodies. Every mutation: (1) authorize, (2) validate against current phase/limits, (3) persist (card row, votes JSONB, etc.), (4) append `session_events` row, (5) return event for broadcast.
- REST handlers call `applyMutation` then return state; WS handlers call it then `publish(event)`.
- Optimistic concurrency on `retro_sessions.updated_at` for phase/theme edits to prevent lost updates.

**Testing**:
- `Unit: applyMutation(vote) over limit → throws VoteLimitError (same path REST+WS)`
- `Integration: REST add + WS add produce identical board state`
- `Unit: stale updated_at on phase advance → ConflictError`

#### 4.3 — Client real-time sync & replay

**What**: Browser WebSocket client with reconnect, event application to local store, and replay for late joiners.

**Design**:
- `apps/web/src/lib/ws-client.ts`: connects, applies inbound events to a client store (Zustand), sends mutations optimistically with rollback on server reject.
- Late-joiner replay: on connect the client fetches `GET /sessions/:id/board` (snapshot) then subscribes; the gateway sends events with monotonically increasing per-session sequence so the client discards events older than the snapshot.
- Reconnect with exponential backoff; on reconnect, re-fetch snapshot to resync.

**Testing**:
- `E2E (two browser contexts): user B joins mid-session → sees all existing cards + live updates`
- `Integration: drop connection, reconnect → board resyncs to current state`
- `Unit: optimistic vote rejected by server → UI rolls back`

---

## Phase 5: Action Items & Carry-Forward

### Purpose
Close the gap incumbents leave open (research.md): actions that fall through the cracks. Implement action items with owners/due dates and reliable automatic carry-forward to the next sprint — the TeamRetro-grade differentiator. Depends on Phase 3.

### Tasks

#### 5.1 — Action item CRUD

**What**: Create actions (optionally from a card/theme) with owner, due date, priority, status.

**Design**:
- `POST /sessions/:id/actions` → `{title, description?, cardId?, ownerId?, dueDate?, priority?}`. Created during `actions`/`discussing` phases.
- `PATCH /actions/:id` (status, owner, due, priority). Status machine: `pending → in_progress → completed | cancelled`, plus `carried_forward` set by the carry-forward job. `completed_at` set on completion.
- `GET /teams/:id/actions?status=&owner=&overdue=` — the team's action backlog across sessions.

**Testing**:
- `Unit: action from card → card_id linked`
- `Unit: complete action → status completed, completed_at set`
- `Integration: list overdue actions → returns past-due, non-terminal actions`

#### 5.2 — Carry-forward

**What**: When a new session starts, propose uncompleted actions from the team's previous session; carrying one records its chain.

**Design**:
- On session creation, query prior session's non-terminal actions → return as `proposedCarryForward`.
- `POST /sessions/:id/actions/carry` → `{actionIds[]}`. For each: create a new `action_items` row in the new session, set `carry_forward` JSONB (suggestion-2 decision #4): append `{sessionId, sprintName, status, createdAt}` to `chain`, set `original_session_id`, increment `times_carried`; mark the source action `carried_forward`.
- Surfaces "carried N times" in UI to flag chronically deferred actions.

**Testing**:
- `Integration: incomplete action + new session → appears in proposals`
- `Unit: carry action → new row, times_carried incremented, chain appended, source marked carried_forward`
- `Integration: carry an already-carried action → times_carried = 2, chain length 2`
- `Unit: completed action → not proposed for carry-forward`

#### 5.3 — Notifications (email + queued)

**What**: Owner assignment and overdue reminders via email, queued through BullMQ.

**Design**:
- Worker jobs: `notify.actionAssigned` (on owner set), `overdueSweep` (cron, daily) → enqueues `notify.actionOverdue` per overdue action.
- React Email templates; Nodemailer transport (SMTP from env). All sends logged to `notifications_log` with dedupe key `(actionId, kind, date)`.

**Testing**:
- `Integration (mocked SMTP): assign owner → actionAssigned email enqueued + logged`
- `Integration: overdueSweep with one overdue action → one reminder, second run same day → deduped (no resend)`

---

## Phase 6: AI Layer

### Purpose
Deliver the AI-native advantages from research.md: per-card sentiment, automatic theme clustering, and action-effectiveness scoring. Built as queued, schema-validated analyses stored on the entities they describe. Depends on Phases 3 and 5.

### Tasks

#### 6.1 — AI service & structured output

**What**: `packages/ai` wrapping the AI SDK with provider abstraction and Zod-validated structured outputs.

**Design**:
```ts
// provider-agnostic; default Anthropic Claude
async function analyzeSentiment(cards: {id:string;content:string}[]): Promise<CardSentiment[]>;
async function clusterCards(cards: {...}[]): Promise<SuggestedTheme[]>; // {title, cardIds, confidence}
async function scoreActionEffectiveness(team: TeamActionHistory): Promise<EffectivenessReport>;
```
- All via `generateObject({ schema: <Zod>, … })`, so malformed model output throws and is retried (BullMQ retry policy).
- Prompt templates versioned in `packages/ai/src/prompts/*` with a `model_version` recorded into the stored `ai` JSONB.
- Sentiment prompt (structure): system = "Classify each retrospective card's sentiment …"; returns `{cardId, sentiment: positive|neutral|negative, score: -1..1, keywords[]}`.
- Clustering prompt: input the column-bucketed card list; returns groups with `confidence`.

**Testing**:
- `Unit (mocked provider): sentiment returns schema-valid objects → parsed`
- `Unit (mocked): model returns invalid JSON → throws, job retried`
- `Unit: clustering output cardIds all exist in input set (validation)`

#### 6.2 — AI analysis jobs & board integration

**What**: Trigger analyses at phase transitions; store results in `ai` JSONB; surface in board UI.

**Design**:
- On `collecting → voting`: enqueue `aiAnalyze.sentiment` (per card → `retro_cards.ai`) and `aiAnalyze.cluster` (→ `retro_sessions.ai.suggested_themes`).
- On `actions`: enqueue `aiAnalyze.effectiveness` using the team's historical actions → `retro_sessions.ai.action_effectiveness`.
- Results broadcast via the WebSocket gateway (`ai.themes_suggested`, `ai.sentiment_analyzed`) so the live board updates. Facilitator can accept a suggested theme to create a real theme (Phase 3.4) in one click.

**Testing**:
- `Integration (mocked AI): advance to voting → cards get ai.sentiment, session gets suggested_themes`
- `Integration: accept suggested theme → real theme created, cards grouped`
- `Integration: AI job failure → board still usable, ai field absent (graceful degradation)`

---

## Phase 7: Integrations (OAuth: Jira, GitHub, Slack)

### Purpose
Connect the tool to the systems teams already use: pull sprint context, push action items as issues, and send notifications — addressing the "siloed by ecosystem" gap (research.md). Depends on Phase 5 (actions) and Phase 2 (workspace settings). Slack/Jira/GitHub clients can be built in parallel.

### Tasks

#### 7.1 — OAuth 2.0 connection framework

**What**: Generic authorization-code flow storing encrypted per-provider credentials in `workspaces.settings.integrations`.

**Design**:
- `GET /api/integrations/:provider/connect` → redirect to provider authorize URL (state = signed CSRF token).
- `GET /api/integrations/:provider/callback` → exchange code, encrypt tokens (AES-256-GCM, key from env), store under `settings.integrations.<provider>` (suggestion-2 decision #7).
- Token refresh helper per provider; `DELETE /api/integrations/:provider` to disconnect.

**Testing**:
- `Integration (mocked provider): callback with valid code → encrypted tokens stored`
- `Unit: tampered state param → 400`
- `Unit: stored credentials are ciphertext (not plaintext) at rest`

#### 7.2 — Slack notifications (Block Kit app)

**What**: Post retro reminders and action notifications to Slack via a modern Slack App (not legacy webhooks — discontinued Nov 2026 per standards.md).

**Design**:
- Bot-token posting with Block Kit messages: session-scheduled reminder, session-completed summary, action-overdue with an interactive "Mark done" button (interaction endpoint `/api/webhooks/slack`).
- Channel configurable in `settings.integrations.slack.channel`.

**Testing**:
- `Integration (mocked Slack API): session completed → Block Kit summary posted to channel`
- `Integration: "Mark done" button payload → action status updated, signature verified`
- `Unit: invalid Slack signature → 401`

#### 7.3 — Jira integration (sprint context + issue creation)

**What**: Pull sprint metrics for context and create action items as native Jira issues.

**Design**:
- Jira Cloud REST v3 via OAuth (3LO). `GET /sessions/:id/jira/sprint-context` → velocity, completed vs. spilled, from `/rest/agile/1.0/sprint/{id}`; displayed as read-only cards in the board's intro.
- `POST /actions/:id/push/jira` → create issue (ADF description), store `action_items.external.jira = {issueKey, url, syncedAt}` (suggestion-2 decision: external links JSONB).

**Testing**:
- `Integration (mocked Jira): push action → issue created, external.jira populated`
- `Integration (mocked Jira): fetch sprint context → metrics returned`
- `Unit: push when Jira not connected → 409 with connect prompt`

#### 7.4 — GitHub integration

**What**: Pull issues/PR metrics for context and create actions as GitHub issues.

**Design**:
- GitHub App (JWT + installation token, standards.md). `POST /actions/:id/push/github` → issue in configured repo, store `external.github`.
- Optional: pull recent merged-PR/cycle-time stats as sprint context cards.

**Testing**:
- `Integration (mocked Octokit): push action → issue created, external.github populated`
- `Unit: repo not configured → 409`

---

## Phase 8: Analytics & Team Health

### Purpose
Deliver the trend insight that incumbents gate behind premium tiers (research.md): cross-sprint sentiment/topic/action-completion trends, team health checks, and a dashboard. Depends on Phases 5 (actions), 6 (sentiment), and the health-check schema.

### Tasks

#### 8.1 — Team health checks

**What**: Run a health-check survey (e.g., Spotify 10-dimension) per team/session and compute a summary.

**Design**:
- `POST /teams/:id/health-checks` → `{template}` (built-in templates in shared); `POST /health-checks/:id/responses` → `{scores:{dim:{score,trend,comment?}}}` appended to `responses` JSONB (suggestion-2 decision #5). Anonymous by default (GDPR).
- On close, compute `summary` JSONB: per-dimension averages, respondent count, lowest/highest, `trend_vs_last` vs. previous check.

**Testing**:
- `Unit: submit responses → appended to responses JSONB`
- `Unit: summary computes correct per-dimension averages`
- `Integration: anonymous check → responses never expose user_id in API`

#### 8.2 — Trend analytics queries & dashboard

**What**: Cross-sprint trend computation and a Recharts dashboard (sentiment, action completion rate, health score, retro frequency).

**Design**:
- `GET /teams/:id/analytics?from=&to=` aggregates: action completion rate (suggestion-2 query), avg card sentiment per session, health-check dimension trends, recurring-topic frequency (group cards by AI keyword/theme across sessions).
- Dashboard page with line/bar charts; "chronically carried" actions highlighted; lowest health dimension flagged.

**Testing**:
- `Integration: team with 3 sessions → completion-rate trend has 3 points`
- `Integration: health trend → per-dimension series across checks with trend_vs_last`
- `E2E: analytics page renders charts for a seeded team`

#### 8.3 — Cross-team pattern detection

**What**: AI detection of systemic issues recurring across multiple teams, escalated to workspace admins (the standout gap from research.md — no incumbent does this).

**Design**:
- Scheduled worker `aiAnalyze.crossTeam` (weekly): collect top themes/keywords across all teams' recent sessions → `clusterCrossTeam` returns shared patterns with affected team list and confidence. Stored in a `workspace`-scoped analysis and surfaced on an admin "Org Insights" view.

**Testing**:
- `Integration (mocked AI): two teams both mention "deployment friction" → cross-team pattern with both teams listed`
- `Unit: single-team theme → not escalated`

---

## Phase 9: Export, Public API & MCP Server

### Purpose
Make retro outcomes portable and programmable: PDF/CSV export (MVP), a documented OpenAPI 3.1 REST API, and an MCP server enabling AI agents to drive the tool — the AI-native positioning. Depends on core data being complete (Phases 3, 5, 8).

### Tasks

#### 9.1 — Export (PDF + CSV)

**What**: Export a completed session and an action backlog.

**Design**:
- `GET /sessions/:id/export?format=pdf|csv`. PDF (`@react-pdf/renderer`): session summary, columns/cards, top-voted, themes, action items, health summary. CSV (`csv-stringify`): one row per card and per action.

**Testing**:
- `Integration: export pdf → returns application/pdf, non-empty`
- `Integration: export csv → header row + one row per card/action`
- `Unit: anonymous session export → no author names`

#### 9.2 — Public REST API + OpenAPI 3.1

**What**: API-key-authenticated public endpoints with a generated OpenAPI 3.1 spec.

**Design**:
- API keys (`api_keys` table): `POST /workspaces/:id/api-keys` returns the key once; stored hashed (Argon2). Auth via `Authorization: Bearer <key>`, scope-checked.
- Public surface: sessions (read), actions (read/update status), teams, analytics (read).
- `zod-to-openapi` generates `docs/openapi.json` at build; served at `/api/docs` (Scalar/Swagger UI). JSON Schema 2020-12.

**Testing**:
- `Integration: request with valid key+scope → 200; missing scope → 403`
- `Unit: api key stored hashed, prefix retained for display`
- `Integration (CI): generated openapi.json validates against OpenAPI 3.1 schema`

#### 9.3 — MCP server

**What**: `apps/mcp` exposing retro capabilities to AI agents via Model Context Protocol.

**Design**:
- Tools: `list_teams`, `start_retro(teamId, formatRef)`, `get_action_items(teamId, status?)`, `update_action_status(actionId, status)`, `get_team_trends(teamId)`, `add_card(sessionId, columnKey, content)`.
- Auth via API key (env or per-request). Reuses `packages/db` queries and the same `authorize` checks — agents are principals too.

**Testing**:
- `Integration (MCP client harness): list_teams → returns workspace teams`
- `Integration: start_retro → session created in DB`
- `Unit: update_action_status with bad status → MCP error response`

---

## Phase 10: Hardening — Security, Accessibility, GDPR, Deploy

### Purpose
Production-readiness: security per OWASP Top 10, WCAG 2.2 AA accessibility, GDPR data-subject controls, and a one-command self-hosted deployment. Depends on all prior phases.

### Tasks

#### 10.1 — Security pass (OWASP Top 10)

**What**: Systematic checks for A01 (access control), A02 (crypto), A03 (injection), A07 (auth).

**Design**:
- A01: automated tenancy test suite — every endpoint attempted cross-workspace must 404. A02: verify integration tokens + API keys encrypted/hashed at rest; HTTPS-only cookies; secure JWT signing. A03: all inputs Zod-validated; Drizzle parameterized queries only (no string SQL); sanitize card/comment rendering (React escapes; verify no `dangerouslySetInnerHTML`). A07: magic-link expiry, rate-limited auth endpoints, brute-force lockout.
- Rate limiting middleware (Redis token bucket) on auth + public API.

**Testing**:
- `Integration: cross-tenant access matrix → all 404`
- `Unit: SQL-injection payload in card content → stored literally, no error`
- `Integration: 11 auth attempts/min → 429`

#### 10.2 — Accessibility (WCAG 2.2 AA)

**What**: Site-wide accessibility conformance, focusing on the board.

**Design**:
- Keyboard-operable DnD (dnd-kit keyboard sensor), visible focus (2.4.11/2.4.13), labelled controls, color-contrast AA, screen-reader announcements for live card/vote updates (`aria-live`), accessible authentication (no cognitive-test CAPTCHA — 3.3.8).

**Testing**:
- `E2E (axe-core/playwright) on board, dashboard, auth, action list → zero AA violations`
- `E2E: full retro completable with keyboard only`
- `E2E: screen-reader live region announces new card`

#### 10.3 — GDPR data-subject controls

**What**: Data export and erasure honoring anonymity guarantees.

**Design**:
- `POST /users/:id/export` → all personal data (authored cards de-anonymized only to the subject, health responses, actions owned).
- `POST /users/:id/erase` → crypto-erase/redact: replace `author_id` references with a tombstone, null personal fields; anonymous feedback content retained (de-identified) per legitimate-interest, ensuring no de-anonymisation by metadata (standards.md note). Configurable retention policy per workspace.

**Testing**:
- `Integration: erase user → authored cards' author_id tombstoned, name removed, anonymous content preserved`
- `Integration: export user → returns their cards/actions/health responses`
- `Unit: erased user cannot be re-identified via card timestamps in API output`

#### 10.4 — Deployment & docs

**What**: Production docker-compose, env documentation, self-host guide, CI gating.

**Design**:
- `docker-compose.prod.yml` (web behind reverse proxy, realtime, worker, postgres, redis, healthchecks, restart policies). `make up` / `make migrate`.
- CI gate: lint + typecheck + unit/integration + migration-applies-clean + Playwright (incl. axe) + Docker build must pass to merge.
- `docs/`: self-hosting guide, env reference, OpenAPI link, integration setup.

**Testing**:
- `E2E (CI): docker-compose.prod up → full retro flow over real WS against containers`
- `Integration: fresh DB → migrations apply with zero drift (drizzle-kit check)`
- `CI: all gates pass`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (monorepo, DB, shared types, Docker)   ─── required by everything
    │
Phase 2: Auth, Workspaces, Teams                           ─── requires 1
    │
Phase 3: Retrospective Board (single-user core)            ─── requires 2
    ├── Phase 4: Real-Time Collaboration                   ─── requires 3
    └── Phase 5: Action Items & Carry-Forward              ─── requires 3   (parallel with 4)
         │
         ├── Phase 6: AI Layer                             ─── requires 3,5
         ├── Phase 7: Integrations (Jira/GitHub/Slack)     ─── requires 5,2 (7.2/7.3/7.4 parallel)
         └── Phase 8: Analytics & Team Health              ─── requires 5,6
              │
Phase 9: Export, Public API, MCP                           ─── requires 3,5,8
    │
Phase 10: Hardening (security, a11y, GDPR, deploy)         ─── requires all
```

Parallelism:
- **Phase 4 (real-time) and Phase 5 (actions)** can be built concurrently once Phase 3 lands.
- Within **Phase 7**, the Slack, Jira, and GitHub clients are independent once 7.1 (OAuth framework) exists.
- **Phase 6 (AI)** and **Phase 7 (integrations)** can proceed in parallel.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks implemented to the design specified above.
2. All unit and integration tests pass (`pnpm test`).
3. ESLint and Prettier pass with no errors (`pnpm lint`).
4. `tsc --noEmit` passes in strict mode across the workspace (`pnpm typecheck`).
5. Drizzle migrations apply cleanly to a fresh database and `drizzle-kit check` reports no schema drift.
6. The phase's feature works end-to-end (verified by at least one Playwright E2E or integration flow).
7. New API endpoints appear in the generated `docs/openapi.json` (from Phase 9 onward; before that, Zod schemas exist).
8. New configuration/env vars are added to `.env.example` and documented.
9. New WebSocket event types are added to `packages/shared/src/events` and validated by `WsEnvelopeSchema`.
10. `docker build` succeeds for any app affected by the phase.
11. For UI work: no new WCAG 2.2 AA violations (`@axe-core/playwright`).
```
