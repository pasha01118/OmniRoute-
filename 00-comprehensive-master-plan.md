# OmniRoute — Comprehensive Master Plan

> **Verification basis.** Every number, path, and behavioural claim in this report set was measured against the OmniRoute working tree at commit `5764027` (`feat(providers): integrate wave4 free-tier gateways (#9584)`), package version `3.8.50`. Claims that could not be reproduced are labelled **UNVERIFIED**; claims that were measured and *contradict* upstream marketing copy are labelled **CORRECTED**. Line references are `path:line` against that tree.

---

## 1. Executive Summary

OmniRoute is a self-hosted AI router: one OpenAI-compatible endpoint in front of hundreds of provider accounts, with automatic cross-provider fallback, token compression, an MCP server, an A2A agent protocol, a TLS-interception (MITM) bridge for capturing third-party coding-agent traffic, and a full Next.js operations dashboard.

This document is the entry point for the report set. It records what the codebase **actually is**, corrects several widely-repeated claims that do not survive measurement, and sequences the dashboard redesign so that work lands on a truthful foundation.

### 1.1 Ten findings that change how you should read the rest of this report set

| # | Finding | Measured value | Upstream claim | Evidence |
|---|---|---|---|---|
| F1 | **Provider count is stale everywhere** | **338** unique provider IDs registered | 291 | Executed `src/shared/constants/providers.ts` via `node --import tsx`; section count 10/25/227/34/12/12/12/2/3/1 = 338. `docs/reference/PROVIDER_REFERENCE.md:13` says 291 and `scripts/docs/gen-provider-reference.ts` never unions `NOAUTH_PROVIDERS`. |
| F2 | **MCP tool count is stale** | **108** unique tool names | 105 | `open-sse/mcp-server/toolCount.ts:27-35` dedupes a `Set` because the 12 tool collections overlap; the 108 figure is the deduped result. |
| F3 | **MCP scope count is stale** | **18** distinct scopes | 31 | `open-sse/mcp-server/schemas/tools.ts` is the only scope registry; 18 entries. |
| F4 | **Compression engines are all OFF by default, and the savings claim is not reproducible** | 15 engines registered; 5 of them produce **byte-identical output to the no-op `lite` baseline** on the project's own benchmark corpus | "12-engine compression", "15–95% tokens (~89% avg)" | `open-sse/services/compression/types.ts:386-454`; frozen `scripts/check/compression-budget-baseline.json` shows `rtk`, `caveman`, `aggressive`, `session-dedup`, `headroom` all scoring exactly `lite`'s numbers. |
| F5 | **The compression budget gate can never fail a no-op engine** | One-directional: only a *rise* fails | (implied to be a real gate) | `open-sse/services/compression/budget/budgetGate.ts:52-62` — "Falling cost always passes". |
| F6 | **The dashboard is not on DM Sans, Phosphor, or Framer Motion** | Font is **Inter**; icons are **Material Symbols** (self-hosted); `framer-motion` and `@phosphor-icons/react` are **not declared dependencies** and appear nowhere in `src/` | Reports 02/03/04 specify DM Sans + Phosphor + Framer Motion as if current | `src/app/layout.tsx:1,14-17,138`; `package.json` has `material-symbols: ^0.45.2`; `grep -riE "dm.?sans\|phosphor" src/` returns zero files. |
| F7 | **The redesign targets are mostly *new* files, not modifications** | 5 of 20 inventory targets do not exist yet | Report 04 lists 7 "new" files but mislabels 4 real components | `src/shared/components/Table.tsx`, `Tabs.tsx` (only `docs/Tabs.tsx` exists), `Toast.tsx` (only `NotificationToast.tsx`), `Icon.tsx`, `src/shared/lib/animations.ts` — all absent. `DataTable.tsx` already exists. |
| F8 | **Sidecar WebSocket has no send backpressure** | `sendTo()` never checks `ws.bufferedAmount`; no queue, no drain handling | (not previously stated) | `src/server/ws/liveServer.ts:263-267`. A slow dashboard client grows server memory unbounded. |
| F9 | **Login brute-force lockout is per-IP only, in-process, and not horizontally safe** | 5 failures / 15 min, module-global `Map`, no username dimension | (not previously stated) | `src/server/auth/loginGuard.ts:18-28,76-111`. All clients with an unparseable IP share one counter. |
| F10 | **A2A task routes have no in-handler authorization** | `/api/a2a/tasks*` never calls `requireManagementAuth`; relies solely on the global pipeline's MANAGEMENT classification | (not previously stated) | `src/app/api/a2a/tasks/route.ts` vs `sse/route.ts:32` / `stream/route.ts:44`. |

### 1.2 What is genuinely strong

- **Authz is unusually disciplined for a self-hosted app.** A single middleware (`src/proxy.ts:4-6`) routes *every* matched request through `runAuthzPipeline`, which classifies the route, strips all spoofable auth headers before any handler sees them, and fails closed on unclassifiable paths (`src/server/authz/classify.ts:121-125`).
- **Locality is derived from a stamped socket peer, never from `Host`.** `src/server/authz/peerStamp.ts` validates a token-stamped IP with `timingSafeEqual`, and `classifyStampedPeerLocality` *downgrades* to `"remote"` whenever a proxy hop is detected — closing the "tunnel in a JWT to a localhost route" class.
- **Test and quality density is exceptional.** 4,201 unit test files, ~32k test cases, 24 CI workflows, and a frozen quality ratchet that blocks on CodeQL alert count, secret findings, dependency vulns, and workflow-lint findings.
- **Secrets hygiene is systematic.** AES-256-GCM with an explicitly pinned 16-byte auth tag (`src/lib/db/encryption.ts:40`), never-store-input audit logging (SHA-256 of inputs only, `open-sse/mcp-server/audit.ts:359-390`), and a 20+ pattern credential masker.

---

## 2. Verified Platform Facts

Everything in this table was measured. Use it instead of upstream copy.

### 2.1 Identity and scale

| Property | Verified value | How verified |
|---|---|---|
| Package version | `3.8.50` | `package.json:3` |
| Source HEAD | `5764027` | `git -C /root/OmniRoute log -1` |
| Upstream remote | `https://github.com/diegosouzapw/OmniRoute` | `git remote -v` |
| Node engine | `>=22.22.2 <23 \|\| >=24.0.0 <27` | `package.json` engines |
| Workspaces | `open-sse`, `packages/browser-pool` | `package.json:412` |
| Framework | Next.js `^16.2.11` | `package.json:300` |
| React | `19.2.8` (pinned exact) | `package.json:312` |
| TypeScript | `^6.0.3` | `package.json:396` |
| Tailwind | `^4.0.0` | `package.json:394` |
| Runtime data store | SQLite (WAL), `better-sqlite3: ^13.0.2` **optional** | `package.json` optionalDependencies |
| Registered providers | **338** | executed provider registry |
| Provider sections | 10 (`NOAUTH`, `OAUTH`, `APIKEY`, `WEB_COOKIE`, `LOCAL`, `SEARCH`, `AUDIO_ONLY`, `UPSTREAM_PROXY`, `CLOUD_AGENT`, `SYSTEM`) | `src/shared/constants/providers.ts:275-284` |
| MCP tools | **108** unique | `open-sse/mcp-server/toolCount.ts` |
| MCP scopes | **18** | `open-sse/mcp-server/schemas/tools.ts` |
| MCP transports | 3 (stdio, SSE, streamable HTTP) | `open-sse/mcp-server/index.ts:4-18` |
| Combo routing strategies | **20** | `open-sse/services/combo/strategyDispatch.ts:47-68` |
| Compression engines registered | **15** | `open-sse/services/compression/engines/index.ts:18-45` |
| A2A skills | **6** | `src/lib/a2a/skills/` (6 files), `taskExecution.ts:19-44` |
| MITM intercept targets | **10** | `src/mitm/targets/index.ts:26-37` |
| SQL migrations | **144** `.sql` files | `ls src/lib/db/migrations/*.sql \| wc -l` |
| Dashboard routes (`page.tsx`) | **114** | `find "src/app/(dashboard)" -name page.tsx \| wc -l` |
| Shared components | **135** files | `find src/shared/components -name "*.tsx" -o -name "*.ts"` |
| Scripts | **212** files | `find scripts` |
| Unit test files | **4,201** | `find tests/unit -name "*.test.ts*" \| wc -l` |
| App API `route.ts` files | **655** across 99 top-level groups | `find src/app/api -name route.ts` |
| i18n locales | **43** | `config/i18n.json` |
| CI workflows | **24** | `ls .github/workflows` |

### 2.2 Claims that are wrong and must not be repeated

1. **"291 providers."** Under-counts by 47 (16%). `NOAUTH_PROVIDERS` (10) is imported by the docs generator at `scripts/docs/gen-provider-reference.ts:8` and then never used, so the generated reference omits them; adding them back still yields 328, not 338. **Fix:** union all ten collections in `main()` and make the total generated rather than hardcoded.
2. **"105 MCP tools" / "31 MCP scopes."** Both stale; see F2/F3.
3. **"12-engine token compression."** 15 are registered; 8 are benchmarked by default (`open-sse/services/compression/harness/benchmark.ts:295-304`).
4. **"15–95% tokens (~89% avg)."** Not reproducible from the repo's own frozen corpus. See §5.3 — this is the single most damaging claim in the project's marketing surface and needs a measurement plan, not a restatement.
5. **"19 routing strategies."** 20 are dispatched.
6. **"Circuit-breaker thresholds 3 / 5 / 2."** `AGENTS.md` states OAuth 3 / API-key 5 / local 2; `open-sse/config/constants.ts:222-268` sets 8 / 12 / 2. The reset intervals (60s/30s/15s) in AGENTS are correct. Additionally, the `local` profile is annotated "Not yet wired into `getProviderProfile()`" (`constants.ts:256`) — a live inconsistency between documentation and behaviour.
7. **"130 migrations."** 144 exist.
8. **"DM Sans + Phosphor + Framer Motion."** None of the three is present. See §6.2.

---

## 3. System Architecture (Verified)

### 3.1 Request path, end to end

```
client
  │
  ├─ src/proxy.ts:8-26  config.matcher  ── only matched prefixes enter the pipeline
  │    • /  /dashboard/*  /home*  /api/*  /v1*  /v1beta*  /chat/*  /responses*  /codex*  /models
  │
  └─ src/server/authz/pipeline.ts:253-419  runAuthzPipeline(request, {enforce:true})
       1. generateRequestId()                                          :260
       2. "/" → 307 → /dashboard                                       :262-267
       3. classifyRoute(normalizedPathname)                            :269-271
       4. corsRelaxOrigin = CLIENT_API or read-only PUBLIC only        :282-284
       5. drain check on /api/* → 503 SERVICE_UNAVAILABLE              :286-291
       6. body-size check on /api/* non-GET/OPTIONS                     :293-304
       7. STRIP every AUTHZ_TRUSTED_HEADERS + peer/via-proxy headers    :306-314  ◄── critical
       8. stamp x-omniroute-route-class / -request-id / -peer-locality :316-332
       9. OPTIONS → 204 preflight                                       :334-340
      10. IP filter (skipped for loopback)                              :361-380
      11. policy.evaluate()  → PUBLIC | CLIENT_API | MANAGEMENT         :382-393
      12. CSRF/origin gate (MANAGEMENT + dashboard_session + unsafe)    :395-409
      13. stampSubject → NextResponse.next() + CORS                    :411-419
            │
            ▼
     src/app/api/v1/chat/completions/route.ts:1-237
       • 415 if Content-Type ≠ application/json                       :83-99
       • admitChatRequest() reserves capacity + buffers body           :104-112
       • request.json() EXACTLY ONCE                                    :141   ◄── OOM fix #4380
       • chatCompletionsRouteShapeSchema.safeParse() (permissive)       :63-70
       • admitChatStructure → resolveModelAliasOnBody → injectionGuard :155-176
       • 400 SECURITY_001 on prompt injection
            │
            ▼
     src/sse/handlers/chat.ts  (1,988 lines) → chatHelpers.ts (972)
            │
            ▼
     open-sse/handlers/chatCore.ts → translator/ → executors/ → provider
```

### 3.2 The three route classes

`src/server/authz/classify.ts:50-126` assigns exactly one class to every request:

| Class | Paths | Policy |
|---|---|---|
| `PUBLIC` | `/`, `/dashboard/onboarding`, `/connect*`, read-only public API prefixes | `policies/public.ts:4-9` — always `allow(anonymous)` |
| `CLIENT_API` | `/api/v1/*`, `/api/v1beta/*` | `policies/clientApi.ts:57-101` — Bearer / `x-api-key` / `x-goog-api-key` / URL key; WS handshake metadata read allowed unauthenticated |
| `MANAGEMENT` | `/dashboard*`, all other `/api/*`, **and every unclassifiable path** (`:121-125`) | `policies/management.ts` — see §3.3 |

The fallback-to-MANAGEMENT default is the correct design: a new route added without classification is *locked*, not open.

### 3.3 Management policy tiers

`src/server/authz/policies/management.ts` evaluates in this order:

1. **Codex WS bridge carve-out** (`:97-115`) — `/api/internal/codex-responses-ws` with `x-omniroute-ws-bridge-secret` compared as SHA-256 digests via `timingSafeEqual`. Checked *before* the loopback gate.
2. **LOCAL_ONLY enforcement** (`:152-224`) — 26 spawn-capable prefixes + 2 regex patterns (`routeGuard.ts:33-68`). Non-loopback, non-private-LAN requests get a *manage-scope bypass attempt* (header-only key extraction, `{allowUrl:false}`); `/api/mcp/` accepts `mcp:connect`, everything else requires `manage`. Backend failure → `503 AUTH_BACKEND_UNAVAILABLE`. Otherwise `403 LOCAL_ONLY`.
3. **Internal service + model-sync allows** (`:240-250`).
4. **CLI machine token** (`:75-85`) — loopback-only, `timingSafeEqual`.
5. **MCP carve-out** (`:256-287`) — `/api/mcp/*` accepts `mcp:connect|manage|admin` from *any* locality (issue #9159).
6. **Auth-disabled mode** (`:289-292`) — MANAGEMENT allowed as anonymous when `isAuthRequired()` is false and the path is not always-protected.
7. **Dashboard session** (`:294-296`).
8. **`oma_` access token** (`:313-333`) — `ok` → allow; `error` → 503; `invalid` → 401 `AUTH_001`; `insufficient` → 403 `AUTH_SCOPE`.
9. **API key with `manage` scope** (`:335-355`), header-only.
10. **Final reject** (`:357-362`) — 403 if a `Bearer` header was present, else 401.

`ALWAYS_PROTECTED_API_PATHS` (`routeGuard.ts:116-120`) is `/api/shutdown`, `/api/providers/health-autopilot/actions`, `/api/settings/database` — never bypassed by the auth-disabled path.

### 3.4 Why `SPAWN_CAPABLE_*` is the load-bearing list

`routeGuard.ts:33-94` enumerates routes that can reach a child process, an arbitrary JS `vm.Script`, a `tar` spawn, or a `git checkout && npm install`. Three properties make the list hard to extend incorrectly:

- The `GET` exemption (`routeGuard.ts:198,217-228`) applies only on an **exact** string match, not a prefix.
- `matchesPrefix` (`accessScopes.ts:39-46`) requires a real segment boundary, so `/api/auth` never matches `/api/authz-inventory`.
- `isLocalOnlyBypassableByManageScope` (`:242-267`) refuses a bypass when the requested prefix **equals, is a child of, or is a parent of** any spawn-capable prefix — a bidirectional containment check, so both over-broad and over-narrow additions fail safe.

### 3.5 Realtime sidecar

`src/server/ws/liveServer.ts` (628 lines) runs as a **separate process on `127.0.0.1:20132`**, not inside Next.

| Constant | Value | Line |
|---|---|---|
| `DEFAULT_PORT` / `DEFAULT_HOST` | `20132` / `127.0.0.1` | `:40,:44` |
| `HEARTBEAT_INTERVAL_MS` / `HEARTBEAT_TIMEOUT_MS` | `15_000` / `35_000` | `:45-46` |
| `MAX_CLIENTS` | `500` | `:47` |
| `MAX_EVENTS_PER_SECOND` | `100` | `:48` |
| `MAX_PENDING_MESSAGES_PER_CLIENT` | `32` | `:49` |
| `MAX_PENDING_MESSAGE_BYTES` | `16_384` | `:50` |
| `BACKLOG_MAX` | `500` | `:85` |
| Internal-ingest body cap | `1_000_000` **string length**, not bytes | `:330` |

Connection gate order: message listener attached first (queued ≤32 frames / ≤16 KB) → origin check (`4003`) → capacity (`1013`) → auth (`4001`) → register → replay queue.

Auth accepts **header-only** tokens: `Authorization: Bearer` or `X-Live-WS-Token`, else the `auth_token` cookie verified with `jwtVerify` against `JWT_SECRET` (`:179-191`). Query-string tokens are explicitly rejected (`:124-126`) because they leak into access logs and `Referer`.

> **Two live defects worth filing.** (a) `sendTo()` (`:263-267`) never checks `ws.bufferedAmount` — a stalled dashboard tab grows server memory without bound. (b) `useLiveDashboard.ts:174-178` appends the API key to the **query string**, which the server deliberately does not read; cookie auth is the only path that actually works, so a token-only headless client is broken today.

### 3.6 Storage and caching

Four independent caches with *no shared invalidation*:

| Cache | Size | TTL | Notes |
|---|---|---|---|
| `cacheLayer.ts` | 50 entries / 2 MiB | 5 min | LRU; key = first 16 hex of `sha256(sorted JSON)` |
| `semanticCache.ts` memory tier | 50 / 2 MiB | 30 min | key prefixed with plaintext `apiKeyId` for per-key isolation |
| `semanticCache.ts` SQLite tier | **uncapped** | 1 h default | only TTL expiry and manual purge bound it |
| `db/readCache.ts` connections | 500 | 5 s | |
| `db/readCache.ts` connectionById | 10,000 | 5 s | |
| `db/readCache.ts` nodes / settings / pricing / lkgp | **unbounded** | 5 s / 30 s | `maxSize` 0 = no eviction |
| `db/reasoningCache.ts` | — | 2 h | keyed by `tool_call_id` |

Semantic caching only engages at `temperature === 0` (`semanticCache.ts:357-377`).

### 3.7 MITM / traffic-capture subsystem

`src/mitm/manager.ts` (806 lines) supervises a detached `server.cjs` child:

- **Start lock** `mitmStarting` guards a real TOCTOU race (`manager.ts:94`, upstream #2316).
- **Readiness is a 2,000 ms timer**, resolved early on child exit or any stderr line containing `❌` (`:672-701`). **There is no port probe** — a slow-but-healthy start can be reported failed.
- **Stop order is load-bearing:** DNS is removed *first* (`:770-806`), then the process is killed (SIGTERM → 1,000 ms → SIGKILL), then the password cache and PID file are cleared. Reordering reintroduces the #1809 ECONNREFUSED window.
- `server.cjs:324-333` resolves upstream IPs through a **hard-coded `8.8.8.8`** resolver and caches them for the process lifetime; `rejectUnauthorized` is on unless `MITM_DISABLE_TLS_VERIFY=1` (`:432`).
- `server.cjs:538-547` fetches upstream with **no timeout and no `AbortSignal`**.
- `server.cjs:559-564` always writes HTTP **200** for an SSE response, even when the upstream was non-2xx-shaped.
- tproxy mode requires a native addon (`IP_TRANSPARENT` → `CAP_NET_ADMIN`) and supports **iptables only** — no nftables path exists.
- In Docker builds the manager is deliberately stubbed: `Dockerfile:114` sets `OMNIROUTE_MITM_STUB=1`, and `manager.stub.ts:11-13` makes `startMitm`/`stopMitm` throw.

### 3.8 MCP

`open-sse/mcp-server/` is 11,751 TS lines. `createMcpServer()` (`server.ts:671`) overrides `registerTool` (`:680-709`) so **every** tool is wrapped in a scope-enforcing `filteredHandler`. 41 explicit `registerTool(` sites plus 12 record-shaped collections dedupe to **108** tool names.

Auth is the full management ladder (`sse/route.ts:32,42`, `stream/route.ts:44,50,56`). Because MCP tools spawn child processes, `LOCAL_ONLY_API_PREFIXES[0]` is `/api/mcp/` — loopback is enforced *before* authentication.

### 3.9 A2A

`src/lib/a2a/` is 520 lines: `taskManager` (5 states), `taskExecution` (6 lazily-imported skills), `streaming` (event types include `provider_selected`, `fallback_triggered`, `budget_check`, `quota_check`), `routingLogger`. Routes: `/api/a2a/tasks`, `/[id]`, `/[id]/cancel`, `/status`, plus an Agent Card at `/.well-known/agent.json`.

> **Gap.** Unlike every MCP route, the A2A task routes never call `requireManagementAuth`. They inherit protection from the global pipeline's MANAGEMENT classification alone — one `config.matcher` edit away from exposure. See §7, S-3.

---

## 4. Redesign Scope

### 4.1 What already exists (do not rebuild)

| Asset | Location | Size |
|---|---|---|
| App shell | `src/shared/components/layouts/DashboardLayout.tsx` | 139 lines |
| Sidebar | `src/shared/components/Sidebar.tsx` | **758 lines** — the report's "758 lines" claim is CORRECT |
| Header | `src/shared/components/Header.tsx` | 286 lines |
| Breadcrumbs | `src/shared/components/Breadcrumbs.tsx` | 182 lines |
| Card / Button / Input / Select / Modal / Badge / Avatar | `src/shared/components/` | 141 / 88 / 157 / 115 / 267 / 68 / 81 |
| Notification toast | `src/shared/components/NotificationToast.tsx` | 208 lines |
| **Data table (already exists!)** | `src/shared/components/DataTable.tsx` | — |
| Global CSS | `src/app/globals.css` | 590 lines |
| Theming | `src/store/themeStore.ts` | Zustand + `persist` + CSS vars + `dark` class; **7 colour presets**, default `coral` |
| Icon set | `material-symbols ^0.45.2` (self-hosted via `@import`) | — |
| Font | Inter via `next/font/google` in `layout.tsx:1` | — |

### 4.2 What must be created

| Asset | Status | Note |
|---|---|---|
| `src/shared/components/Table.tsx` | **absent** | `DataTable.tsx` exists — decide: wrap or rename |
| `src/shared/components/Tabs.tsx` | **absent** | only `src/shared/components/docs/Tabs.tsx` exists |
| `src/shared/components/Toast.tsx` | **absent** | only `NotificationToast.tsx` exists |
| `src/shared/components/Icon.tsx` | **absent** | must wrap Material Symbols, or adopt Phosphor as a new dep |
| `src/shared/lib/animations.ts` | **absent** | `src/shared/lib/` exists, so the path is valid |
| DM Sans | **not installed** | requires a `next/font/google` swap or a new dep |
| `@phosphor-icons/react` | **not installed** | `node_modules/@phosphor-icons` does not exist |

### 4.3 Two redesign decisions the report set must force to a conclusion

1. **Iconography.** The spec calls for Phosphor duotone 16–24px. The product ships Material Symbols, self-hosted, with an offline-friendly `@import`. Adopting Phosphor means a new runtime dependency, a dual-set migration window, and an accessibility re-audit of 135 components. **Recommendation:** keep Material Symbols as the base and add a thin `Icon.tsx` abstraction; revisit Phosphor only if a design review requires the duotone treatment.
2. **Typeface.** The spec calls for DM Sans. The product uses Inter via `next/font/google`, which is already the correct Next.js pattern and has zero layout-shift risk. **Recommendation:** keep Inter, or commit to DM Sans as a one-line `next/font` change with a full re-audit of the 590-line `globals.css` type scale.

Neither is a small change; both are *smaller* than the report set currently implies, and both are currently stated as if already done.

### 4.4 Phasing (supersedes Report 03's day-by-day plan)

Report 03's 39-day / 7-phase plan is not executable as written, because its Phase 1 assumes DM Sans and Phosphor are configuration toggles rather than new dependencies, and its Phase 6 references two page routes that do not exist (`resilience/page.tsx` → actual `resilience/connections/page.tsx`; `system/page.tsx` → actual `system/1proxy`, `system/proxy`, `system/mitm-proxy`).

| Phase | Scope | Gate |
|---|---|---|
| **0. Truth** | Correct the 8 stale claims in §2.2; fix the docs generator; align `AGENTS.md` breaker thresholds | `npm run check:docs-all` green |
| **1. Decisions** | Icon set, typeface, dark-only vs light+dark (the product currently ships light+dark+system — "dark-only" is a *removal*, not an addition) | written ADR |
| **2. Tokens** | `globals.css` rewrite, Tailwind `@theme inline`, contrast audit against measured ratios | WCAG AA automated pass |
| **3. Shell** | `DashboardLayout`, `Sidebar` (758 lines → staged), `Header`, `Breadcrumbs` | visual regression on shell only |
| **4. Primitives** | `Card`, `Button`, `Input`, `Select`, `Badge`, `Avatar`, `Modal` restyle + `Table`/`Tabs`/`Toast`/`Icon` creation | Storybook coverage |
| **5. Core pages** | `HomePageClient` (1,385), `providers` (1,951), `analytics` (155), `settings` (33) | per-page review |
| **6. Batch pages** | remaining 100+ `page.tsx` under a single per-page pattern | lint + a11y gates |
| **7. Motion** | `animations.ts` + Framer Motion (new dep) + reduced-motion | `prefers-reduced-motion` verified |
| **8. QA** | contrast, keyboard, screen reader, cross-browser, bundle budget | ratchet green |

**Report 03's line estimates are wrong for every large file** — it budgets `HomePageClient` at ~400 lines when it is 1,385, and `providers/page.tsx` at ~200 when it is 1,951. `settings/page.tsx` is 33 lines, not ~200. Rebudget from measured sizes.

---

## 5. Verification Ledger

### 5.1 Method

- Counts: `find` / `wc -l` against the working tree.
- Provider/MCP/compression/scope totals: **executed** via `node --import tsx` where the module is side-effect-free, otherwise derived by unioning the registries and deduping.
- Behavioural claims: read the implementation, cited `path:line`.
- Marketing copy: compared against `README.md`, `AGENTS.md`, `package.json:4`, `docs/reference/PROVIDER_REFERENCE.md:13`.

### 5.2 Reproduce these

```bash
cd /root/OmniRoute
# F1 — 338 providers
node --import tsx -e "import('./src/shared/constants/providers.ts').then(m=>console.log(Object.keys(m.AI_PROVIDERS).length))"
# F2 — 108 MCP tools
grep -c '"name":' open-sse/mcp-server/schemas/tools.ts   # then dedupe; collections overlap
# F4/F5 — compression baseline
cat scripts/check/compression-budget-baseline.json
sed -n '380,460p' open-sse/services/compression/types.ts
# F6 — no DM Sans / Phosphor
grep -rniE "dm.?sans|phosphor" src/ --include=*.ts --include=*.tsx --include=*.css || echo ABSENT
# F8 — no WS backpressure
sed -n '263,268p' src/server/ws/liveServer.ts
# F9 — IP-only lockout
sed -n '18,28p;76,111p' src/server/auth/loginGuard.ts
# §3.4 — spawn-capable list
sed -n '33,94p' src/server/authz/routeGuard.ts
```

### 5.3 The compression claim, specifically

The project's own reproducible corpus (`scripts/check/check-compression-budget.ts`, 2% tolerance) scores **mean compressed tokens per task** across three task groups. The frozen baseline:

| Engine | prose | tool-output | json |
|---|---|---|---|
| `lite` (no-op control) | 129 | 127 | 160 |
| `caveman` | **129** | **127** | **160** |
| `rtk` | **129** | **127** | **160** |
| `aggressive` | **129** | **127** | **160** |
| `session-dedup` | **129** | **127** | **160** |
| `headroom` | **129** | **127** | **160** |
| `ultra` | 92 | 116 | 117 |
| `ccr` | 129 | 81 | 64 |

Five engines are indistinguishable from doing nothing on the corpus the project itself ships. Two possibilities, and the team must determine which:

1. The benchmark corpus does not exercise those engines' trigger conditions (e.g. `rtk` only fires on `bash|shell|terminal|run_command|execute_command|exec|command` — `engines/rtk/index.ts:31` — and the corpus may contain no shell tool calls).
2. The engines are effectively no-ops in the current build path.

Either way, "15–95% tokens (~89% avg)" is **not currently supportable**, and the gate is structurally incapable of catching it because it only fails on a *rise* (`budget/budgetGate.ts:52-62`). **Recommended remediation:** make the gate bidirectional, add a per-engine minimum-effect assertion, and extend the corpus with shell-heavy and tool-output-heavy tasks.

---

## 6. Deliverable Inventory

### 6.1 Report set

| # | File | Subject |
|---|---|---|
| 00 | `00-comprehensive-master-plan.md` | This document |
| 01 | `01-architecture-blueprint.md` | System architecture, layers, request pipeline (corrected) |
| 02 | `02-design-system.md` | Tokens, components, motion (corrected) |
| 03 | `03-development-roadmap.md` | Phased plan, rebudgeted from measured sizes |
| 04 | `04-implementation-checklist.md` | QA and acceptance criteria (corrected inventory) |
| 05 | `05-security-compliance.md` | Authz, crypto, redaction, CI security gates |
| 06 | `06-performance-scalability.md` | Caching, breakers, backpressure, capacity |
| 07 | `07-agentic-ai.md` | MCP, A2A, skills, memory, guardrails, evals |
| 08 | `08-mcp-integrations.md` | MCP transports, scopes, tool surface, external integrations |
| 09 | `09-networking-proxy.md` | Egress, tunnels, origin/CORS model, MITM/tproxy |
| 10 | `10-ux-design.md` | Design system, accessibility, motion, page patterns |
| 11 | `11-delivery-devops.md` | CI/CD, quality ratchets, release, Docker, Termux |

### 6.2 Corrections applied to the original four reports

| Report | Correction |
|---|---|
| 01 | 291→**338** providers; 130→**144** migrations; "50+ pages"→**114**; "100+ components"→**135**; "48+ scripts"→**212**; "1000+ tests"→**4,201**; "105 MCP tools"→**108**; 19→**20** strategies; 12→**15** engines; removed "Framer Motion" from the current stack (it is not a dependency); added the verified authz pipeline, sidecar topology, and cache table. |
| 02 | DM Sans / Phosphor / Framer Motion marked **not present**; added the real baseline (Inter, Material Symbols, 7 colour presets, light+dark+system); added measured contrast targets instead of asserted ratios. |
| 03 | Rebudgeted from measured file sizes; `resilience/page.tsx` and `system/page.tsx` path corrections; added Phase 0 (truth) and Phase 1 (decisions); replaced the day-by-day plan with a gated phase plan. |
| 04 | Inventory corrected: 4 real components relabelled, `Table`/`Tabs`/`Toast`/`Icon`/`animations.ts` confirmed absent, `DataTable.tsx` discovered, every `~lines` estimate replaced with the measured value. |
| README | Broken `[LICENSE](LICENSE)` link resolved by adding `LICENSE`; navigation extended to the full 12-document set; `pnpm` → `npm` (the tree has `package-lock.json` and no pnpm lockfile); provider/tool/engine counts corrected. |

---

## 7. Engineering Findings

Ranked by production risk × confidence. Each is code-grounded; none requires speculation.

| ID | Severity | Finding | Evidence | Recommendation |
|---|---|---|---|---|
| **S-1** | High | **WebSocket send path has no backpressure.** `sendTo` writes to any `OPEN` socket; a slow client buffers indefinitely in the server process. | `src/server/ws/liveServer.ts:263-267` | Gate on `ws.bufferedAmount`; close or drop frames past a threshold; add a send queue with drain handling. |
| **S-2** | High | **A2A task routes have no in-handler authorization.** They rely solely on `config.matcher` + MANAGEMENT classification. | `src/app/api/a2a/tasks/route.ts` vs `sse/route.ts:32` | Add `requireManagementAuth` to every A2A route, matching the MCP pattern. |
| **S-3** | High | **Client/server WS auth mismatch.** The client puts the API key in the query string; the server reads headers only, by design. | `useLiveDashboard.ts:174-178` vs `liveServer.ts:124-127` | Use `X-Live-WS-Token`, or document cookie-only. |
| **S-4** | Medium | **Idle listeners are terminated.** `lastActivity` updates only on inbound messages, but the server's own `pong` frames do not refresh it. A subscribe-then-listen client is killed at 35 s. | `liveServer.ts:226,417-439` | Update `lastActivity` on every successful send, or track liveness separately from activity. |
| **S-5** | Medium | **CSRF tokens are replayable for their full 10-minute TTL.** No one-time-use tracking; MAC is bound to the session hash only. | `src/server/authz/csrf.ts:5-7,63-85` | Acceptable for CSRF; document the property, or shorten the TTL. |
| **S-6** | Medium | **Login lockout is per-IP, in-process.** No username dimension; all unparseable-IP clients share one counter; not horizontally safe. | `loginGuard.ts:18-28,54-57,76-111` | Add a per-account dimension; move state to the shared store if multi-instance. |
| **S-7** | Medium | **MITM upstream fetch has no timeout or abort signal.** A hung upstream holds the request forever. | `src/mitm/server.cjs:538-547` | Add `AbortSignal.timeout()`. |
| **S-8** | Medium | **MITM upstream DNS is pinned to `8.8.8.8` and cached for process lifetime.** Bypasses `/etc/hosts` (deliberate) but also breaks on networks that block it. | `server.cjs:324-333` | Make the resolver configurable; add TTL to the cache. |
| **S-9** | Medium | **MITM readiness is a 2,000 ms timer, not a probe.** | `manager.ts:672-701` | Poll the port before reporting success. |
| **S-10** | Low | **MITM reports HTTP 200 for every SSE response** regardless of upstream shape. | `server.cjs:559-564` | Propagate a meaningful status for non-streaming failures. |
| **P-1** | Medium | **Unbounded caches.** `nodesCache`, `settingsCache`, `pricingCache`, `lkgpCache` have no eviction; the semantic SQLite tier has no row cap. | `db/readCache.ts:26-28,68-131`; `semanticCache.ts:233-254` | Set `maxSize`; add a row/byte budget to `semantic_cache`. |
| **P-2** | Medium | **Compression budget gate is one-directional.** Only a rise fails, so an engine that does nothing passes forever. | `budget/budgetGate.ts:52-62` | Assert a minimum per-engine effect. |
| **P-3** | Low | **Local resilience profile is unwired** but documented as active. | `open-sse/config/constants.ts:256` | Wire it or annotate it as reserved in `AGENTS.md`. |
| **D-1** | — | **Docs generator omits `NOAUTH_PROVIDERS`.** | `scripts/docs/gen-provider-reference.ts:8` vs `:180-196` | Union all ten collections; generate the total. |
| **D-2** | — | **`AGENTS.md` breaker thresholds contradict source** (3/5/2 vs 8/12/2). | `AGENTS.md` vs `open-sse/config/constants.ts:222-268` | Sync from source; add a docs-drift check. |
| **D-3** | — | **8 stale marketing numbers** across README/package.json/PROVIDER_REFERENCE. | §2.2 | Fix at the generator, not the copy. |

---

## 8. Definition of Done

1. Every number in every report is either measured (with `path:line`) or explicitly marked **UNVERIFIED**.
2. No report asserts a dependency, font, or icon set that is absent from `package.json`.
3. No report's file inventory lists a file that does not exist without marking it "to create".
4. The marketing numbers in §2.2 are corrected at their source, and a CI check fails if a generated count drifts from the executed registry.
5. Findings S-1, S-2, S-3 are filed as issues with the reproduction commands in §5.2.
6. `npm run check:docs-all`, `npm run lint`, and `npm run test:unit` are green.
7. This report set is published and the remote tree is verified byte-for-byte after publication.
