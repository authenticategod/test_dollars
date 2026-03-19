# Workspace

## Overview

pnpm workspace monorepo using TypeScript. Each package manages its own dependencies.

## Project: Dollars Anonymous

Anonymous invite-only community platform inspired by the Dollars from Durarara!! anime. Members join via invite codes with a chosen @usertag (min 5 chars; 4 for owner). Features anonymous auth, invite system, and an admin panel.

### Architecture

- **Frontend**: React + Vite at root path `/`, dark atmospheric UI with framer-motion animations
- **Backend**: Express 5 API server with security hardening (helmet, rate limiting, CORS, timing-safe auth)
- **Database**: PostgreSQL with Drizzle ORM — tables: `invite_codes`, `members`, `sessions`, `rooms`, `messages`, `reactions`
- **Auth**: Token-based sessions (32-byte random hex), no PII stored, anonymous-first design
- **Admin**: Secret-based auth via `X-Admin-Secret` header, accessible at `/admin`

### Key Features (Task #1 — Foundation)

- Dark landing page with glitch-text "DOLLARS" heading and live member count
- Multi-step join flow: enter invite code → pick @usertag → pick display name → lobby
- Real-time usertag availability checking with debounced API calls
- Owner auto-assignment: first member to join becomes owner (gets crown badge, 4-char tag privilege)
- Early member badge: first 100 members
- Admin panel: generate invite codes, revoke codes, send email invites, view member ledger
- Invite link sharing: `/?code=XXXX` auto-fills the join form
- Branded HTML email invites via SMTP (Nodemailer)
- **Passphrase re-login**: each member receives a `word-word-word-1234` passphrase on join (stored as scrypt hash, shown once only). Members can sign back in via `/join?action=login` with @usertag + passphrase

### Key Features (Task #2 — Real-time Chat)

- WebSocket server at `/api/ws` with full authentication, presence, and room management
- Three seeded rooms: `general`, `random`, `announcements` (owner-only post)
- Real-time messaging with reactions, replies, typing indicators, and sticker support
- Chat page at `/chat`: room sidebar + message feed + online member panel
- Sticker picker: Dollars-themed emoji sticker pack with 18 stickers
- Inline emoji picker (quick 20 emoji grid) for message composition
- Reaction bar (hover any message): 6 quick reactions + toggle support
- Reply threads: click CornerUpLeft icon on hover to quote-reply
- Message types: `text`, `sticker`, `system`
- Connection status indicator (CONNECTED/OFFLINE with Wifi icon)
- Unread counts per room (badge on room in sidebar)
- Notification sound on new messages from others
- Global Chat Protocol module in lobby shows "ONLINE" (green) and links to /chat

### Security

- Helmet (15+ secure HTTP headers), HSTS, CSP, anti-clickjacking
- Per-endpoint rate limiting (join: 10/15min, tag-check: 30/min, email: 10/hr)
- Timing-safe admin secret comparison (`crypto.timingSafeEqual`)
- Vague error messages (no enumeration attacks)
- Input size limit (100KB body), CORS lockdown via `ALLOWED_ORIGIN` env var
- No IPs, no emails, no PII stored

### Environment Variables

- `DATABASE_URL` — PostgreSQL connection (auto-provided)
- `ADMIN_SECRET` — Admin panel passphrase
- `ALLOWED_ORIGIN` — CORS origin lock (set on production VPS)
- `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `SMTP_FROM` — Email config
- `PORT` — Server port (auto-assigned)

### Pending Tasks

- Task #3: Media messaging (photos, videos, voice notes)
- Task #4: Engagement features (polls, voice rooms, stories, disappearing messages, mission board)

## Stack

- **Monorepo tool**: pnpm workspaces
- **Node.js version**: 24
- **Package manager**: pnpm
- **TypeScript version**: 5.9
- **API framework**: Express 5
- **Database**: PostgreSQL + Drizzle ORM
- **Validation**: Zod (`zod/v4`), `drizzle-zod`
- **API codegen**: Orval (from OpenAPI spec)
- **Build**: esbuild (CJS bundle)
- **Frontend**: React 19 + Vite, wouter (routing), framer-motion, tanstack-query, shadcn/ui, lucide-react

## Structure

```text
artifacts-monorepo/
├── artifacts/              # Deployable applications
│   ├── api-server/         # Express API server (port 8080)
│   └── dollars/            # React + Vite frontend (root path /)
├── lib/                    # Shared libraries
│   ├── api-spec/           # OpenAPI spec + Orval codegen config
│   ├── api-client-react/   # Generated React Query hooks
│   ├── api-zod/            # Generated Zod schemas from OpenAPI
│   └── db/                 # Drizzle ORM schema + DB connection
├── scripts/                # Utility scripts
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── tsconfig.json
└── package.json
```

## TypeScript & Composite Projects

Every package extends `tsconfig.base.json` which sets `composite: true`. The root `tsconfig.json` lists all packages as project references. This means:

- **Always typecheck from the root** — run `pnpm run typecheck` (which runs `tsc --build --emitDeclarationOnly`). This builds the full dependency graph so that cross-package imports resolve correctly. Running `tsc` inside a single package will fail if its dependencies haven't been built yet.
- **`emitDeclarationOnly`** — we only emit `.d.ts` files during typecheck; actual JS bundling is handled by esbuild/tsx/vite...etc, not `tsc`.
- **Project references** — when package A depends on package B, A's `tsconfig.json` must list B in its `references` array. `tsc --build` uses this to determine build order and skip up-to-date packages.

## Root Scripts

- `pnpm run build` — runs `typecheck` first, then recursively runs `build` in all packages that define it
- `pnpm run typecheck` — runs `tsc --build --emitDeclarationOnly` using project references

## Packages

### `artifacts/api-server` (`@workspace/api-server`)

Express 5 API server with security hardening. Routes live in `src/routes/` and use `@workspace/api-zod` for request and response validation and `@workspace/db` for persistence.

- Entry: `src/index.ts` — reads `PORT`, starts Express
- App setup: `src/app.ts` — mounts helmet, CORS, rate limiting, JSON parsing, routes at `/api`
- Routes: `src/routes/auth.ts` (join, check-tag, me, logout), `src/routes/admin.ts` (invite codes, members, email), `src/routes/health.ts`, `src/routes/members.ts`
- Middlewares: `src/middlewares/auth.ts` (requireAuth, requireAdmin)
- Lib: `src/lib/security.ts` (token gen, tag validation), `src/lib/email.ts` (SMTP invites)
- Depends on: `@workspace/db`, `@workspace/api-zod`

### `artifacts/dollars` (`@workspace/dollars`)

React + Vite frontend. Dark atmospheric theme with glitch effects.

- Pages: `src/pages/landing.tsx`, `src/pages/join.tsx`, `src/pages/lobby.tsx`, `src/pages/chat.tsx`, `src/pages/admin.tsx`, `src/pages/not-found.tsx`
- Hooks: `src/hooks/use-chat-ws.ts` (WebSocket connection with presence/typing/reactions)
- Lib: `src/lib/stickers.ts` (Dollars sticker pack), `src/lib/notification-sound.ts`
- Context: `src/contexts/auth-context.tsx` (token + admin secret management)
- Uses `@workspace/api-client-react` for type-safe API calls

### `lib/db` (`@workspace/db`)

Database layer using Drizzle ORM with PostgreSQL.

- Schema: `src/schema/inviteCodes.ts`, `src/schema/members.ts`, `src/schema/sessions.ts`
- Push: `pnpm --filter @workspace/db run push`

### `lib/api-spec` (`@workspace/api-spec`)

OpenAPI 3.1 spec and Orval config. Codegen: `pnpm --filter @workspace/api-spec run codegen`

### `lib/api-zod` (`@workspace/api-zod`)

Generated Zod schemas from the OpenAPI spec.

### `lib/api-client-react` (`@workspace/api-client-react`)

Generated React Query hooks and fetch client from the OpenAPI spec.

### `scripts` (`@workspace/scripts`)

Utility scripts. Run via `pnpm --filter @workspace/scripts run <script>`.
