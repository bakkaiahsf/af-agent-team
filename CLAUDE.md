# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Nexus Console** is an AI-powered SDLC orchestration platform. It integrates with Jira, GitLab/GitHub, Confluence, Salesforce, and other enterprise tools to let agents autonomously execute software delivery tasks (BA analysis, architecture design, code implementation, testing, PR creation, deployment) on behalf of developers and engineering leaders.

## Development Commands

All commands run from `nexus-console/`:

```bash
npm install          # also runs prisma generate via postinstall
npm run dev          # dev server on port 3001
npm run build        # prisma generate + next build
npm run typecheck    # tsc --noEmit
npm run lint         # next lint
npm test             # vitest run (single pass)
npm run test:watch   # vitest in watch mode
npm run test:coverage # vitest with coverage
```

Run a single test file:
```bash
npx vitest run src/lib/capability.test.ts
```

Tests use Vitest with the `node` environment. Path alias `@` resolves to `src/`. Currently the only test file is `src/lib/capability.test.ts`.

### Database

Prisma uses the Neon serverless adapter. There are no migration files — the schema is applied directly.

```bash
# Apply schema to database (from nexus-console/)
npx prisma db push

# Seed demo data
npx ts-node prisma/seed-demo.ts
```

`DATABASE_URL` must be set in `.env.local` before any DB commands work.

## Architecture

### Agent Orchestration (`src/lib/agentRunner.ts`)

The core engine. `runAgent()` is an async generator that streams `AgentEvent` objects. Called by `GET /api/run-agent` which wraps the generator in a `ReadableStream` and emits Server-Sent Events.

**Flow:**
1. Loads the agent's system prompt from `data/prompts/<agent>.md` at runtime
2. Prepends a role-specific preamble from `ROLE_PROMPTS` (developer / scrum-master / program-manager / leadership)
3. Calls Anthropic SDK with streaming + the `TOOLS` list (Jira, GitHub, Salesforce, Confluence, board-health tools)
4. Runs an agentic loop (max 20 turns) executing tool calls synchronously until `stop_reason !== "tool_use"`
5. Emits typed SSE events: `stage_start`, `tool_call`, `tool_result`, `text`, `stage_end`, `done`, `error`
6. Persists each `AgentRun` and `AgentEvent` to Postgres via Prisma

**Valid `action` values** map to named agents in `ACTION_CONFIGS`:

| action | agent | purpose |
|--------|-------|---------|
| `analyze` | ba-agent | BA spec extraction |
| `design` | architect-agent | Architecture doc |
| `develop` | developer-agent | Metadata + branch + deploy-validate |
| `test` | qa-agent | Apex/Jest tests + Code Analyzer |
| `pr` | developer-agent | Open GitHub PR |
| `deploy` | orchestrator | Quick deploy + Confluence page + PR comment |
| `board-health-scan` | board-health-agent | Health score report |
| `chat` | orchestrator | Conversational Q&A |

### Database (`src/lib/db.ts` + `prisma/schema.prisma`)

Postgres via Neon serverless. Prisma client is generated to `src/generated/prisma/`. The singleton pattern in `db.ts` prevents connection storms in dev.

**Key models:** `Project`, `AgentRun`, `AgentEvent`, `BoardHealthReport`, `JiraHygieneReport`, `ApprovalRequest`, `ApprovalWorkflow`, `CompliancePolicy`, `Risk`, `Decision`, `MeetingTranscript`, `AuditLog`.

Design invariant: **Jira/GitLab/GitHub objects are never cached** — only Nexus-owned computed data (health reports, risks, decisions, audit logs) is persisted. External system keys are stored as reference pointers only.

### Role Context (`src/context/RoleContext.tsx`)

Global React context that persists the active role in localStorage. The selected role is passed as a query param to `/api/run-agent`, where `agentRunner.ts` prepends the matching `ROLE_PROMPTS` block so the same agent responds differently for a developer vs. a senior leader.

### Onboarding Wizard + Admin Console (`src/app/onboarding/OnboardingWizard.tsx`)

A large `"use client"` component (~2300 lines) with two top-level views controlled by `activeView`:

- **WizardView** — 8-step provisioning flow that calls `POST /api/projects/provision`, which runs `src/lib/provision.ts` to create a real Git repo, Jira project, Confluence space, CI/CD pipeline file, and `.claude/` config files
- **AdminView** — 8-section tenant config console (Policies, Rules, Hooks, Templates, Onboarding Flow, Integrations, Approvals, Audit)

Types and constants are split into `types.ts` and `constants.ts` in the same directory.

### Board Health (`/board-health`)

Triggers the `board-health-scan` action. The agent returns a `BoardHealthReport` JSON blob inside SSE `text` events. The client parses it, stores it via `POST /api/board-health/report`, and renders it across 6 tabs (Board Owner, Jira Hygiene, Scrum of Scrums, Leadership, Standards, Scan Log). Thresholds come from `data/enterprise-standards.json` via `GET /api/standards`.

### Capability Tier System (`src/lib/capability.ts`)

Agent actions are gated by a **tier** computed at read time from a project's connected integrations — never stored in the DB. Tiers are cumulative:

| Tier | Requirement | Unlocked actions |
|------|-------------|-----------------|
| 0 | none | `chat` |
| 1 | GitLab or GitHub | + `analyze`, `design` |
| 2 | + Jira | + `develop`, `pr`, `test` |
| 3 | + CI/CD | + `board-health-scan` |
| 4 | + Vault + Snyk/SonarQube | + `deploy` |

`computeTier()`, `allowedActions()`, and `missingForNextTier()` are the public API. This logic has unit tests in `src/lib/capability.test.ts`.

### Skills System (`src/lib/skill-merger.ts`, `prisma/schema.prisma`)

Admin-governed catalog (`Skill` model) of agent skill packs. Project owners assign skills; `mergeSkillsIntoIntelligence()` merges them into an `IntelligenceConfig` before each agent run. Merge rules: `toolAccess: false` always wins; `agentOverrides.systemPromptAppend` strings are concatenated; project-level config wins on `model` and `maxTurns`. Built-in skills are seeded via `POST /api/admin/skills/seed`.

`IntelligenceConfig` (`src/lib/intelligence-types.ts`) carries: `toolAccess` (per-tool enable/disable), `agentOverrides` (per-agent model/prompt overrides), `groundingContext` (text prepended to the agent's grounding), and `resources`.

### Integration Clients (`src/lib/`)

| file | purpose |
|------|---------|
| `jira.ts` | Jira REST API (issues, comments, transitions, boards, sprints) |
| `jira-hygiene.ts` | Jira hygiene score computation |
| `github.ts` | GitHub REST API (repos, PRs, branches, commits) |
| `gitlab.ts` | GitLab API (projects, MRs, pipelines) |
| `confluence.ts` | Confluence REST API (pages, spaces) |
| `salesforce.ts` | Salesforce metadata + deploy-validate via SFDX |
| `provision.ts` | Onboarding provisioning engine (streaming ProvisionEvent) |
| `activityFeed.ts` | Recent agent-run activity for the dashboard feed |

### Agent Prompts (`data/prompts/`)

Runtime Markdown files loaded by `agentRunner.ts` via `readFileSync`. Each file maps to one agent: `ba-agent.md`, `architect-agent.md`, `developer-agent.md`, `qa-agent.md`, `security-agent.md`, `board-health-agent.md`, `orchestrator.md`. Editing these files changes agent behavior without a code redeploy on Vercel (they are read at request time).

## Environment Variables (`.env.local`)

```
DATABASE_URL=               # Neon Postgres connection string
ANTHROPIC_API_KEY=          # Claude API key
JIRA_BASE_URL=              # https://yourorg.atlassian.net
JIRA_EMAIL=
JIRA_API_TOKEN=
JIRA_PROJECT_KEY=
CONFLUENCE_SPACE_KEY=
GITHUB_TOKEN=               # Fine-grained PAT (repo, pull_requests, contents)
GITHUB_ORG=
GITHUB_REPO=
NEXT_PUBLIC_JIRA_URL=       # Same as JIRA_BASE_URL (client-side)
SF_ORG_ALIAS=               # Salesforce org alias for sfdx commands
```

### API Routes (`src/app/api/`)

Notable surface area beyond the basic CRUD:
- `GET /api/run-agent` — SSE stream, query params: `action`, `jiraKey`, `projectId`, `role`, `model`
- `POST /api/projects/provision` — streams `ProvisionEvent` objects (SSE) while creating Git repo + Jira + Confluence
- `POST /api/board-health/report` — persists `BoardHealthReport` from agent output
- `GET /api/standards` / `GET /api/standards/[section]` — reads `data/enterprise-standards.json`
- `POST /api/admin/skills/seed` — seeds built-in skills from `data/builtin-skills.json`
- `POST /api/projects/[id]/detect-skills` — auto-detects applicable skills from repo contents
- `POST /api/demo/seed` — seeds demo projects + data (uses `data/projects.json`)
- `GET /api/cron/board-hygiene` — scheduled hygiene scan (called by Vercel Cron)

## Technology Stack

- **Next.js 14** (App Router) + **React 18** + **TypeScript 5**
- **Tailwind CSS v3** — dark theme, custom tokens
- **`@anthropic-ai/sdk`** — streaming agent calls with tool use
- **Prisma 7** + **`@prisma/adapter-neon`** — Postgres via Neon serverless
- **Vitest** — unit testing (`vitest.config.ts` at `nexus-console/`)
- **Vercel** — auto-deploy on push to `main`; live at `https://nexus-console-pi.vercel.app`

Prisma client is generated to `src/generated/prisma/` (not the default location). Import from `@/generated/prisma/client`, not `@prisma/client`.

## Data files (`nexus-console/data/`)

| file | purpose |
|------|---------|
| `prompts/<agent>.md` | Agent system prompts, loaded at request time via `readFileSync` |
| `enterprise-standards.json` | Board health thresholds served by `/api/standards` |
| `builtin-skills.json` | Seed data for built-in skill catalog |
| `projects.json` | Demo project seed data |
| `registry.json`, `skill-packs.json` | Additional skill/capability registry data |
