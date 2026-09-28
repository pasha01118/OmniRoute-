# OmniRoute Dashboard Redesign - Architecture Blueprint

## 1. Framework Digital Mapping & Diagram

### 1.1 Technology Stack Map

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        OMNIROUTE TECHNOLOGY STACK                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────┐  │
│  │  FRONTEND    │    │  BUILD/DEV   │    │  RUNTIME     │    │  QUALITY │  │
│  ├──────────────┤    ├──────────────┤    ├──────────────┤    ├──────────┤  │
│  │ Next.js 16   │    │ Turbopack    │    │ Node.js ≥22  │    │ ESLint 9 │  │
│  │ React 19.2   │    │ TypeScript 6 │    │ ES Modules   │    │ Prettier │  │
│  │ Tailwind 4   │    │ TypeScript 6 │    │ Pino Logger  │    │ Vitest   │  │
│  │ Fumadocs UI  │    │ Bun 1.3.14   │    │ Sharp        │    │ Playwright│ │
│  │ Recharts     │    │ SWC          │    │ Undici       │    │ Striker  │  │
│  │ Zustand      │    │ @tailwindcss │    │ WS           │    │ c8       │  │
│  │ next-intl    │    │ postcss      │    │ better-sqlite3│   │ Knip     │  │
│  │ Framer Motion│    │              │    │ @atjsh/llm   │    │ TypeCov  │  │
│  └──────────────┘    └──────────────┘    └──────────────┘    └──────────┘  │
│                                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────┐  │
│  │  BACKEND     │    │  PROXY/CORE  │    │  DATABASE    │    │  INFRA   │  │
│  ├──────────────┤    ├──────────────┤    ├──────────────┤    ├──────────┤  │
│  │ Express 5    │    │ open-sse     │    │ SQLite (WAL) │    │ Docker   │  │
│  │ HTTP Proxy   │    │ Executors    │    │ 130 Migrations│   │ Electron │  │
│  │ CORS Middle  │    │ Translators  │    │ Domain Mods  │    │ ngrok    │  │
│  │ AuthZ Guard  │    │ Transformers │    │ FTS5 Search  │    │ Selfsigned│ │
│  │ Rate Limit   │    │ Combo Router │    │ Qdrant (opt) │    │ MITM/TLS │  │
│  │ OpenAPI Spec │    │ Resilience   │    │ LRU Cache    │    │ systemd  │  │
│  └──────────────┘    └──────────────┘    └──────────────┘    └──────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Architecture Layers (Clean Architecture)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ARCHITECTURE LAYERS                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ PRESENTATION LAYER (Next.js App Router)                            │   │
│  │ src/app/(dashboard)/dashboard/*  ── 50+ Pages                      │   │
│  │ src/shared/components/*      ── 100+ Shared Components             │   │
│  │ src/shared/hooks/*           ── React Hooks                        │   │
│  │ src/store/*                  ── Zustand Stores                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ APPLICATION LAYER (API Routes & Services)                          │   │
│  │ src/app/api/v1/*           ── REST/SSE Endpoints                   │   │
│  │ src/sse/handlers/*         ── Chat/Embeddings/Streaming            │   │
│  │ src/sse/services/*         ── Combo, Cache, RateLimit, Auth        │   │
│  │ src/lib/*                  ── Domain Libraries (50+)               │   │
│  │ src/server/*               ── Middleware, AuthZ, CORS              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ DOMAIN LAYER (Business Logic)                                      │   │
│  │ src/domain/*               ── Policy Engine, Cost Rules, Fallback  │   │
│  │ src/shared/schemas/*       ── Zod Validation Schemas               │   │
│  │ src/shared/constants/*     ── Provider Configs, Capabilities       │   │
│  │ src/shared/types/*         ── TypeScript Definitions               │   │
│  │ src/models/*               ── Data Models                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ INFRASTRUCTURE LAYER                                               │   │
│  │ src/lib/db/*               ── SQLite Domain Modules (130 Migs)     │   │
│  │ src/lib/db/core.ts         ── WAL Connection Singleton             │   │
│  │ src/mitm/*                 ── TLS Interception, Cert Management    │   │
│  │ src/sse/executors/*        ── Provider HTTP Clients (291)          │   │
│  │ src/sse/translator/*       ── Format Conversion (OAI↔Claude↔Gemini)│   │
│  │ src/sse/transformer/*      ── Responses API ↔ Chat Completions     │   │
│  │ open-sse/mcp-server/*      ── 105 MCP Tools, 3 Transports          │   │
│  │ src/lib/a2a/*              ── A2A JSON-RPC Server                  │   │
│  │ src/lib/skills/*           ── Extensible Skill Framework           │   │
│  │ src/lib/memory/*           ── FTS5 + Qdrant Vector Memory          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Repository Diagram

### 2.1 Monorepo Structure

```
omniroute/
├── 📁 .github/                    # CI/CD, Copilot, PR templates
├── 📁 .claude/worktrees/          # Git worktrees for parallel dev
├── 📁 @omniroute/                 # Internal packages
│   ├── opencode-plugin/           # OpenCode integration
│   └── opencode-provider/         # OpenCode provider
├── 📁 bin/                        # CLI Entry Points
│   ├── cli/                       # Ink-based TUI
│   ├── tray/                      # System tray (Electron)
│   └── omniroute.mjs              # Main CLI binary
├── 📁 changelog.d/                # Automated changelog fragments
├── 📁 _references/                # Research docs (CLIs, etc.)
├── 📁 _tasks/                     # Planning artifacts (separate git repo)
├── 📁 docs/                       # Documentation (architecture, routing, security)
├── 📁 electron/                   # Desktop App
├── 📁 open-sse/                   # Streaming Engine Workspace
│   ├── handlers/                  # Request handlers
├── 📁 executors/                 # Provider HTTP clients
├── 📁 translator/                # Format translators
├── 📁 transformer/               # Responses API ↔ Chat
├── 📁 services/                  # Combo, Cache, RateLimit
├── 📁 mcp-server/                # MCP Server (105 tools)
├── 📁 config/                    # Provider registry, constants
├── 📁 packages/                   # pnpm workspaces
│   └── browser-pool/              # Playwright browser pool
├── 📁 scripts/                    # Build/Dev/Quality scripts (48+)
│   ├── build/, check/, dev/, docs/, i18n/, quality/, release/
├── 📁 src/                        # MAIN SOURCE (Next.js App)
│   ├── app/                       # Next.js App Router
│   │   ├── (dashboard)/           # Dashboard Route Group
│   │   │   └── dashboard/         # 50+ Dashboard Pages
│   │   ├── api/v1/                # REST API Endpoints
│   │   ├── auth/, callback/       # OAuth Flows
│   │   └── .well-known/           # A2A Agent Card
│   ├── domain/                    # Domain Logic (Policy, Costs)
├── 📁 lib/                       # Domain Libraries (50+)
│   ├── db/                    # SQLite Modules + Migrations
│   ├── a2a/, mcp/, skills/    # Protocol Implementations
│   ├── memory/                # FTS5 + Qdrant
│   ├── guardrails/            # PII, Injection, Vision
│   └── compliance/            # Audit, Webhooks
├── 📁 middleware/                # Next.js Middleware
├── 📁 mitm/                      # TLS Interception
├── 📁 models/                    # Data Models
├── 📁 sse/                       # Streaming Engine
├── 📁 server/                    # AuthZ, CORS, Origin
├── 📁 shared/                    # Shared Code
│   ├── components/            # 100+ React Components
│   ├── components/layouts/    # DashboardLayout, Sidebar, Header
│   ├── hooks/                 # Custom React Hooks
│   ├── constants/             # Providers, Sidebar, Pricing
│   ├── schemas/               # Zod Schemas
│   ├── types/                 # TypeScript Types
│   ├── utils/                 # Utility Functions
│   ├── validation/            # Validation Helpers
│   └── providers/             # React Context Providers
├── 📁 store/                     # Zustand Stores
│   └── types/                     # Global Types
├── 📁 tests/                      # Test Suite (Unit, Integration, E2E)
│   ├── unit/                      # 1000+ Unit Tests
│   ├── integration/               # Integration Tests
│   └── e2e/                       # Playwright E2E
└── 📁 _config files_              # package.json, tsconfig, eslint, etc.
```

### 2.2 Dashboard Page Map (50+ Pages)

```
src/app/(dashboard)/dashboard/
├── 🏠 HomePageClient.tsx                 # Main landing (metrics, topology, quick-start)
├── 📊 analytics/                         # 7 Analytics Sub-pages
│   ├── page.tsx                          # Overview
│   ├── combo-health/                     # Combo health analysis
│   ├── utilization/                      # Provider utilization
│   ├── compression/                      # Compression analytics
│   ├── search/                           # Search analytics
│   ├── evals/                            # Evals dashboard
│   └── components/                       # ProviderCharts, DiversityScoreCard
├── 🔧 api-manager/                       # API Key Management
├── 🤖 auto-combo/                        # Auto-Combo Configuration
├── 📦 batch/                             # Batch Processing (files, wizard)
├── 💾 cache/                             # Cache Dashboard (entries, trends, media)
├── 🔗 combos/                            # Combo Builder + Detail
├── 📈 costs/                             # Costs (budget, pricing, quota-share)
├── 🔍 discovery/                         # Provider Discovery
├── 🎯 endpoint/                          # Endpoint Management + Components
├── 📝 logs/                              # Logs (proxy, console, activity)
├── 🤖 mcp/                               # MCP Server Management
├── 🤖 a2a/                               # A2A Server Management
├── 🔐 providers/                         # Provider Connections (291)
├── ⚙️ settings/                          # 8 Settings Sub-pages
│   ├── general/, appearance/, ai/, security/
│   ├── routing/, resilience/, cache/, advanced/
├── 🛠️ tools/                             # Tool Management
├── 🔄 translator/                        # Translation Studio
├── 🎮 playground/                        # Model Playground
├── 🧠 memory/                            # Conversational Memory
├── 🧩 skills/                            # Agent Skills
├── 📋 audit/                             # MCP/A2A Audit
├── 🏥 health/                            # Health Monitoring
├── 🌪️ chaos/                             # Chaos Engineering
├── 🎭 compression/                       # Compression Studio (live, studio)
├── 📡 webhooks/                          # Webhook Management
├── 📋 changelog/                         # Changelog Viewer
├── 👤 profile/                           # User Profile
├── 🚀 onboarding/                        # Onboarding Flow
├── 📋 cli-agents/                        # CLI Agent Management
├── 💻 cli-code/                          # CLI Code Tools
☁️ cloud-agents/                       # Cloud Agents (Codex, Devin, Jules)
├── 🔌 api-endpoints/                     # API Endpoint Registry
├── 🆓 free-provider-rankings/            # Free Provider Leaderboard
├── 🆓 free-tiers/                        # Free Tier Management
├── 📏 limits/                            # Rate Limits
├── 📋 quota/                             # Quota Management
├── 🛡️ resilience/                        # Resilience Dashboard
├── ⚡ runtime/                            # Runtime Configuration
├── 🔍 search-tools/                      # Search Tools
├── 🤖 acp-agents/                        # ACP Agents
├── 🤖 agent-skills/                      # Agent Skills Marketplace
├── 🎮 gamification/                      # Gamification
├── 🎭 conductor/                         # Conductor (Voice/Chat)
└── 🔧 system/                            # System Admin
```

### 2.3 Component Dependency Graph

```
src/shared/components/
├── layouts/
│   ├── DashboardLayout.tsx    ◄── Root layout (Sidebar, Header, Main)
│   ├── Sidebar.tsx            ◄── Navigation (758 lines, complex state)
│   ├── Header.tsx             ◄── Top bar (theme, lang, notifications)
│   └── Breadcrumbs.tsx        ◄── Navigation breadcrumbs
├── 📋 Core UI (to be redesigned)
│   ├── Card.tsx               ◄── Base card container
│   ├── Button.tsx             ◄── Primary/Secondary/Ghost variants
│   ├── Input.tsx              ◄── Form inputs with icons
│   ├── Select.tsx             ◄── Dropdown select
│   ├── Table.tsx              ◄── Data tables (needs creation)
│   ├── Tabs.tsx               ◄── Tab navigation
│   ├── Modal.tsx              ◄── Dialog/Confirm modals
│   ├── Badge.tsx              ◄── Status badges
│   ├── Avatar.tsx             ◄── User/Provider avatars
│   ├── Tooltip.tsx            ◄── Hover tooltips
│   └── Toast.tsx              ◄── Notification system
├── 🎯 Specialized
│   ├── ProviderIcon.tsx       ◄── Provider brand icons
│   ├── TokenHealthBadge.tsx   ◄── API key health
│   ├── DegradationBadge.tsx   ◄── System degradation
│   ├── LanguageSelector.tsx   ◄── i18n selector
│   ├── ThemeToggle.tsx        ◄── Dark/Light toggle
│   ├── CommandPalette.tsx     ◄── ⌘K quick nav
│   ├── NavigationProgress.tsx ◄── Top progress bar
│   ├── CloudSyncStatus.tsx    ◄── Sync indicator
│   └── OmniRouteLogo.tsx      ◄── Brand logo
├── 📊 Data Viz
│   ├── analytics/ProviderCharts.tsx
│   ├── compression/CompressionChart.tsx
│   └── flow/                  # React Flow diagrams
├── 📝 Forms
│   ├── Modal.tsx              ◄── ConfirmModal, FormModal
│   ├── Input.tsx              ◄── With icon, validation
│   └── Select.tsx             ◄── Multi-select, async
└── 📄 Docs/Markdown
    ├── docs/MDXComponents.tsx
    └── docs/TOC.tsx
```

---

## 3. Complete Architecture Blueprint

### 3.1 Request Pipeline (Production Flow)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        REQUEST PIPELINE (SSE/CHAT)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CLIENT REQUEST                                                             │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ NEXT.JS ROUTE: src/app/api/v1/chat/completions/route.ts            │   │
│  │ • CORS Preflight                                                    │   │
│  │ • Zod Body Validation (ChatCompletionRequest)                      │   │
│  │ • Optional Auth (extractApiKey / isValidApiKey)                    │   │
│  │ • API Key Policy Enforcement                                       │   │
│  │ • Handler Delegation → open-sse/handlers/chatCore.ts               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ HANDLECHATCORE (open-sse/handlers/chatCore.ts)                     │   │
│  │ • Cache Check (semantic cache)                                      │   │
│  │ • Rate Limit (per key/account)                                      │   │
│  │ • Combo Routing?                                                    │   │
│  │   ├─ YES: resolveComboTargets() → handleSingleModel() per target   │   │
│  │   └─ NO:  Direct to single model handling                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ TRANSLATION LAYER (open-sse/translator/)                           │   │
│  │ • translateRequest() → OpenAI↔Claude↔Gemini↔Custom Formats         │   │
│  │ • Model-specific parameter mapping                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ EXECUTION LAYER (open-sse/executors/)                              │   │
│  │ • getExecutor(provider) → BaseExecutor subclass                    │   │
│  │ • executor.execute() → fetch() upstream with retry/backoff         │   │
│  │ • Streaming (SSE) or JSON response                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ RESPONSE TRANSFORMATION                                            │   │
│  │ • translateResponse() → Normalize to OpenAI format                 │   │
│  │ • Responses API → Chat Completions (transformer/)                  │   │
│  │ • SSE Stream or JSON output                                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Resilience Architecture (3-Layer)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      THREE-LAYER RESILIENCE MODEL                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ LAYER 1: PROVIDER CIRCUIT BREAKER                                  │   │
│  │ Scope: Entire Provider (e.g., openai, anthropic, glm)              │   │
│  │ States: CLOSED → OPEN → HALF_OPEN                                  │   │
│  │ Triggers: 500/502/503/504/408 (threshold: 3-5 failures)           │   │
│  │ Reset: 15-60s timeout (lazy recovery via getStatus())             │   │
│  │ Storage: domain_circuit_breakers table                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ LAYER 2: CONNECTION COOLDOWN                                       │   │
│  │ Scope: Single Provider Connection/Account/Key                      │   │
│  │ Purpose: Skip bad key while other keys for same provider work      │   │
│  │ Fields: rateLimitedUntil, testStatus, lastError, backoffLevel      │   │
│  │ Cooldown: 3-5s base × 2^failureIndex (exponential backoff)         │   │
│  │ Anti-thundering-herd: Single cooldown extension per failure burst  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ LAYER 3: MODEL LOCKOUT                                             │   │
│  │ Scope: Provider + Connection + Specific Model                      │   │
│  │ Purpose: One model quota-exhausted ≠ whole connection down         │   │
│  │ Examples: Per-model 429, Local 404 missing model, Grok mode perms  │   │
│  │ Storage: In-memory per connection (accountFallback.ts)             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DEBUGGING GUIDANCE:                                                        │
│  • All keys skipped? → Check BOTH provider breaker + connection cooldown   │
│  • Provider excluded post-reset? → Use getStatus()/canExecute() not raw    │
│  • One key fails, others work? → Prefer connection cooldown               │
│  • One model fails? → Prefer model lockout                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Data Flow: Database Layer

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DATABASE ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ src/lib/db/core.ts  ── getDbInstance() (Singleton, WAL Journaling) │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│         ┌──────────────────────────┼──────────────────────────┐           │
│         ▼                          ▼                          ▼           │
│  ┌─────────────┐           ┌─────────────┐           ┌─────────────┐    │
│  │ Domain Mods │           │ Domain Mods │           │ Domain Mods │    │
│  │ (130 Migs)  │           │ (130 Migs)  │           │ (130 Migs)  │    │
│  ├─────────────┤           ├─────────────┤           ├─────────────┤    │
│  │ providers.ts│           │ settings.ts │           │ metrics.ts  │    │
│  │ models.ts   │           │ combos.ts   │           │ usage.ts    │    │
│  │ combos.ts   │           │ keys.ts     │           │ cache.ts    │    │
│  │ keys.ts     │           │ logs.ts     │           │ memory.ts   │    │
│  │ ...         │           │ ...         │           │ ...         │    │
│  └─────────────┘           └─────────────┘           └─────────────┘    │
│         │                          │                          │           │
│         └──────────────────────────┼──────────────────────────┘           │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ src/lib/localDb.ts  ── Re-export Layer (NO LOGIC - only re-exports)│   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  MIGRATIONS: src/lib/db/migrations/ (130 versioned SQL files)             │
│  • Idempotent, transaction-wrapped                                        │
│  • Run on startup via getDbInstance()                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.4 Authentication & Authorization

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AUTHZ ARCHITECTURE (Route Guards)                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  src/server/authz/routeGuard.ts  ── isLocalOnlyPath(), extractApiKey()    │
│                                                                             │
│  TIER 1: PUBLIC (no auth)                                                   │
│  • /api/healthz, /api/status, /api/v1/models                              │
│  • /.well-known/agent.json (A2A)                                          │
│                                                                             │
│  TIER 2: LOCAL ONLY (loopback enforcement BEFORE auth)                    │
│  • /api/mcp/*          ── Spawns child processes                          │
│  • /api/cli-tools/*    ── Runtime execution                               │
│  • /api/services/*     ── npm install, node spawn                         │
│  • /dashboard/providers/services/*/embed/                                  │
│  • Enforcement: isLocalOnlyPath() → 403 if not localhost                  │
│                                                                             │
│  TIER 3: API KEY REQUIRED                                                  │
│  • /api/v1/chat/completions                                                │
│  • /api/v1/embeddings                                                      │
│  • /api/v1/completions                                                     │
│  • Validation: isValidApiKey() → policy check                             │
│                                                                             │
│  TIER 4: DASHBOARD (Session/JWT)                                          │
│  • /dashboard/*                                                            │
│  • Cookie-based session (iron-session)                                    │
│  • RBAC: OWNER/MEMBER/DEVELOPER/SECURITY/BILLING/VIEWER                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```
