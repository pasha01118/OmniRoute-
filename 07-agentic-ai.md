# 07 — Agentic AI & Autonomy Review

> **Verification basis.** OmniRoute `5764027`, version `3.8.50`. Tool, scope, and skill counts were **executed**, not copied from documentation — and three widely-repeated numbers turned out to be wrong.

---

## 1. The Autonomy Stack

OmniRoute exposes four distinct agentic surfaces over one provider fabric. They are frequently conflated in the marketing copy; they are architecturally separate.

```
                      ┌───────────────────────────────────────────┐
                      │            Provider fabric                │
                      │   338 providers · 20 combo strategies     │
                      │   circuit breakers · account fallback     │
                      └───────────────┬───────────────────────────┘
                                      │
        ┌──────────────┬──────────────┼──────────────┬────────────────┐
        ▼              ▼              ▼              ▼                ▼
   ┌─────────┐   ┌──────────┐   ┌──────────┐   ┌─────────┐   ┌────────────┐
   │   MCP   │   │   A2A    │   │  Skills  │   │ Memory  │   │   Evals    │
   │ server  │   │  server  │   │ executor │   │FTS5+Qdr │   │  harness   │
   │108 tools│   │ 6 skills │   │ sandbox  │   │         │   │            │
   │18 scopes│   │5 states  │   │container │   │decay    │   │            │
   └────┬────┘   └────┬─────┘   └────┬─────┘   └────┬────┘   └─────┬──────┘
        │             │              │              │              │
        └─────────────┴──────────────┴──────┬───────┴──────────────┘
                                            │
                       ┌────────────────────▼────────────────────┐
                       │  Guardrails: promptInjection · piiMasker │
                       │  credentialMasker · modalityBridge      │
                       │  Agent-bridge / MITM (capture 3rd-party)│
                       └─────────────────────────────────────────┘
```

---

## 2. MCP Server

### 2.1 Size and shape

`open-sse/mcp-server/` is **11,751 TS lines**. `createMcpServer()` is at `server.ts:671`.

The critical design decision is at `server.ts:680-709`: `registerTool` is **overridden** so that every registration is wrapped in a scope-enforcing `filteredHandler`. There is no way to register a tool that bypasses scope checking — the enforcement is at the registration seam, not at each call site. This is the correct place for it.

### 2.2 Tool count: 108, not 105

`open-sse/mcp-server/toolCount.ts:1-15` explains that the tool collections are **not guaranteed disjoint**, and `countUniqueMcpTools()` (`:27-35`) unions names into a `Set` for exactly that reason.

Dedupe of `MCP_TOOLS` (43) plus the record collections:

| Collection | Count |
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
| **Raw sum** | **122** |
| **After dedupe (`Set`)** | **108** |

The 14-name overlap is the reason naive counting produces 105, 108, or 122 depending on where you stop. The `Set` is authoritative.

### 2.3 Scope count: 18, not 31

`open-sse/mcp-server/schemas/tools.ts` is the only scope registry. Measured contents:

```
read:tools          read:health        read:combos         write:combos
read:quota          execute:completions read:usage         read:models
execute:search      write:budget       write:resilience    pricing:write
read:cache          write:cache        read:compression    write:compression
read:proxies        read:catalog
```

18 scopes. `AGENTS.md` says 31. The same registry also carries a scope *inference* layer for HTTP routes (`src/server/authz/accessScopes.ts:52-62`) that derives `read` / `write` / `admin` from method and path prefix — a separate mechanism that should not be conflated with MCP tool scopes.

### 2.4 Transports: three, all real

| Transport | Entry point | Gate |
|---|---|---|
| **stdio** | `StdioServerTransport` (`server.ts:2`), `startMcpStdio` (`index.ts:4`) | local process |
| **SSE** | `src/app/api/mcp/sse/route.ts` GET+POST → `handleMcpSSE` (`httpTransport.ts:292`) | `settings.mcpTransport === "sse"` (`:23-28`) |
| **Streamable HTTP** | `src/app/api/mcp/stream/route.ts` POST+GET+DELETE → `handleMcpStreamableHTTP` (`httpTransport.ts:283`) | `settings.mcpTransport === "streamable-http"` (`:24-30`) |

Only one HTTP transport is active at a time, selected by settings. This is a deliberate, documented constraint — enabling both would double the tool surface exposed over HTTP.

### 2.5 Authentication: the full management ladder

Every MCP HTTP route calls `requireManagementAuth(request)` before dispatch:

```
src/app/api/mcp/sse/route.ts:32,42
src/app/api/mcp/stream/route.ts:44,50,56
```

`src/lib/api/requireManagementAuth.ts:46-124` resolves in order:

1. `isAuthRequired()` gate
2. Dashboard session JWT
3. Trusted-loopback internal service
4. CLI machine token
5. `oma_` access token → 401 / 403 / 503
6. **Header-only** API key with `manage` scope (`:88` forces `extractApiKey(request, {allowUrl:false})`)

**Plus a pre-auth locality gate.** Because MCP tools spawn child processes, `LOCAL_ONLY_API_PREFIXES[0]` is `/api/mcp/` (`routeGuard.ts:33-63`). Loopback is enforced *before* any credential check — so even a valid admin key cannot reach MCP over a non-loopback interface unless the runtime carve-out in `policies/management.ts:256-287` applies (it does: `/api/mcp/*` accepts `mcp:connect|manage|admin` from any locality, per issue #9159).

This two-layer design — locality first, then identity, then scope — is the strongest authorization structure in the codebase.

### 2.6 Scope enforcement

`scopeEnforcement.ts:111-133` — `evaluateToolScopes()` compares the tool's declared `scopes` against `extra.authInfo.scopes`, then `meta.scopes`, then an env fallback (`:93`). The verdict returns `missing_scopes` (`:133`), so a denial names exactly what is missing.

`httpAuthContext.ts` and `mcpCallerIdentity.ts` supply caller identity; `mcpCallerIdentity.ts` is what backs the `api_key_id` in the audit row.

### 2.7 Token cost management

MCP has its own context-pressure problem: 108 tool descriptions consume tokens on **every** request. Three mechanisms address it.

| Mechanism | File | Purpose |
|---|---|---|
| Cardinality reduction | `toolCardinality.ts` — `reduceToolManifest`, `readMcpToolProfileFromEnv` | drop duplicate/derived tools from the manifest |
| Description compression | `descriptionCompressor.ts` — `compressMcpRegistryMetadata` | shorten descriptions |
| Smart filtering | `engines/mcpAccessibility` — `smartFilterText`, `DEFAULT_MCP_ACCESSIBILITY_CONFIG` | drop low-value tools for constrained clients |

Plus tool-search over the registry: `toolSearch/register.ts` → `registerToolSearchTool(server, withScopeEnforcement)` (`server.ts:1068`), so a client can discover a tool by intent instead of receiving all 108 descriptions.

### 2.8 Audit

`open-sse/mcp-server/audit.ts:359-390` — `logToolCall()` writes:

```
tool_name
input_hash      ← SHA-256 of the input, NEVER the input
output_summary  ← first 200 chars
duration_ms
api_key_id      ← OMNIROUTE_API_KEY_ID
success
error_code
```

The contract is stated in `schemas/audit.ts:1-9`: *"Input data is never stored in clear text. Only SHA-256 hashes of input and truncated output summaries are persisted."* Audit failures are caught and never propagated — *"Never let audit failure break tool execution"* (`:388-391`). Query API clamps `limit` to 1–500 (`:403`).

### 2.9 Heartbeat

`runtimeHeartbeat.ts` — `startMcpHeartbeat`, `resolveMcpHeartbeatPath`, `isMcpProcessAlive` (`index.ts:7-12`), surfaced at `src/app/api/mcp/status/route.ts`. Stdio MCP runs as a child process, so liveness must be polled from outside.

---

## 3. A2A (Agent-to-Agent)

### 3.1 Shape

`src/lib/a2a/` is **520 lines** across 5 TS files plus 6 skill modules.

`taskManager.ts` (240 lines) — a singleton with 5 states, validated at `app/api/a2a/tasks/route.ts:9-15`:

```
submitted → working → completed
                     ↘ failed
                     ↘ cancelled
```

`taskExecution.ts:19-44` — `A2A_SKILL_HANDLERS` with **6** handlers, each lazily imported:

| Skill | File |
|---|---|
| `smart-routing` | `skills/smartRouting.ts` |
| `quota-management` | `skills/quotaManagement.ts` |
| `provider-discovery` | `skills/providerDiscovery.ts` |
| `cost-analysis` | `skills/costAnalysis.ts` |
| `health-report` | `skills/healthReport.ts` |
| `list-capabilities` | `skills/listCapabilities.ts` |

`AGENTS.md` says 5 skills; there are 6 files and 6 handlers.

`executeA2ATaskWithState()` (`:46-64`) transitions the task to `completed` with artifacts, or `failed` with the error message as an `error` artifact, then re-throws.

### 3.2 Streaming events

`streaming.ts` (149 lines) + `routingLogger.ts` (67 lines) emit A2A event types defined in `open-sse/mcp-server/schemas/audit.ts:44-58`:

```
provider_selected   fallback_triggered   budget_check
quota_check         streaming_started    streaming_ended
```

These are the observability events that make agent routing auditable — an operator can see *why* a provider was chosen, not just that one was.

### 3.3 Routes

```
/api/a2a/tasks               POST create, GET list
/api/a2a/tasks/[id]          GET status
/api/a2a/tasks/[id]/cancel   POST cancel
/api/a2a/status              GET service status
/.well-known/agent.json      Agent Card
```

`tasks/route.ts:52-55` — POST defaults `skill: "conductor"`, delegating to `createConductorTask` from `lib/conductor/hubProxy` (`lib/conductor/`: `boot`, `bridge`, `hubProxy`).

### 3.4 Finding S-2 (High): no in-handler authorization

Every MCP route calls `requireManagementAuth`. **The A2A task routes call it nowhere.** They depend entirely on the global pipeline classifying them as MANAGEMENT.

```bash
grep -rn "requireManagementAuth" src/app/api/a2a/
# → no output
# compare
grep -rn "requireManagementAuth" src/app/api/mcp/
# → sse/route.ts:32,42  stream/route.ts:44,50,56
```

Today this is safe because `classifyRoute` assigns MANAGEMENT to unclassified `/api/*` paths (`classify.ts:103-125`). It becomes an exposure if anyone:
- adds the path to `PUBLIC_READONLY_API_ROUTE_PREFIXES`,
- renames or restructures the route,
- edits `proxy.ts` `config.matcher`,
- or mounts the A2A handlers outside the Next middleware chain.

The MCP pattern should be applied to A2A. This is a 4-line change with no behavioural cost.

---

## 4. Skills Framework

`src/lib/skills/` — 18 files, 3,567 lines:

| Component | Purpose |
|---|---|
| `executor.ts` | skill execution |
| `registry.ts` | skill registration/lookup |
| `sandbox.ts` | execution isolation |
| `containerProvider.ts` | containerized execution |
| `builtin/` | built-in skill set |

`src/lib/skills/sandbox.ts` plus `containerProvider.ts` is the security-relevant surface: agent-executed code must be isolated. The `LOCAL_ONLY_API_PREFIXES` list includes `/api/plugins/`, `/api/middleware/` (which executes arbitrary JS via `new vm.Script`), and `/api/skills/collect/` — all treated as spawn-capable.

Related: `src/lib/plugins/` (14 files, 2,581 lines) with `loader.ts`, `scanner.ts`, `sdk.ts` — a plugin **scanner** exists, which is the right primitive for third-party code.

### 4.1 Agent Skills generation

`src/lib/agentSkills/` — 6 files, 1,220 lines: `generator.ts`, `openapiParser.ts`, `cliRegistryParser.ts`. This generates agent skills from OpenAPI specs and CLI registries, so a new API or tool can be exposed as a skill without hand-authoring.

### 4.2 Cloud agents

`src/lib/cloudAgent/` — 12 files, 1,417 lines: `agents/`, `registry.ts`, `julesApi.ts`, `credentials.ts`. Delegation to hosted agent runtimes, with credential management per agent.

### 4.3 Copilot engine

`src/lib/copilot/` — 5 files, 1,303 lines: `engine.ts`, `tools.ts`, `systemPrompt.ts`. An in-product copilot with its own tool set and prompt.

---

## 5. Memory

`src/lib/memory/` — 33 files, 7,917 lines.

| Component | Detail |
|---|---|
| `sqliteBackend.ts` | FTS5 full-text search |
| `qdrant.ts` | optional vector backend |
| `retrieval/` | retrieval strategies |
| `embedding/` | embedding providers |
| `typedDecay.ts` | **type-aware decay** — entries fade by type |

`typedDecay.ts` is the interesting design: not all memories should decay at the same rate. Facts, preferences, and transient context plausibly have different half-lives. This is a small module with a real conceptual payoff.

The FTS5 + optional-Qdrant split is a sensible default: full-text works with zero infrastructure, vectors when quality demands it.

`db/migrations` includes a `memory entries` retention cleaner (see `06` §7.2), so memory is not unbounded.

---

## 6. Guardrails

`src/lib/guardrails/` — 14 files, 3,425 lines.

| Module | Function | Default |
|---|---|---|
| `promptInjection.ts` | injection detection | **on** in the `/v1/chat/completions` pipeline (`route.ts:155-176` → `400 SECURITY_001`) |
| `piiMasker.ts` | PII masking in requests | `PII_REDACTION_ENABLED` default **false** |
| `credentialMasker.ts` | 20+ provider credential patterns | `CREDENTIAL_REDACTION_ENABLED` default **false** (opt-in) |
| `modalityBridge/` | cross-modality conversion | — |

`src/middleware/promptInjectionGuard.ts` is the request-path guard; it is backed by a dedicated CI check and nightly `promptfoo` runs (block mode) plus `garak` probes.

**Design note:** prompt-injection detection is **on by default** while PII and credential masking are **off by default**. That is a defensible split — an injection guard protects the *system*, whereas PII/credential masking alters *output* and is a deployment choice. The `AGENTS.md` Hard Rule #20 requirement that both PII flags stay `false` by default is an explicit, documented product decision.

`src/lib/compliance/providerAudit.ts:3-11` — `WARNING_PATTERNS` detects `[sanitizer]`, `prompt injection detected`, `content filtered`, `safety filter`, `policy violation` in provider responses, capped at 5 hits, depth 6, length 400. This is **upstream** safety-signal detection, distinct from OmniRoute's own guards.

---

## 7. Evals

`src/lib/evals/` — 3 files, 1,288 lines: `evalRunner.ts`, `runtime.ts`, `evalRunner/`.

CI includes `quality-extended` (advisory), `nightly-property.yml` (fast-check with `FC_SEED=random FC_NUM_RUNS=2000`), and `dast-smoke.yml` which runs `promptfoo` against the injection guard on PRs.

**Gap worth noting:** there is no per-provider quality eval in the measured CI job list. Given 338 providers with wildly different output quality, and routing decisions made by 20 strategies, a regression in "which provider answers a given prompt well" would not be caught by any existing gate. The evals module exists (`evalRunner.ts`); it is not wired into a blocking provider-quality comparison.

---

## 8. Agent Bridge / MITM Capture

The most distinctive capability: intercepting third-party coding-agent traffic (Claude Code, Codex, Cursor, Copilot, Zed, Kiro, Antigravity, openCode, trae) and routing it through OmniRoute's provider fabric.

### 8.1 Targets: 10

`src/mitm/targets/index.ts:26-37`:

| Target | Hosts | Endpoint |
|---|---|---|
| `antigravity` | 4 cloudcode hosts | `/v1internal:generateContent`, `:streamGenerateContent`, `:loadCodeAssist`, `:onboardUser` |
| `kiro` | `api.anthropic.com` | `/v1/messages` |
| `claude-code` | `api.anthropic.com` | `/v1/messages` |
| `codex` | `chatgpt.com` | `/backend-api/codex/chat/completions`, `/v1/chat/completions` |
| `copilot` | `api.githubcopilot.com`, `copilot-proxy.githubusercontent.com` | 3 default models |
| `cursor` | `api2.cursor.sh` | `/v1/chat/completions` |
| `zed` | `api.zed.dev` | `/v1/chat/completions` |
| `openCode` | `opencode.ai` | `/v1/chat/completions` |
| `ghe-copilot` | runtime `gheUrl` | `/chat/completions`, `/v1/chat/completions`, `/responses` |
| `trae` | `trae.invalid` (non-routable placeholder) | `viability: "investigating"` |

`kiro` and `claude-code` share `api.anthropic.com` — interception is therefore opt-in between them, which is correct: you cannot capture Claude Code traffic without also being able to capture Kiro's.

`trae` is a **deliberate, correctly-gated stub**: non-routable host, `viability: "investigating"`, and `handlers/trae.ts:16-23` throws `"Not yet implemented — Trae viability under investigation. See plan 11 §5."` This is the right way to advertise an unimplemented target.

`resolveTarget(hostname)` (`:43-52`) is an **exact** case-insensitive host match — no suffix matching, so `evil-api.anthropic.com.evil.com` cannot match.

`routeConnection(hostname, userBypass)` (`:67-79`) precedence: `bypass → target → passthrough`.

### 8.2 Bypass safety

`_internal/bypass.cjs:27-32` defaults:

```
/\.bank\./i
/(^|\.)gov(\.|$)/i
/(^|\.)okta\.com$/i
/(^|\.)auth0\.com$/i
```

Banking, government, and SSO hosts are exempt by default. This matters: those flows use certificate pinning and step-up authentication that break under interception, and a banking session captured in the inspector buffer would be a serious privacy incident. The defaults are the right ones and are tested (`mitm-passthrough.test.ts`, 10 cases).

`bypassGlobMatch` (`:42-64`) rejects patterns with more than 9 segments — bounding glob cost.

`isSelfLoopDestination` (`:127-147`) + `server.cjs:421-428` returns HTTP **508 Loop Detected** rather than forwarding a request into itself.

### 8.3 Format translation

`handlers/antigravity.ts:107-143` `convertGeminiToOpenAI`:
- `systemInstruction` → system message
- `contents` (with role `model` → `assistant`) → messages
- `generationConfig.maxOutputTokens/temperature/topP/stopSequences` → `max_tokens/temperature/top_p/stop`

`resolveGeminiSource` (`:65-75`) unwraps the cloudcode `.request` envelope (#4294).

`handlers/claudeCode.ts:26-44` rewrites `payload.model`, then pops **all** trailing `assistant` messages while `length > 1` — prefill stripping, and it deliberately never empties the array.

Six handlers (`codex`, `copilot`, `cursor`, `openCode`, `zed`, and `kiro` to `/v1/messages`) share an identical 55-line shape: rewrite model → `fetchRouter` → `pipeSSE` with body accumulation → `hookBufferUpdate` → catch → `hookBufferError` + `writeError`.

### 8.4 Traffic inspector

`src/mitm/inspector/` — 14 files. Capabilities that matter:

| Component | Capability |
|---|---|
| `sseMerger.ts` (316) | Rebuilds Anthropic / OpenAI / Gemini from SSE; `detectApiFormat` infers format from **chunk shape, not URL** (`:37-49`) |
| `conversationNormalizer.ts` (393) | Unified conversation view; `normalizeConversation` returns null unless `detectedKind === "llm"` (`:377-393`) |
| `kindDetector.ts` (87) | 18 host regexes + 7 path regexes + 4 body shapes + UA regex → `llm \| app \| unknown` |
| `contextKey.ts` (98) | Extracts the system prompt (Anthropic `system`, `messages[0].role==="system"`, Gemini `systemInstruction.parts[]`), hashes to 12 hex chars |
| `llmMetadataExtractor.ts` (183) | 20 provider host matchers, 7 api-kind path matchers, usage extraction across three formats |
| `pricing.ts` (57) | 10 hard-coded USD/MTok entries; first case-insensitive substring match wins (`:33-40`); `estimateCost` returns **null** when no price is found — never guesses |
| `processAttribution.ts` (107) | Maps a socket to a process via `/proc/net/tcp` inode → `/proc/<pid>/fd` symlink → `/proc/<pid>/comm` (Linux only) |
| `buffer.ts` (201) | 1,000-entry ring, 1 MB body cap with truncation marker, filter by profile/host/agent/source/sessionId/status |

`processAttribution.ts` is a genuinely useful capability: it answers "which process made this request?" by correlating the TCP socket's inode with `/proc/<pid>/fd`. That is how you distinguish a Claude Code request from a background poller on the same machine.

`pricing.ts` returning `null` rather than a default when a model is unknown is the right call for a cost dashboard — a wrong number is worse than no number.

### 8.5 Inspector capture and safety

- `agentBridgeHook.ts:42-101` — `recordRequestStart` masks the body, classifies `source: "custom-host"` vs `"agent-bridge"` via `isCustomHost`, attributes the process best-effort.
- `recordRequestComplete` (`:107-124`) sanitizes response headers, masks the response body, sets `proxyLatencyMs`, `upstreamLatencyMs`, and `totalLatencyMs` (the sum) — so the proxy overhead is separable from the provider latency.
- `httpProxyServer.ts` (272) — `buildFetchHeaders` (`:45-71`) strips hop-by-hop and framing headers but **deliberately keeps `Authorization` for the upstream**. Listens on `127.0.0.1` only (`:238-271`), default port 8080. `handleConnect` (`:171-231`) records the tunnel as `"TLS tunnel — for body capture, redirect host via Custom Hosts mode"` and pipes raw TCP with no TLS termination — an honest label rather than a silent no-op.
- `systemProxyConfig.ts` (294) — macOS `networksetup`, Linux GNOME `gsettings`, Windows `netsh winhttp`. All calls use argv arrays with a swappable `execImpl` (`__setExec`). Revert restores prior state; Windows revert is unconditional `netsh winhttp reset proxy`.

### 8.6 Credential handling on the interception path

`handlers/base.ts:132-152` `fetchRouter()`:
- Base `process.env.OMNIROUTE_BASE_URL ?? "http://127.0.0.1:20128"`.
- `Authorization: Bearer ${ROUTER_API_KEY}` — and the **client's** `Authorization` is stripped/overridden via `sanitizeHeaders`, along with hop-by-hop headers.
- No timeout and no abort signal on the fetch.

`server.cjs:440` — on **passthrough**, only `Host` is rewritten; `Authorization` and `Cookie` forward verbatim to the genuine host. Correct: passthrough is a transparent proxy, so the real credentials must reach the real host.

### 8.7 Alias mapping

`src/lib/db/models/mitmAlias.ts` (32 lines) — `getMitmAlias(tool?)` reads `key_value WHERE namespace='mitmAlias'`, `setMitmAliasAll` writes with `INSERT OR REPLACE` then `backupDbFile("pre-write")`.

`agentBridgeMappings.ts:68-75` syncs agent-bridge mappings into the `mitmAlias` namespace for `MITM_ALIAS_AGENTS = {antigravity, claude-code, kiro}` (#8656).

`handlers/antigravity.ts` + `aliasConfig.cjs` apply `{model, reasoningEffort}` overrides; `hasInvalidReasoningEffort` rejects unknown values at the route boundary (`alias/route.ts:63-65`) even though the Zod schema is deliberately permissive (`cli.ts:27-36`).

### 8.8 Test coverage of the bridge

53 directly-named `mitm-*` unit test files, **263 test cases, 4,543 lines**, plus ~18 `agent-bridge-*` files. Notable:

| Test file | Cases | Asserts |
|---|---|---|
| `mitm-server-connect.test.ts` | 29 | CONNECT/bypass precedence, `parseBypassJson`, defaults, C1/C2 header contract, Hard Rule #12 sanitization |
| `mitm-antigravity-reasoning-effort-override.test.ts` | 14 | alias normalization, canonical `xhigh`, precedence vs `thinkingConfig` |
| `mitm-masksecrets.test.ts` | 13 | sk-/ak-/pk-/Bearer/opaque masking |
| `mitm-dnsConfig.test.ts` | 8 | env gates, graceful degradation |
| `mitm-startup-error-3606.test.ts` | 6 | stderr → error classification |
| `mitm-sudo-gate-822.test.ts` | 5 | Windows/root/NOPASSWD short-circuits |
| `integration/security-hardening.test.ts` | — | 316 lines |
| `e2e/agent-bridge-traffic-cross.spec.ts` | — | 155 lines |

An OOM/lock regression in bridge startup has explicit test coverage (`mitm-start-guard.test.ts`, 4 cases, TOCTOU race). That is the kind of test that only gets written after a production incident — and the fact that it exists at all suggests the incidents happened.

---

## 9. Findings

| ID | Sev | Finding | Section | Fix |
|---|---|---|---|---|
| **S-2** | **High** | A2A task routes have no in-handler authorization; they rely solely on pipeline classification. | §3.4 | Add `requireManagementAuth` to all 4 A2A routes. |
| **A-1** | Medium | `fetchRouter()` and the MITM upstream `fetch` have no timeout or `AbortSignal`. | §8.6 | `AbortSignal.timeout()`; a hung agent request currently holds forever. |
| **A-2** | Medium | No per-provider quality eval in CI. 338 providers with different output quality, 20 routing strategies, no regression gate on "did routing quality degrade". | §7 | Wire `evalRunner.ts` into a nightly provider-quality comparison. |
| **A-3** | Low | `detection/index.ts:22-32` has no `ghe-copilot` entry, so `detectAgent("ghe-copilot")` falls through to the `undefined` guard and returns `{installed:false}` even when configured. | §8.1 | Add a detector or document the intentional omission. |
| **A-4** | Low | `pricing.ts` has 10 hard-coded price entries with first-substring-match resolution; unknown models return `null`. Correct behaviour, but the table is a maintenance liability as providers change pricing. | §8.4 | Source from `lib/pricingSync.ts` as the default, keeping `null` as the fallback. |
| **A-5** | Info | MCP scope count is documented as 31 in `AGENTS.md`; the registry has 18. | §2.3 | Sync docs from `schemas/tools.ts`. |
| **A-6** | Info | MCP tool count documented as 105; dedupe yields 108. | §2.2 | Publish `countUniqueMcpTools()` in the docs build. |
| **A-7** | Info | A2A skill count documented as 5; there are 6. | §3.1 | Sync docs. |
| **A-8** | Info | Credential masker and PII masking are opt-in. | §6 | Document prominently; they are deliberately off. |
| **A-9** | Info | tproxy is iptables-only; no nftables path. | §8 (see 05 §8) | Document; relevant for nftables-only distros. |

---

## 10. Stale Numbers to Correct

| Claim | Documented | Measured | Where |
|---|---|---|---|
| MCP tools | 105 | **108** | `open-sse/mcp-server/toolCount.ts` |
| MCP scopes | 31 | **18** | `open-sse/mcp-server/schemas/tools.ts` |
| A2A skills | 5 | **6** | `src/lib/a2a/skills/`, `taskExecution.ts:19-44` |
| Routing strategies | 19 | **20** | `open-sse/services/combo/strategyDispatch.ts:47-68` |
| Providers | 291 | **338** | executed provider registry |
| Compression engines | 12 | **15** | `compression/engines/index.ts:18-45` |
| Compression savings | 15–95% (~89% avg) | **not reproducible** | `scripts/check/compression-budget-baseline.json` — 5 engines score identical to the no-op control |

**Root cause for the count drift is a single missing line.** `scripts/docs/gen-provider-reference.ts` imports `NOAUTH_PROVIDERS` at line 8 and never uses it in `main()` (`:180-196`), so the generated reference omits 10 providers. All count claims should be **generated from the executed registry**, never hand-maintained.

---

## 11. Recommendations

**Immediate**
1. Add `requireManagementAuth` to the 4 A2A routes (S-2) — 4 lines.
2. Add `AbortSignal.timeout()` to `fetchRouter` and the MITM upstream fetch (A-1).
3. Fix the docs generator to union all ten provider collections and emit the total (A-5/6/7 + the 291 claim).

**Short term**
4. Wire `evalRunner.ts` into a nightly per-provider quality comparison (A-2). Start with the 20 providers behind the default strategies, not all 338.
5. Add a `ghe-copilot` detector or document the omission (A-3).
6. Publish `countUniqueMcpTools()` output in the docs build so the MCP tool count can never drift again.

**Medium term**
7. Consider a provider-quality-weighted routing signal. With 338 providers and 20 strategies, the router currently optimises for *availability* (breakers, fallback) far more than *quality*. That is a defensible default, but it should be a stated one.
8. Expand the compression benchmark corpus so RTK is actually exercised, then substantiate or retract the 15–95% claim.
9. Add a process-attribution export, so the inspector's process-mapping capability is usable outside the dashboard.
