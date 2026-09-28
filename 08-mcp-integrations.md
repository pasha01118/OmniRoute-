# 08 — MCP & Integrations Reference

> **Verification basis.** OmniRoute `5764027`, version `3.8.50`. Tool and scope counts were **executed** from the registries.

---

## 1. MCP at a Glance

| Property | Value | Evidence |
|---|---|---|
| Implementation size | **11,751** TS lines | `open-sse/mcp-server/` |
| Server factory | `createMcpServer()` | `server.ts:671` |
| Unique tools | **108** | `toolCount.ts:27-35` (Set-deduped) |
| Explicit `registerTool` sites | 41 | `server.ts` |
| Tool collections | 12 (MCP_TOOLS + 11 record shapes) | `index.ts:4-18` |
| Distinct scopes | **18** | `schemas/tools.ts` |
| Transports | 3 — stdio, SSE, streamable HTTP | `server.ts:2`; `httpTransport.ts:283,292` |
| Enforcement seam | `registerTool` override | `server.ts:680-709` |
| Audit | SHA-256 input hash, 200-char output summary | `audit.ts:359-390` |
| Locality gate | `LOCAL_ONLY_API_PREFIXES[0] = "/api/mcp/"` | `routeGuard.ts:33-63` |
| Heartbeat | file-based liveness for the stdio child | `runtimeHeartbeat.ts` |

---

## 2. Transport Configuration

### 2.1 stdio

`StdioServerTransport` (`server.ts:2`), exported as `startMcpStdio` (`index.ts:4`). The process is a child of the OmniRoute process, so liveness is polled via `runtimeHeartbeat` (`isMcpProcessAlive`) and surfaced at `/api/mcp/status`.

This is the transport desktop MCP clients (Claude Desktop, Cursor, Zed, etc.) use.

### 2.2 SSE

`src/app/api/mcp/sse/route.ts` — GET + POST → `handleMcpSSE` (`httpTransport.ts:292`).

Gated: the route returns early unless `settings.mcpTransport === "sse"` (`sse/route.ts:23-28`). `protectMcpSseResponse()` (`httpTransport.ts:169`) wraps the response.

### 2.3 Streamable HTTP

`src/app/api/mcp/stream/route.ts` — POST + GET + DELETE → `handleMcpStreamableHTTP` (`httpTransport.ts:283`).

Gated on `settings.mcpTransport === "streamable-http"` (`:24-30`). The DELETE verb is for session termination, which is why this transport needs three methods while SSE needs two.

### 2.4 Only one HTTP transport is active

The settings gate means an operator enables **either** SSE **or** streamable HTTP, never both. Enabling both would expose the 108-tool surface twice over HTTP. This is a deliberate, documented constraint — worth stating in operator documentation, because a user who edits the setting to `"sse,streamable-http"` gets neither.

Helper functions: `getMcpHttpStatus()` (`httpTransport.ts:309`), `isMcpHttpTransportReady()` (`:331`), `shutdownMcpHttp()` (`:338`).

---

## 3. Authentication Chain

Every MCP HTTP route authenticates **before** dispatch:

| Route | Lines |
|---|---|
| `src/app/api/mcp/sse/route.ts` | `:32`, `:42` |
| `src/app/api/mcp/stream/route.ts` | `:44`, `:50`, `:56` |

`requireManagementAuth` (`src/lib/api/requireManagementAuth.ts:46-124`) resolves in five tiers:

```
1. isAuthRequired() gate
2. dashboard session JWT (auth_token cookie, HS256, jwtVerify vs JWT_SECRET)
3. trusted-loopback internal service
4. CLI machine token (timingSafeEqual)
5. oma_ scoped access token          → 401 AUTH_001 / 403 AUTH_SCOPE / 503
6. header-only API key, manage scope   (allowUrl:false — #3300)
```

### 3.1 Two-layer gate: locality before identity

Because MCP tools spawn child processes, `/api/mcp/` is the **first** entry in `LOCAL_ONLY_API_PREFIXES` (`routeGuard.ts:33-63`). Loopback is enforced *before* any credential check.

The runtime carve-out in `policies/management.ts:256-287` then accepts `mcp:connect | manage | admin` from **any** locality for `/api/mcp/*` (issue #9159), with a backend throw → 503 rather than a silent allow.

**Result:** two independent layers must both be satisfied — the locality tier decides *whether the request is eligible*, and the scope tier decides *what it may do*. Neither alone is sufficient.

---

## 4. Tool Surface

### 4.1 Count reconciliation

`toolCount.ts:1-15` documents that the collections **overlap**, and `:27-35` unions names into a `Set` for that reason.

| Collection | Raw count |
|---|---|
| `MCP_TOOLS` | 43 |
| memory | 3 |
| skill | 4 |
| agentSkill | 3 |
| githubSkill | 3 |
| plugin | 8 |
| compression | 13 |
| pool | 6 |
| gamification | 8 |
| notion | 6 |
| obsidian | 22 |
| localCorpus | 3 |
| **Sum** | **122** |
| **`Set` dedupe** | **108** |

14 names appear in more than one collection. The `Set` is the only correct count.

### 4.2 Scopes (18)

From `schemas/tools.ts`:

```
read:tools           read:health         read:combos         write:combos
read:quota           execute:completions read:usage          read:models
execute:search       write:budget        write:resilience    pricing:write
read:cache           write:cache         read:compression    write:compression
read:proxies         read:catalog
```

Note the three non-`read`/`write` prefixes: `execute:completions`, `execute:search`, and `pricing:write`. These exist because "read" and "write" are insufficient descriptions — a completion **executes** inference (spends money) and a search **executes** an external query. The taxonomy is right.

### 4.3 Enforcement

`scopeEnforcement.ts:111-133` — `evaluateToolScopes()` compares the tool's declared scopes against, in order:

1. `extra.authInfo.scopes` (the authenticated subject)
2. `meta.scopes`
3. an environment fallback (`:93`)

The verdict returns `missing_scopes` (`:133`), so a denial is actionable rather than opaque.

`server.ts:701` wraps each handler in the enforcing `filteredHandler`, so **no tool can be registered without scope checking** — the override at `:680-709` is the only registration path.

### 4.4 Context-cost management

108 tool descriptions consume tokens on every request. Four mechanisms:

| Mechanism | File | Function |
|---|---|---|
| Cardinality reduction | `toolCardinality.ts` | `reduceToolManifest`, `readMcpToolProfileFromEnv` |
| Description compression | `descriptionCompressor.ts` | `compressMcpRegistryMetadata` |
| Smart filtering | `engines/mcpAccessibility` | `smartFilterText`, `DEFAULT_MCP_ACCESSIBILITY_CONFIG` |
| Tool search | `toolSearch/register.ts` → `registerToolSearchTool` (`server.ts:1068`) | discover a tool by intent rather than receiving all 108 |

`server.ts:75-80` wires the accessibility defaults. A client that cannot absorb 108 descriptions can get a reduced manifest plus a search tool, which is a better degradation than a hard failure.

### 4.5 Management-only tools

`tools/advancedTools.ts` carries `handleSimulateRoute`, `handleSetBudgetGuard`, `handleSetRoutingStrategy`, `handleSetResilienceProfile`, `handleTestCombo`, `handleCacheFlush`. These operate on the routing fabric itself, so they are the highest-consequence tools in the set and should carry the strictest scopes.

---

## 5. Audit Trail

`open-sse/mcp-server/audit.ts:359-390` — `logToolCall()` persists:

| Column | Content |
|---|---|
| `tool_name` | tool identifier |
| `input_hash` | **SHA-256 of the input — never the input** |
| `output_summary` | first 200 chars |
| `duration_ms` | latency |
| `api_key_id` | from `OMNIROUTE_API_KEY_ID` |
| `success` | boolean |
| `error_code` | on failure |

The contract is explicit in `schemas/audit.ts:1-9`: *"Input data is never stored in clear text. Only SHA-256 hashes of input and truncated output summaries are persisted."*

**Availability trade-off:** audit-write failures are caught and logged, never propagated (`:388-391`, *"Never let audit failure break tool execution"*). Correct for a tool server — an agent should not fail because an audit insert did.

Query API clamps `limit` to 1–500 (`:403`).

Retention is handled by `lib/db/cleanup.ts:171` (a `mcp_tool_audit` retention cleaner), so the audit table is not unbounded.

**Surfaces:** `/api/mcp/audit`, `/api/mcp/audit/stats`.

---

## 6. A2A — The Second Protocol

### 6.1 Shape

`src/lib/a2a/` is **520 lines** across 5 TS files + 6 skill modules.

```
taskManager.ts     240 lines   singleton, 5 states
taskExecution.ts              6 lazy skill handlers
streaming.ts        149        event types
routingLogger.ts     67        routing event persistence
skills/             6 files    lazily imported
```

States (validated at `app/api/a2a/tasks/route.ts:9-15`):

```
submitted → working → completed
                     ↘ failed
                     ↘ cancelled
```

### 6.2 Skills (6)

`taskExecution.ts:19-44`, each `await import("./skills/…")` — lazy load means a cold task pays nothing for the 5 skills it does not use:

| Skill | File |
|---|---|
| `smart-routing` | `skills/smartRouting.ts` |
| `quota-management` | `skills/quotaManagement.ts` |
| `provider-discovery` | `skills/providerDiscovery.ts` |
| `cost-analysis` | `skills/costAnalysis.ts` |
| `health-report` | `skills/healthReport.ts` |
| `list-capabilities` | `skills/listCapabilities.ts` |

`AGENTS.md` says 5. There are 6.

`executeA2ATaskWithState()` (`:46-64`) transitions to `completed` with artifacts, or `failed` with the message as an `error` artifact, then re-throws.

### 6.3 Streaming events

`streaming.ts` + `routingLogger.ts`, types at `schemas/audit.ts:44-58`:

```
provider_selected    fallback_triggered    budget_check
quota_check          streaming_started     streaming_ended
```

These are the events that make agent routing **auditable** — an operator can see why a provider was selected, not merely that one was.

### 6.4 Routes

| Route | Method |
|---|---|
| `/api/a2a/tasks` | POST create, GET list |
| `/api/a2a/tasks/[id]` | GET status |
| `/api/a2a/tasks/[id]/cancel` | POST cancel |
| `/api/a2a/status` | GET service status |
| `/.well-known/agent.json` | Agent Card |

`tasks/route.ts:52-55` — POST defaults `skill: "conductor"`, delegating to `createConductorTask` from `lib/conductor/hubProxy`.

### 6.5 Finding S-2: no in-handler authorization

```bash
grep -rn "requireManagementAuth" src/app/api/a2a/
# → (no output)
```

Every MCP route calls it. No A2A route does. They rely entirely on `classifyRoute` assigning MANAGEMENT to unclassified `/api/*` paths (`classify.ts:103-125`).

Safe today; fragile tomorrow. The four triggers that would expose it: adding the path to `PUBLIC_READONLY_API_ROUTE_PREFIXES`, renaming the route, editing `proxy.ts` `config.matcher`, or mounting the handlers outside the middleware chain.

---

## 7. Other Integration Surfaces

### 7.1 CLI tools (33 API files)

`src/app/api/cli-tools/` — 33 route files for per-tool settings, config, logs, backups, and status. Per-tool settings writers: claude, cline, codewhale, codex, crush, deepseek-tui, droid, forge, grok-build, hermes-agent, jcode, kilo, letta, omp, openclaw, pi, qwen, smelt. Shared helpers: `_lib/jsoncConfig.ts`, `backups/`, `logs/`, `status/`, `all-statuses/`, `apply/`.

The pattern: each `*-settings/route.ts` reads and writes a JSONC config for that CLI, with a shared JSONC parser. `all-statuses/route.ts` (209 lines) aggregates every tool's status in one call, which is what makes the CLI-code dashboard page viable.

Note: `cli-tools-no-mitm-tab.test.tsx` (4 cases) asserts the MITM tab was **removed** from the CLI-tools settings surface — the UI moved to the agent-bridge page. A test guarding a *removal* is unusual and good.

### 7.2 OAuth (59 files)

`src/lib/oauth/` — 59 files, 8,474 lines: `providers/`, `services/`, `config/`, `constants/`, `utils/`, `codexDeviceFlow.ts`, `credentialBlob.ts`.

`codexDeviceFlow.ts` is a device-code OAuth flow, which is the only workable auth for CLI agents without a browser. `credentialBlob.ts` handles the credential storage shape.

### 7.3 Memory backends

`src/lib/memory/` — 33 files, 7,917 lines: `sqliteBackend.ts` (FTS5), `qdrant.ts` (optional vector), `retrieval/`, `embedding/`, `typedDecay.ts`.

Zero-infrastructure default (FTS5) with an opt-in vector backend is the right layering. `typedDecay.ts` decays entries by type, implying different half-lives for facts, preferences, and transient context.

### 7.4 Plugins (14 files)

`src/lib/plugins/` — `loader.ts`, `scanner.ts`, `sdk.ts`. The **scanner** is the security-relevant component: third-party code is inspected before load.

`/api/plugins` and `/api/plugins/` are both in `LOCAL_ONLY_API_PREFIXES` (`routeGuard.ts:33-63`) — plugin loading is spawn-capable and localhost-only.

### 7.5 Webhooks

`src/lib/webhookDispatcher.ts` (7.9 KB). Outbound notification dispatch.

### 7.6 Cloud agents

`src/lib/cloudAgent/` — 12 files, 1,417 lines: `agents/`, `registry.ts`, `julesApi.ts`, `credentials.ts`. Delegation to hosted agent runtimes with per-agent credential management.

### 7.7 Remote-memory / corpus integrations

- `notion` — 6 MCP tools
- `obsidian` — 22 MCP tools (the largest single collection)
- `localCorpus` — 3 MCP tools
- `src/lib/localCorpus/` — 2 files, 660 lines

Obsidian at 22 tools is more than a third of the core `MCP_TOOLS` collection's size for one integration. Worth a scope-level review: should all 22 require the same scope?

### 7.8 Skills framework

`src/lib/skills/` — 18 files, 3,567 lines: `executor.ts`, `registry.ts`, `sandbox.ts`, `containerProvider.ts`, `builtin/`.

`sandbox.ts` + `containerProvider.ts` are the isolation boundary for agent-executed code. `/api/skills/collect/` is in `LOCAL_ONLY_API_PREFIXES`.

`src/lib/agentSkills/` — 6 files, 1,220 lines: `generator.ts`, `openapiParser.ts`, `cliRegistryParser.ts`. Generates skills from OpenAPI specs and CLI registries, so a new integration can be exposed without hand-authoring a skill.

---

## 8. Integration Security Posture

| Control | Status | Evidence |
|---|---|---|
| Locality before auth on MCP | Yes | `routeGuard.ts:33-63` |
| Scope enforcement at registration | Yes | `server.ts:680-709` |
| Denial names missing scopes | Yes | `scopeEnforcement.ts:133` |
| No tool bypasses scope | Yes (structural) | registration override |
| Audit hashes inputs | Yes | `audit.ts:359-390` |
| Audit failure non-fatal | Yes (documented) | `audit.ts:388-391` |
| Audit retention | Yes | `cleanup.ts:171` |
| Plugin scanner | Present | `lib/plugins/scanner.ts` |
| Skill sandbox | Present | `lib/skills/sandbox.ts` |
| **A2A in-handler auth** | **No** | §6.5 — finding S-2 |
| **Request timeouts on outbound agent fetches** | **No** | `handlers/base.ts:132-152`, `server.cjs:538-547` |

---

## 9. Recommendations

**Immediate**
1. `requireManagementAuth` on all 4 A2A routes (S-2).
2. `AbortSignal.timeout()` on `fetchRouter` and the MITM upstream `fetch` (A-1).
3. Regenerate the docs counts from `countUniqueMcpTools()` and `schemas/tools.ts` so 105/31/5 cannot drift again.

**Short term**
4. Review the 22 Obsidian tools' scope granularity — one integration holding 20% of the tool surface under one scope is a blast-radius concern.
5. Audit the scope assignments on `tools/advancedTools.ts` (`handleSimulateRoute`, `handleSetRoutingStrategy`, `handleCacheFlush`) — these mutate the routing fabric and should be the most tightly scoped tools in the set.
6. Document that only one HTTP transport may be active, and validate the setting value so `"sse,streamable-http"` is rejected at the settings boundary rather than silently disabling both.

**Medium term**
7. Expose MCP tool-call latency and error rates as a first-class dashboard metric; the audit table already has `duration_ms` and `error_code`.
8. Add a CI check that fails if `countUniqueMcpTools()` changes without a corresponding docs update.
9. Surface A2A streaming events (`provider_selected`, `fallback_triggered`) in the same timeline as MCP audit entries, so agent routing across both protocols is visible in one place.
