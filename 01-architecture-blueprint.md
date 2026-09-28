# OmniRoute Dashboard Redesign - Architecture Blueprint

> **Verification notice.** Counts in this document were measured against OmniRoute `5764027` (v3.8.50). Corrected from the original: providers **291 → 338**, MCP tools **105 → 108**, MCP scopes **31 → 18**, A2A skills **5 → 6**, routing strategies **19 → 20**, compression engines **12 → 15**, migrations **130 → 144**, pages **50+ → 114**, shared components **100+ → 135**, scripts **48+ → 212**, unit test files **1000+ → 4,201**. “Framer Motion” was removed from the current stack (it is not a declared dependency). The authZ tier 4 description was replaced — the session is a signed JWT, not `iron-session`. See [`00-comprehensive-master-plan.md`](./00-comprehensive-master-plan.md) §2.2 for the full correction list and [`10-ux-design.md`](./10-ux-design.md) §2 for the typeface/icon corrections.

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
│  │ Tailwind 4   │    │ Bun 1.3.14   │    │ Pino Logger  │    │ Vitest   │  │
│  │ Fumadocs UI  │    │ npm          │    │ Sharp        │    │ Playwright│ │
│  │ Recharts     │    │ SWC          │    │ Undici       │    │ Striker  │  │
│  │ Zustand      │    │ @tailwindcss │    │ WS           │    │ c8       │  │
│  │ next-intl    │    │ postcss      │    │ better-sqlite3│   │ Knip     │  │
│  │ Material Sym.│    │              │    │ @atjsh/llm   │    │ TypeCov  │  │
│  └──────────────┘    └──────────────┘    └──────────────┘    └──────────┘  │
│                                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────┐  │
│  │  BACKEND     │    │  PROXY/CORE  │    │  DATABASE    │    │  INFRA   │  │
│  ├──────────────┤    ├──────────────┤    ├──────────────┤    ├──────────┤  │
│  │ Express 5    │    │ open-sse     │    │ SQLite (WAL) │    │ Docker   │  │
│  │ HTTP Proxy   │    │ Executors    │    │ 144 Migrations│   │ Electron │  │
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
│  │ src/app/(dashboard)/dashboard/*  ── 114 Pages                      │   │
│  │ src/shared/components/*      ── 135 Shared Components             │   │
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
│  │ src/lib/*                  ── Domain Libraries (888 files)         │   │
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
│  │ src/lib/db/*               ── SQLite Domain Modules (144 Migs)     │   │
│  │ src/lib/db/core.ts         ── WAL Connection Singleton             │   │
│  │ src/mitm/*                 ── TLS Interception, Cert Management    │   │
│  │ src/sse/executors/*        ── Provider HTTP Clients (338)          │   │
│  │ src/sse/translator/*       ── Format Conversion (OAI↔Claude↔Gemini)│   │
│  │ src/sse/transformer/*      ── Responses API ↔ Chat Completions     │   │
│  │ open-sse/mcp-server/*      ── 108 MCP Tools, 18 Scopes, 3 Transp.  │   │
│  │ src/lib/a2a/*              ── A2A Task Server (6 skills, 5 states) │   │
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
│   ├── executors/                 # Provider HTTP clients
│   ├── translator/                # Format translators
│   ├── transformer/               # Responses API ↔ Chat
│   ├── services/                  # Combo, Cache, RateLimit
│   ├── mcp-server/                # MCP Server (108 tools, 18 scopes)
│   └── config/                    # Provider registry, constants
├── 📁 packages/                   # npm workspaces
│   └── browser-pool/              # Playwright browser pool
├── 📁 scripts/                    # Build/Dev/Quality scripts (212)
│   ├── build/, check/, dev/, docs/, i18n/, quality/, release/
├── 📁 src/                        # MAIN SOURCE (Next.js App)
│   ├── app/                       # Next.js App Router
│   │   ├── (dashboard)/           # Dashboard Route Group
│   │   │   └── dashboard/         # 114 Dashboard Pages
│   │   ├── api/v1/                # REST API Endpoints
│   │   ├── auth/, callback/       # OAuth Flows
│   │   └── .well-known/           # A2A Agent Card
│   ├── domain/                    # Domain Logic (Policy, Costs)
│   ├── lib/                       # Domain Libraries (888 files)
│   │   ├── db/                    # SQLite Modules + Migrations
│   │   ├── a2a/, mcp/, skills/    # Protocol Implementations
│   │   ├── memory/                # FTS5 + Qdrant
│   │   ├── guardrails/            # PII, Injection, Vision
│   │   └── compliance/            # Audit, Webhooks
│   ├── middleware/                # Next.js Middleware
│   ├── mitm/                      # TLS Interception
│   ├── models/                    # Data Models
│   ├── sse/                       # Streaming Engine
│   ├── server/                    # AuthZ, CORS, Origin
│   ├── shared/                    # Shared Code
│   │   ├── components/            # 135 React Components
│   │   ├── components/layouts/    # DashboardLayout, Sidebar, Header
│   │   ├── hooks/                 # Custom React Hooks
│   │   ├── constants/             # Providers, Sidebar, Pricing
│   │   ├── schemas/               # Zod Schemas
│   │   ├── types/                 # TypeScript Types
│   │   ├── utils/                 # Utility Functions
│   │   ├── validation/            # Validation Helpers
│   │   └── providers/             # React Context Providers
│   ├── store/                     # Zustand Stores
│   └── types/                     # Global Types
├── 📁 tests/                      # Test Suite (Unit, Integration, E2E)
│   ├── unit/                      # 4,201 Unit Test Files
│   ├── integration/               # Integration Tests
│   └── e2e/                       # Playwright E2E
└── 📁 _config files_              # package.json, tsconfig, eslint, etc.
```

### 2.2 Dashboard Page Map (114 Pages)

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
├── 🔐 providers/                         # Provider Connections (338)
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
│   ├── Table.tsx              ◄── Data tables (needs creation — see note)
│   ├── Tabs.tsx               ◄── Tab navigation (needs creation)
│   ├── Modal.tsx              ◄── Dialog/Confirm modals
│   ├── Badge.tsx              ◄── Status badges
│   ├── Avatar.tsx             ◄── User/Provider avatars
│   ├── Tooltip.tsx            ◄── Hover tooltips
│   └── Toast.tsx              ◄── Notification system (needs creation)
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

> **Note on `Table.tsx`.** It is listed as "needs creation" because it does not exist — but `src/shared/components/DataTable.tsx` **does**, along with `ColumnToggle.tsx` and `FilterBar.tsx`. Read `DataTable.tsx` before writing a new table, or the redesign will end up with two competing table components. The same applies to `Tabs.tsx` (only `docs/Tabs.tsx` exists, docs-scoped) and `Toast.tsx` (only `NotificationToast.tsx` exists, 208 lines). See [`10-ux-design.md`](./10-ux-design.md) §3.2.

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
│  │ MIDDLEWARE: src/proxy.ts → runAuthzPipeline (authz/pipeline.ts)    │   │
│  │ • classifyRoute() → PUBLIC | CLIENT_API | MANAGEMENT               │   │
│  │ • STRIP all spoofable auth headers before any handler runs          │   │
│  │ • body-size check, IP filter, drain check                           │   │
│  │ • policy.evaluate() → 200 / 401 / 403 / 503                        │   │
│  │ • CSRF + origin gate on cookie-authenticated mutations              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ NEXT.JS ROUTE: src/app/api/v1/chat/completions/route.ts            │   │
│  │ • 415 unless Content-Type: application/json                        │   │
│  │ • admitChatRequest() — capacity + body buffer BEFORE parse         │   │
│  │ • request.json() exactly ONCE (OOM fix #4380)                      │   │
│  │ • Permissive shape validation + model alias resolution              │   │
│  │ • injectionGuard() → 400 SECURITY_001                              │   │
│  │ • API key policy enforcement                                       │   │
│  │ • Handler Delegation → open-sse/handlers/chatCore.ts               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ HANDLECHATCORE (open-sse/handlers/chatCore.ts)                     │   │
│  │ • Cache Check (semantic cache, temperature===0 only)               │   │
│  │ • Rate Limit (per key/account)                                      │   │
│  │ • Combo Routing? (20 strategies)                                    │   │
│  │   ├─ YES: resolveComboTargets() → handleSingleModel() per target   │   │
│  │   └─ NO:  Direct to single model handling                           │   │
│  │ • On target exhaustion → globalFallbackModel (GLOBAL_FALLBACK)      │   │
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
│  │ • Circuit breaker + account fallback gate the call                  │   │
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
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ OBSERVABILITY                                                      │   │
│  │ • EventBus emit → WS sidecar (127.0.0.1:20132) → dashboard        │   │
│  │ • Call log summary row + on-disk body artifact (sha256)            │   │
│  │ • MCP tool audit (input_hash only, never cleartext)                │   │
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
│  │ States: CLOSED → DEGRADED → OPEN → HALF_OPEN                       │   │
│  │ Triggers: 500/502/503/504/408 (threshold 8 oauth / 12 apikey)     │   │
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
│  │ Overflow eviction: evictModelLockoutOverflow() bounds the map      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DEBUGGING GUIDANCE:                                                        │
│  • All keys skipped? → Check BOTH provider breaker + connection cooldown   │
│  • Provider excluded post-reset? → Use getStatus()/canExecute() not raw    │
│  • One key fails, others work? → Prefer connection cooldown               │
│  • One model fails? → Prefer model lockout                                │
│                                                                             │
│  NOTE: breaker thresholds above are the SOURCE values. `AGENTS.md` still    │
│  states 3/5/2, which is wrong — see 11-delivery-devops.md §8 (D-3).        │
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
│  │ (144 Migs)  │           │ (144 Migs)  │           │ (144 Migs)  │    │
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
│  MIGRATIONS: src/lib/db/migrations/ (144 versioned SQL files)             │
│  • Idempotent, transaction-wrapped                                        │
│  • Run on startup via getDbInstance()                                     │
│  • Driver: better-sqlite3 (optional) / node / bun / sqljs adapters        │
│  • Retention: cleanup.ts runs 8 scoped cleaners + artifact purge          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.4 Authentication & Authorization

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AUTHZ ARCHITECTURE (Route Guards)                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  src/proxy.ts → server/authz/pipeline.ts → runAuthzPipeline()              │
│  src/server/authz/routeGuard.ts  ── isLocalOnlyPath(), class extraction    │
│                                                                             │
│  FAIL-CLOSED DEFAULT: any path matching no rule becomes MANAGEMENT.         │
│  A new unclassified route is locked, not open. (classify.ts:121-125)        │
│                                                                             │
│  TIER 1: PUBLIC (no auth)                                                   │
│  • /api/healthz, /api/status, /api/v1/models                              │
│  • /.well-known/agent.json (A2A)                                          │
│                                                                             │
│  TIER 2: LOCAL ONLY (loopback enforcement BEFORE auth)                    │
│  • 26 spawn-capable prefixes + 2 regex patterns                           │
│  • /api/mcp/*          ── Spawns child processes                          │
│  • /api/cli-tools/*    ── Runtime execution                               │
│  • /api/services/*     ── npm install, node spawn                         │
│  • /api/middleware/*   ── arbitrary JS via new vm.Script                   │
│  • Enforcement: isLocalOnlyPath() → 403 LOCAL_ONLY if not local            │
│  • Bypass requires manage scope (mcp:connect for /api/mcp/* only)          │
│                                                                             │
│  TIER 3: API KEY REQUIRED                                                  │
│  • /api/v1/chat/completions                                                │
│  • /api/v1/embeddings                                                      │
│  • /api/v1/completions                                                     │
│  • Validation: isValidApiKey() → policy check                             │
│                                                                             │
│  TIER 4: DASHBOARD (Session/JWT)                                          │
│  • /dashboard/*                                                            │
│  • Cookie session: HS256 JWT in auth_token cookie, jwtVerify vs JWT_SECRET│
│  • Management password: bcrypt, cost factor 12                            │
│  • CSRF: HMAC-SHA256 token, 10-min TTL, enforced on unsafe methods        │
│                                                                             │
│  TIER 5: SCOPED ACCESS TOKENS (oma_ prefix)                               │
│  • inferRequiredScope(method, path) → read | write | admin                │
│  • Insufficient scope → 403 AUTH_SCOPE naming both scopes                 │
│                                                                             │
│  LOCALITY IS NEVER READ FROM A REQUEST HEADER.                             │
│  It comes from a token-stamped peer IP (peerStamp.ts) validated with       │
│  timingSafeEqual, and is force-downgraded to "remote" when a proxy hop is  │
│  detected. Both headers are stripped before any handler runs.              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

> **Gap.** The A2A task routes (`/api/a2a/tasks*`) never call `requireManagementAuth` in-handler, unlike every MCP route. They rely entirely on the pipeline's MANAGEMENT classification. See [`05-security-compliance.md`](./05-security-compliance.md) §3.8 (finding S-2).
