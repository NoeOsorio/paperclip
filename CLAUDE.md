# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Paperclip is a Node.js + React full-stack monorepo that orchestrates teams of AI agents to run a business. It provides a control plane with an org chart, budgets, goal alignment, and agent coordination. Users interact with a web dashboard to manage agent teams, track tasks, and monitor costs.

**Key Principles:**
- Bring your own agents (any runtime, any provider)
- Goal-driven task management with org charts
- Built-in governance, budgets, and cost tracking
- Agents wake on heartbeat schedules for work delegation

## Technology Stack

### Core
- **Runtime:** Node.js (>=20), pnpm 9.15.4
- **Language:** TypeScript 5.7.3 with strict mode
- **Module System:** ESM (ES2023 target, NodeNext resolution)

### Server
- **Framework:** Express 5.1.0
- **Database:** PostgreSQL (via embedded-postgres or external) + Drizzle ORM 0.38.4
- **Auth:** better-auth 1.4.18
- **Validation:** Zod + AJV with JSON Schema
- **Real-time:** WebSockets
- **Logging:** Pino + Pino-HTTP

### UI
- **Framework:** React 19.0.0 + Vite 6.1.0
- **UI Kit:** Radix UI + Tailwind CSS 4.0.7
- **Rich Text:** Lexical editor + MDX editor
- **State:** TanStack React Query 5.90.21
- **Routing:** React Router 7.1.5
- **Rendering:** Markdown + Mermaid diagrams

### Adapters (LLM Integration Layer)
Multiple adapters support different AI providers:
- `@paperclipai/adapter-claude-local` (Claude, via local HTTP)
- `@paperclipai/adapter-codex-local` (Codex)
- `@paperclipai/adapter-cursor-local` (Cursor)
- `@paperclipai/adapter-gemini-local` (Gemini)
- `@paperclipai/adapter-opencode-local` (OpenCode)
- `@paperclipai/adapter-pi-local` (Pi)
- `@paperclipai/adapter-openclaw-gateway` (OpenClaw via gateway)

All adapters share common utils via `@paperclipai/adapter-utils`.

### Testing & Build
- **Testing:** Vitest 3.0.5
- **E2E Tests:** Playwright
- **Build:** TypeScript + esbuild (CLI), Vite (UI)
- **Linting:** Check for forbidden tokens via custom script

## Project Structure

```
paperclip/
├── server/              # Express app: API, WebSocket, auth, database
├── ui/                  # React SPA: Dashboard, org charts, task views
├── cli/                 # CLI tool for orchestration (esbuild bundle)
├── packages/
│   ├── db/             # Drizzle ORM schema, migrations, seed
│   ├── shared/         # Type definitions, telemetry helpers, Zod schemas
│   ├── adapter-utils/  # Common adapter interface + utilities
│   ├── adapters/       # 7 LLM adapter implementations
│   ├── mcp-server/     # MCP (Model Context Protocol) server
│   └── plugins/        # Plugin SDK for extensibility
├── scripts/            # 39 automation scripts (builds, releases, DB ops)
├── tests/              # E2E and smoke tests
├── docs/               # User documentation
├── docker/             # Docker compose files for local dev
├── .github/workflows/  # CI/CD: pr.yml, release.yml, e2e.yml, etc.
└── .claude/            # Claude Code settings
```

## Key Commands

### Development
```bash
pnpm dev              # Server in watch mode (alias: dev:watch)
pnpm dev:once         # Server single run (no watch)
pnpm dev:server       # Server only (port 3100)
pnpm dev:ui           # UI dev server (port 5173, proxies API to 3100)
```
To develop full-stack, run `pnpm dev` and `pnpm dev:ui` in separate terminals.

### Building
```bash
pnpm build           # Build all packages + UI (requires workspace links)
pnpm build:npm       # Create npm release bundle
```

### Testing
```bash
pnpm test            # Run all tests (alias: test:run)
pnpm test:watch      # Watch mode for tests
pnpm test:e2e        # Playwright E2E tests
pnpm test:e2e:headed # E2E tests in headed browser
```

### Database
```bash
pnpm db:generate     # Generate Drizzle schema from src/schema.ts
pnpm db:migrate      # Apply pending migrations
pnpm db:backup       # Backup database
```

### Type Checking & Linting
```bash
pnpm typecheck       # TypeScript type checking (all packages)
pnpm check:tokens    # Scan for forbidden tokens (secrets, etc.)
```

### Monorepo Management
```bash
pnpm preflight:workspace-links  # Ensure workspace package links are correct
```

## Architecture Patterns

### Adapter Pattern
All LLM integrations follow a common interface defined in `@paperclipai/shared` and implemented in `packages/adapters/*`. This allows swapping providers without changing core logic. Adapters handle:
- Model-specific API calls
- Context management (with provider-specific compaction strategies)
- Cost tracking
- Error handling and retries

### Database Layer
Drizzle ORM with managed PostgreSQL (embedded-postgres in dev, external in prod). Migrations are numbered SQL files in `packages/db/src/migrations/` and applied sequentially. Schema is defined in TypeScript for type safety.

### Real-time Communication
Express + WebSocket for:
- Agent heartbeat subscriptions
- Task updates (create, assign, complete)
- Cost tracking streams
- Org chart changes

### UI State Management
TanStack React Query for server state, local React state for UI-only state. Proxy config in vite.config.ts routes `/api/*` to server (port 3100).

## Working with the Codebase

### Adding a New Feature
1. **Small change:** Push directly, write good commit messages, ensure tests pass
2. **Bigger change:** Discuss in Discord #dev, get rough agreement, then build with before/after proof

### Database Changes
1. Create migration in `packages/db/src/migrations/`
2. Update schema in `packages/db/src/schema.ts`
3. Run `pnpm db:generate` to sync Drizzle types
4. Run `pnpm db:migrate` to apply
5. Test with `pnpm test`

### Adding a New Adapter
1. Create `packages/adapters/{provider-local}/`
2. Implement interface from `@paperclipai/adapter-utils`
3. Export from `server/src/index.ts` and `cli/src/index.ts`
4. Add to tsconfig.json references
5. Add package.json to Dockerfile for builds

### TypeScript References
The root `tsconfig.json` uses project references for fast incremental builds. Always run `pnpm typecheck` before committing to ensure no regressions across the monorepo.

## Common Issues & Debugging

### Database Lock Issues
If embedded-postgres gets stuck: `pkill -f embedded-postgres` or restart dev server.

### Workspace Link Errors
Run `pnpm preflight:workspace-links` before building. This ensures all workspace:* dependencies resolve correctly.

### Hot Reload Not Working
- Server: Edit files in `server/src/` and check terminal for watch errors
- UI: Check Vite hot reload status in browser console; proxy settings in vite.config.ts

### Port Conflicts
- Server: 3100 (change via PAPERCLIP_PORT env var if needed)
- UI: 5173 (change via Vite config)

## PR Requirements

**All PRs must include:**
1. **Model Used:** Specify the AI model and version used (or "None — human-authored")
2. **Tests passing:** `pnpm test:run` and `pnpm typecheck`
3. **PR template:** Use `.github/PULL_REQUEST_TEMPLATE.md`
4. **Code review:** Greptile review must pass
5. **Commit messages:** Clear, explain the "why" not just the "what"

**For bigger/impactful changes:**
- Discuss in Discord #dev first
- Include before/after screenshots
- Provide manual testing notes
- Ensure all CI workflows pass

## Environment Variables

Key variables for local development (see `.env.example`):
- `PAPERCLIP_PORT` - Server port (default: 3100)
- `PAPERCLIP_MIGRATION_PROMPT` - Set to `never` to skip migration prompts in dev
- `PAPERCLIPAI_LOCAL_DB` - Path to local DB (embedded-postgres in dev)

## Useful Documentation

- README.md - Full project overview and feature table
- CONTRIBUTING.md - Detailed contribution guidelines
- AGENTS.md - Agent system concepts and configuration
- adapter-plugin.md - Guide for building adapter plugins
- docs/ - User-facing documentation

## Important Notes

- Run `pnpm check:tokens` before committing to catch accidentally committed secrets
- Run `pnpm preflight:workspace-links` if you get unresolved workspace dependency errors
- The root `tsconfig.json` uses project references — add new packages there for incremental builds
