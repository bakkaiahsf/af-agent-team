# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Nexus Console** is a Software Agentic Development Platform (SADP) — a Next.js 14 web application that orchestrates AI agents to analyze, design, develop, test, and review software work items. Humans are reviewers and approvers; agents are the primary implementers.

The platform is model-agnostic and tool-agnostic. It connects to Jira, GitHub, and AI providers (Anthropic Claude) through configurable integrations managed in the Admin Console.

## Development Commands

```bash
# Install dependencies (run from nexus-console/)
npm install

# Start dev server (port 3001)
npm run dev

# Production build
npm run build

# Type-check (no emit)
npm run typecheck

# Lint
npm run lint
```

All commands run from `nexus-console/`. There is no root-level workspace to manage.

## Repository Structure

```
nexus-console/
├── src/
│   ├── app/                    # Next.js App Router pages + API routes
│   │   ├── onboarding/         # Project Onboarding Wizard + Admin Console
│   │   │   ├── page.tsx        # Server component wrapper
│   │   │   └── OnboardingWizard.tsx  # Full wizard + admin UI (~2300 lines)
│   │   ├── board-health/       # Board Health multi-tab page
│   │   ├── dashboard/          # Executive dashboard
│   │   ├── analytics/          # Analytics page
│   │   ├── projects/           # Projects hub
│   │   └── api/                # API routes (see below)
│   ├── components/
│   │   ├── Sidebar.tsx         # Left nav (GitLab-style collapsible)
│   │   ├── PipelineBoard.tsx   # Kanban-style work item board
│   │   ├── AgentDrawer.tsx     # SSE agent output panel
│   │   ├── AgentActionMenu.tsx # Action trigger dropdown
│   │   ├── ModelSelector.tsx   # LLM model picker
│   │   └── ...
│   ├── context/
│   │   └── RoleContext.tsx     # Global role state (developer/scrum-master/program-manager/leadership)
│   └── lib/
│       ├── agentRunner.ts      # Core SSE orchestration runtime
│       ├── jira.ts             # Jira REST API client
│       └── activityFeed.ts     # Activity feed helpers
├── data/
│   ├── prompts/                # Runtime agent system prompts (read by agentRunner.ts)
│   ├── projects.json           # Project registry (Jira key, GitHub repo, board ID)
│   ├── registry.json           # Agent/skill-pack registry
│   ├── skill-packs.json        # Skill pack definitions
│   └── enterprise-standards.json  # Board health thresholds (served by /api/standards)
├── .env.local                  # Local secrets (gitignored — never commit)
├── next.config.mjs
├── tailwind.config.ts
└── tsconfig.json
```

## Architecture

### Agent Orchestration Runtime (`src/lib/agentRunner.ts`)

Core SSE streaming engine. Called by `GET /api/run-agent`.

**Flow:**
1. Loads agent system prompt from `data/prompts/` at runtime
2. Prepends role-specific preamble from `ROLE_PROMPTS` map
3. Calls Anthropic SDK with streaming + a hardcoded `TOOLS` list
4. Runs agentic loop (max 20 turns) until `stop_reason !== "tool_use"`
5. Emits SSE events: `stage_start`, `tool_call`, `tool_result`, `text`, `stage_end`, `done`, `error`

**Valid `action` values:** `analyze`, `design`, `develop`, `test`, `pr`, `deploy`, `board-health-scan`, `chat`

### Onboarding Wizard + Admin Console (`src/app/onboarding/OnboardingWizard.tsx`)

Single large `"use client"` component (~2300 lines) with two top-level views:

- **WizardView** — 8-step project provisioning flow (Project Basics → Review & Launch)
- **AdminView** — 8-section tenant configuration console:
  - Policies, Rules, Hooks, Templates, Onboarding Flow, Integrations, Approvals, Audit

The `AdminIntegrations` section renders brand-colored cards for 10 connected tools: GitLab, Jira, Confluence, Teams, Slack, CI/CD, SonarQube, Snyk, Vault, Datadog.

### Role Context (`src/context/RoleContext.tsx`)

Persists active role in localStorage. Passed to `agentRunner.ts` which prepends a role-specific instruction block so the same agent responds differently for a developer vs a senior leader.

### Board Health (`/board-health`)

Multi-tab page: Board Owner, Jira Hygiene, Scrum of Scrums, Leadership, Standards, Scan Log. Scanning triggers `board-health-scan` action in `agentRunner.ts`. Agent JSON output (`BoardHealthReport`) is parsed from SSE events and stored via `POST /api/board-health/report`.

### API Routes

| Route | Status |
|-------|--------|
| `GET /api/run-agent` | Implemented (SSE, all 8 actions) |
| `GET /api/board-health/boards` | Implemented |
| `GET/POST /api/board-health/report` | Implemented (in-memory) |
| `GET /api/dashboard` | Implemented |
| `GET /api/analytics` | Implemented |
| `GET /api/agent-activity` | Implemented |
| `GET /api/standards[/section]` | Implemented |
| `GET /api/projects` | Implemented |
| `POST /api/jira-forge` | Implemented |
| `POST /api/demo/seed` | Implemented |
| `GET /api/pipeline`, `GET /api/repos`, `GET /api/environments`, `POST /api/deploy` | Stubs |

## Environment Variables (`.env.local`)

```
ANTHROPIC_API_KEY=          # Claude API key
JIRA_BASE_URL=              # https://yourorg.atlassian.net
JIRA_EMAIL=                 # Atlassian account email
JIRA_API_TOKEN=             # Atlassian API token
JIRA_PROJECT_KEY=           # Default project key (e.g. SCRUM)
CONFLUENCE_SPACE_KEY=       # Confluence space key
GITHUB_TOKEN=               # Fine-grained PAT (repo + pull_requests + contents)
GITHUB_ORG=                 # GitHub org name
GITHUB_REPO=                # Target repository name
NEXT_PUBLIC_JIRA_URL=       # Same as JIRA_BASE_URL (client-side)
```

## Technology Stack

- **Next.js 14** (App Router) + **React 18** + **TypeScript 5**
- **Tailwind CSS v3** — dark theme with custom color tokens
- **`@anthropic-ai/sdk`** — streaming agent calls
- **Vercel** — auto-deploy on push to `main` (connected to `https://github.com/bakkaiahsf/nexus-console.git`)

## Deployment

Push to `main` → Vercel auto-deploys. No manual deploy steps needed.
Live at: `https://nexus-console-pi.vercel.app`

Secrets are managed in Vercel project environment variables (not committed to git).
