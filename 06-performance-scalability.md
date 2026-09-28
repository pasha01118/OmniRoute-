# 06 — Performance & Scalability Review

> **Verification basis.** OmniRoute `5764027`, version `3.8.50`. All TTLs, caps, and thresholds were read from source; all counts were measured.

---

## 1. Performance Model

OmniRoute is a single-node, self-hosted edge. Its performance envelope is therefore governed less by horizontal scale and more by **per-process resource discipline**: how much it buffers, how much it caches without eviction, and whether it sheds load before allocating.

```
                       ┌──────────────────────────────────┐
   request ───────────▶│ proxy matcher (11 prefixes)      │
                       └──────────────┬───────────────────┘
                                      ▼
                       ┌──────────────────────────────────┐
                       │ 1 body-size check  (pre-parse)    │  ◄── first gate
                       │ 2 IP filter                      │
                       │ 3 authz policy                   │
                       └──────────────┬───────────────────┘
                                      ▼
      ┌───────────────────────────────────────────────────────────┐
      │ admitChatRequest()  capacity reservation + body buffer   │  ◄── second gate
      │   → request.json()  EXACTLY ONCE                         │
      │   → resolveModelAliasOnBody                              │
      │   → injectionGuard                                       │
      └──────────────┬────────────────────────────────────────────┘
                     ▼
      ┌───────────────────────────────────────────────────────────┐
      │ cacheLayer (50 / 2 MiB / 5 min)   semanticCache mem      │
      │ readCache (settings 5s, pricing 30s)  semanticCache disk  │
      │ reasoningCache (2h)  promptCache (accounting only)        │
      └──────────────┬────────────────────────────────────────────┘
                     ▼
      ┌───────────────────────────────────────────────────────────┐
      │ combo strategy (20 options) → accountSelector (P2C)      │
      │ circuit breaker (CLOSED/DEGRADED/OPEN/HALF_OPEN)         │
      │ accountFallback (lockout, cooldown, backoff 1m→20m)       │
      └──────────────┬────────────────────────────────────────────┘
                     ▼
      ┌───────────────────────────────────────────────────────────┐
      │ executor → provider API → stream → translator → SSE       │
      │ compression (15 engines, ALL disabled by default)         │
      └───────────────────────────────────────────────────────────┘
```

---

## 2. Load Shedding

### 2.1 Admission control runs before allocation

`src/app/api/v1/chat/completions/route.ts:104-112` calls `admitChatRequest()` **before** `request.json()`. Admission reserves heavyweight capacity and buffers the body under a hard byte bound. This ordering is the single most important performance property in the request path: a 500 MB body is rejected on its declared size, not after being parsed into a 500 MB object graph.

### 2.2 The body-size and IP gates

| Gate | Where | Order |
|---|---|---|
| Body size (per-route, pre-parse) | `pipeline.ts:293-304` | after drain check, before header strip |
| IP filter (skipped for loopback) | `pipeline.ts:361-380` | after `OPTIONS` short-circuit |
| Drain (`isDraining()`) | `pipeline.ts:286-291` → 503 `SERVICE_UNAVAILABLE` | before policy evaluation |
| Content-Type 415 | `route.ts:83-99` | before admission |
| Single JSON parse | `route.ts:141` | — |

The single-parse property is a documented OOM fix. The comment at `route.ts:126-138` records that a second `request.json()` on 270–550 KB agent payloads doubled heap residency and fed the OOM crash-loop (#4380 / #7862). This is exactly the kind of finding that should never be regressed, and it is not obviously enforced by a test — **worth adding a regression test that counts parses.**

### 2.3 The connection capacity gate is disabled by default

`src/sse/utils/backpressure.ts`:

| Property | Value | Line |
|---|---|---|
| Cap source | `OMNI_MAX_CONCURRENT_CONNECTIONS` | `:3-13` |
| **Default** | **`0` = disabled** | `:54-57` |
| Reject status | `429` with `Retry-After`, `X-RateLimit-Limit`, `X-RateLimit-Remaining: 0` | `:25-51` |
| Backoff formula | `retryAfter = max(1, ceil(active / cap × 30))` seconds | `:31` |
| Live count | read from `open-sse/services/sessionManager` — the same source as the health endpoint's `activeConnections` | `:3-13` |

**Finding P-6 (Medium).** A self-hosted router with a 500-client WebSocket limit and a 338-provider fan-out has no *default* protection against unbounded concurrent work. The mechanism is correct and the formula is sensible; it simply needs a sane default (for example, 4× the container's memory budget in requests) and a startup warning when unset.

### 2.4 Backoff schedule

`open-sse/config/constants.ts:198` — `BACKOFF_STEPS_MS = [60_000, 120_000, 300_000, 600_000, 1_200_000]`, i.e. 1m → 2m → 5m → 10m → 20m on rate limits. `EARLY_RETRY_MAX: 4` (`:331`) permits 4 transparent mid-stream re-opens, which is the right tolerance for a stream that dies after first byte.

---

## 3. Circuit Breaking & Resilience

### 3.1 The breaker

`src/shared/utils/circuitBreaker.ts` (644 lines):

| Property | Value | Line |
|---|---|---|
| States | `CLOSED → DEGRADED → OPEN → HALF_OPEN` | `:68-73` |
| `failureThreshold` (default) | 5 | `:169-171` |
| `resetTimeout` (default) | 30,000 ms | `:170` |
| `halfOpenRequests` | 1 | `:171` |
| Adaptive multiplier | up to `maxBackoffMultiplier` (default **16×**) | `:106-113` |
| `backoffEscalationCount` (default) | 3 open-cycles | `:113` |
| Per-failure-kind thresholds/cooldowns | configurable | `:78-85,99` |
| Persistence | `lib/db/domainState` → `domain_circuit_breakers` | — |
| Error classification | `isLocalStreamLifecycleError()` excludes `AbortError`, "controller is already closed", "client disconnected" | `:46` |

The error classifier is the important detail: a client hanging up must not count as a provider failure, or the breaker opens for reasons unrelated to provider health.

### 3.2 Provider profiles — and a documentation bug

`open-sse/config/constants.ts:222-268`:

| Profile | Transient cooldown | Rate-limit cooldown | Max backoff | Breaker threshold | Reset | Failure threshold | Window | Cooldown | Degradation | Multiplier | Escalation |
|---|---|---|---|---|---|---|---|---|---|---|---|
| `oauth` | 5 s | 60 s | 8 | **8** | 60 s | 10 | 15 min | 5 min | 5 | 8 | 2 |
| `apikey` | 3 s | 0 (respects `Retry-After`) | 5 | **12** | 30 s | 15 | 30 min | 10 min | 7 | 4 | 3 |
| `local` | 2 s | 5 s | 3 | **2** | 15 s | 2 | 5 min | 1 min | — | — | — |

**Finding P-3 (documented, low).** `AGENTS.md` states OAuth 3 / API-key 5 / local 2. The source says **8 / 12 / 2**. The reset intervals in AGENTS (60/30/15 s) are correct. Additionally, `constants.ts:256` annotates the `local` profile "Not yet wired into `getProviderProfile()`" — so the `local` row is aspirational, and `AGENTS.md` presents all three as active.

This matters operationally: an operator tuning breakers from the docs will set thresholds 60% lower than reality for OAuth and API keys, causing premature breaker trips.

### 3.3 Model lockout

`open-sse/services/accountFallback.ts` (2,050 lines) is the most stateful resilience component:

| Concern | Symbol |
|---|---|
| Model lockout | `lockModel` (`:548`), `recordModelLockoutFailure` (`:614`), `isModelLocked` (`:831`) |
| Cooldown selection | `selectLockoutCooldownMs` (`:604`) |
| Overflow eviction | `evictModelLockoutOverflow` (`:533`) — bounds the lockout map |
| Per-model quota | `hasPerModelQuota` (`:732`) |
| Connection cooldown | `recordProviderFailure` (`:982`), `recordProviderSuccess` (`:1047`) |
| Failure classification | `isProviderFailureCode` (`:1110`) |
| `Retry-After` parsing | `parseRetryAfterFromBody` (`:1142`) |
| Quota cooldown | `getQuotaCooldown(backoffLevel)` (`:1419`) |

Split across `accountFallback/{cooldownCap, exactModelLock, lockoutEviction, nonRetryableUpstream}.ts`. The presence of an explicit **eviction** path is a good sign — unbounded lockout growth is the classic failure mode here, and it is handled.

### 3.4 Warmup breaker stores

`src/lib/warmupScheduler/circuitBreakerFactory.ts:76` prefers `REDIS_URL` (ioredis with `maxRetriesPerRequest:3`, `connectTimeout:3000`, `lazyConnect:true`, `retryStrategy:()=>null`) and falls back to SQLite.

**Known ceiling, documented in the source** (`:66-70`): once the SQLite store is cached it is never re-probed, so a Redis outage that starts after first use is not recovered. `RedisCircuitBreakerStoreWithFailureReset` (`:39`) evicts a dead cached client, which mitigates but does not eliminate the issue.

### 3.5 Health checks

| Component | Schedule | Evidence |
|---|---|---|
| Local health check | initial 15 s delay, 5 s timeout, backoff `[30s, 60s, 120s, 300s]`, `Promise.allSettled` fan-out | `src/lib/localHealthCheck.ts:33-36` |
| Credential health | 300,000 ms default, 10,000 ms floor | `open-sse/config/constants.ts:299` |
| Provider autopilot | 6 remediation actions (clear breaker, clear cooldown, clear model lockout, reactivate connection, …) | `src/lib/monitoring/providerHealthAutopilot.ts:19-26` |
| DB integrity | skippable via `OMNIROUTE_SKIP_DB_HEALTHCHECK=1` | `src/lib/db/healthCheck.ts:39` |

`Promise.allSettled` is the right choice for a fan-out health check — one slow provider cannot stall the sweep.

---

## 4. Caching

### 4.1 Seven independent caches

OmniRoute has **no shared invalidation bus**. Each cache is independent, with its own key, TTL, and eviction policy.

| # | Cache | Size | TTL | Invalidation |
|---|---|---|---|---|
| 1 | `lib/cacheLayer.ts` | 50 entries / 2 MiB | 5 min | LRU; key = `sha256(sorted JSON).slice(0,16)` (`:54-57`) |
| 2 | `semanticCache` memory tier | 50 / 2 MiB | 30 min | `invalidateByModel` does a **full memory clear** (`:262`) |
| 3 | `semanticCache` SQLite tier | **uncapped** | 1 h default (`:233-254`) | TTL expiry + `invalidateStale` / `clearCache` |
| 4 | `db/readCache` connections | 500 | 5 s | `invalidateDbCache` |
| 5 | `db/readCache` connectionById | 10,000 | 5 s | `invalidateDbCache` |
| 6 | `db/readCache` nodes / settings / pricing / lkgp | **unbounded** (`maxSize: 0`) | 5 s / 30 s | `invalidateDbCache` |
| 7 | `db/reasoningCache` | — | 2 h | `tool_call_id` key |

### 4.2 Finding P-1 (Medium): unbounded caches

Two of the seven are unbounded, and one more has no row cap:

- `nodesCache`, `settingsCache`, `pricingCache`, `lkgpCache` have `maxSize: 0` (no eviction) at `db/readCache.ts:26-28,68-131`. The catalog and settings tables grow with provider count and user configuration, so a long-running install with heavy configuration churn grows these monotonically.
- The `semantic_cache` SQLite table has no row or byte ceiling; only TTL expiry and manual purge bound it.

**Mitigation already in place:** `invalidateDbCache` bumps two monotonic version counters (`combosCacheVersion`, `modelCatalogCacheVersion`) to break import cycles (`:178,205`), and every invalidation bumps the catalog version. So correctness is fine; only memory is unbounded.

**Recommendation:** set explicit `maxSize` for the four unbounded caches, and add a row-count budget to `semantic_cache` enforced by the existing `cleanup.ts` scheduler.

### 4.3 Semantic cache gating

`semanticCache.ts:357-377` — `isCacheableForRead` / `isCacheableForWrite` require an explicit numeric `temperature === 0`. Bypass via `X-OmniRoute-No-Cache: true`.

This is correct: caching a non-deterministic response would be a correctness bug, not just a quality issue. The explicit `=== 0` (rather than falsy) correctly excludes the case where `temperature` is absent, since absent-means-default-of-1.0 upstream.

### 4.4 Per-key isolation without a secret in the key

`semanticCache.ts:119-141` — the key is `sha256({model, normalizedMessages, temperature, top_p})` **prefixed with plaintext `apiKeyId`**. The comment explains the reasoning: the prefix is deliberate and documented as a non-credential, kept outside the digest to avoid the CodeQL `js/insufficient-password-hash` false positive. This gives per-key isolation while keeping the digest key-only. Correct trade-off, correctly documented.

### 4.5 Prompt cache is accounting, not storage

`src/lib/promptCache/` is 82 lines total — `index.ts` is one line and `prefixAnalyzer.ts` is the logic. This tracks **upstream provider** prompt-cache breakpoints; it is not a local response cache. Reports that describe it as a caching layer with a hit-rate benefit are wrong about the mechanism.

### 4.6 Cache health telemetry already exists

`src/lib/usage/cacheHealth.ts:8-19` documents a production diagnosis baked into the code: 24.8 M cached tokens read / 9.5 M written (`write/read = 0.385`), 18% of calls carrying 94% of the write, with p50/p90/p99 and per-model splits. The write amplification is a **0.385 ratio against a 30-min memory TTL and 1-h disk TTL** — the disk tier is writing heavily relative to what it returns. Worth tuning: either shorten the disk TTL or make disk promotion conditional on observed hit rate.

---

## 5. Streaming

### 5.1 Where the streaming code lives

An important structural fact: `src/sse/` contains **no SSE transport**. It is the auth/credential/routing/admission layer (27 files). The actual `text/event-stream` production lives in the `open-sse` workspace:

```
open-sse/utils/stream.ts
open-sse/utils/earlyStreamKeepalive.ts
open-sse/utils/sseHeartbeat.ts
open-sse/utils/jsonToSse.ts
open-sse/handlers/sseParser.ts
open-sse/handlers/chatCore/*
```

`src/sse/utils/backpressure.ts` is the **only** explicit backpressure control in `src/sse/`, and it is disabled by default (§2.3).

### 5.2 Stream lifecycle state machine

`src/sse/services/streamState.ts:12-53`:

```
initialized → connecting → streaming → completed
                                        ↘ failed
                                        ↘ cancelled
```

Terminal states have **no** outgoing transitions (`:38-53`). `transition()` records `elapsed`, stamps `firstChunkAt` on first entry to `streaming`, stamps `completedAt` for the three terminal states (`:90-123`). `getSummary()` (`:147-161`) returns `duration`, **`ttfb`** (`firstChunkAt - startedAt`, or `null`), `chunkCount`, `totalBytes`, `transitions.length`, `error`.

An enforced state machine with a `ttfb` metric per stream is exactly the instrumentation needed to find stream-level regressions.

### 5.3 Early-stream keepalive

`route.ts:195-211` wraps the streaming branch in `withEarlyStreamKeepalive(...)`, and `open-sse/utils/earlyStreamKeepalive.ts` handles the window before the first upstream byte. This is the right place to spend latency budget: provider TTFB on a cold connection is often 1–3 s, and proxies and load balancers will drop idle connections well before that.

### 5.4 Compression is inert by default

`open-sse/services/compression/types.ts:386-454`:

```ts
DEFAULT_COMPRESSION_CONFIG = {
  enabled: false,
  defaultMode: "off",
  autoTriggerMode: "lite",       // token threshold 0 → never auto-triggers
  autoTriggerTokens: 0,
  stackedPipeline: [
    { engine: "rtk",     intensity: "standard" },
    { engine: "caveman", intensity: "full"    },
  ],
  engines: { /* every engine: { enabled: false } */ }
}
```

`DEFAULT_RTK_CONFIG`: `enabled:false`, `intensity:"minimal"`, `enabledFilters:[]`, `maxLinesPerResult:120`, `maxCharsPerResult:12000`, `deduplicateThreshold:3`, `rawOutputRetention:"never"`, `rawOutputMaxBytes:1_048_576`, grouping and renderers off.

`DEFAULT_CAVEMAN_CONFIG`: `enabled:false`, `compressRoles:["user"]`, `minMessageLength:50`, `intensity:"lite"`, 6 preserve patterns (fenced code, inline code, URLs, paths, error headers, stack-trace lines).

The stacked pipeline is **seeded but inert**. One nuance: migration `035_standard_compression_config.sql:3-8` seeds `cavemanConfig` with `"enabled":true`, and because it is `INSERT OR IGNORE`, a migrated install has Caveman on while a fresh install does not. Migration `102_compression_engines_map.sql` derives the per-engine `enabled` map on read from legacy `defaultMode` / combo steps, with `activeComboId` defaulting to `null` (`db/compression.ts:534-583`).

**So: behaviour differs between fresh and migrated installs.** That is a real operational subtlety and should be documented in the setup guide.

### 5.5 Finding P-2 (Medium): the compression gate is one-directional

`open-sse/services/compression/budget/budgetGate.ts:52-62` — only a **rise** in mean compressed tokens fails; "Falling cost always passes".

The frozen baseline `scripts/check/compression-budget-baseline.json` scores mean compressed tokens per task across three task groups (prose / tool-output / json):

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

Five engines are byte-identical to the no-op control on the project's own corpus. Two hypotheses, and the team must pick one:

1. **The corpus does not exercise the triggers.** `rtk` only fires for `bash|shell|terminal|run_command|execute_command|exec|command` (`engines/rtk/index.ts:31`); a corpus of prose and JSON would never trigger it.
2. **The engines are effectively no-ops in the current build path.**

Either way, the marketing claim "15–95% tokens (~89% avg)" is **not currently supportable**, and the gate cannot detect it. This is the most damaging performance claim in the project and it deserves a measurement plan, not a restatement. See `00-comprehensive-master-plan.md` §5.3.

### 5.6 RTK pipeline, for reference

`engines/rtk/index.ts:1-13`:

```
detectCommandType
  → deduplicateRepeatedLines
  → groupSimilarLines
  → matchRtkFilter        (filters/ has 55 JSON definitions)
  → applyLineFilter
  → smartTruncate
  → stripCode
  → applyRenderer
```

`cache_control`-marked blocks are preserved byte-for-byte so upstream prompt-cache breakpoints are not invalidated (`index.ts:39-45`, rationale #3936). That is a subtle and correct interaction between compression and caching.

### 5.7 Finding S-1 (High): no WebSocket send backpressure

`src/server/ws/liveServer.ts:263-267`:

```ts
function sendTo(ws: WebSocket, msg: unknown): void {
  if (ws.readyState !== WebSocket.OPEN) return;
  ws.send(JSON.stringify(msg));
}
```

No `ws.bufferedAmount` check, no send queue, no `"drain"` handling. A dashboard tab that stops reading — backgrounded, throttled, on a slow connection — causes the server to buffer every subsequent event in process memory. The only pressure valve is the 35 s heartbeat, which is a **liveness** check, not a buffer check: the socket is OPEN and the client is responsive to TCP, so it is never terminated.

The 500-entry backlog (`:271-303`) and the per-client 32-message/16 KB early-queue cap (`:488-494`) bound the *ingest* side but not the *egress* side.

**Recommendation:** check `ws.bufferedAmount` before `send`; on exceeding a threshold (for example 4 MB), terminate the client with a distinct code so the dashboard reconnects and replays the backlog. This is the single highest-value performance fix in the codebase.

---

## 6. Realtime Sidecar Capacity

`src/server/ws/liveServer.ts`:

| Control | Value | Line |
|---|---|---|
| Max clients | 500 | `:47` |
| Per-client message rate | 100/s, fixed 1 s window | `:48,206-216` |
| Early-queue frames | 32 | `:49` |
| Early-queue bytes | 16,384 | `:50` |
| Heartbeat interval | 15,000 ms | `:45` |
| Heartbeat timeout | 35,000 ms | `:46` |
| Backlog | 500 entries | `:85` |
| EventBus history | 100 (its own array) | `lib/events/eventBus.ts:45` |
| Internal ingest body | 1,000,000 **string length** | `:330` |
| Port / host | 20132 / `127.0.0.1` | `:40,44` |

Rate limiting returns `RATE_LIMITED` **without closing** the socket (`:214-215`) — correct, since a burst of `subscribe` frames should not disconnect a legitimate client.

**Two observations:**

- The rate limiter is a fixed 1-second window (`client.eventCounterReset`), so a client can send 100 messages in the last millisecond of one window and 100 in the first millisecond of the next. A token-bucket or sliding window would be more accurate. Low severity at 100/s.
- `1_000_000` is measured as **JS string length (UTF-16 code units)**, not bytes. A string of 500k 4-byte emoji counts as 1,000,000 units = 2 MB of UTF-8. Minor, but the cap is not what it appears to be.

---

## 7. Database

### 7.1 Engine and migrations

| Property | Value | Evidence |
|---|---|---|
| Engine | SQLite, WAL journaling, singleton connection | `src/lib/db/core.ts` |
| Migrations | **144** `.sql` files | measured (reports previously said 130) |
| Better-sqlite3 | `^13.0.2`, **optionalDependency** | `package.json` |
| Adapters | betterSqlite / node / bun / sqljs | `lib/db/adapters/` |
| Integrity check | skippable via `OMNIROUTE_SKIP_DB_HEALTHCHECK=1` | `lib/db/healthCheck.ts:39` |

WAL mode is the right choice for a read-heavy dashboard with a write-heavy request logger. The optional-dependency declaration of `better-sqlite3` is deliberate — it lets the Bun/SQL.js/node adapter path work where the native build fails.

### 7.2 Retention and cleanup

`src/lib/db/cleanup.ts` (760 lines) implements 8 retention-scoped cleaners: `quota_snapshots` (`:34`), `call_logs` (`:64`), `usage_history` (`:92`), `compression_analytics` (`:141`), `mcp_tool_audit` (`:171`), plus a2a events, memory entries, domain cost history, compression cache stats, xp audit log, and compression run telemetry.

`cleanup/usagePurge.ts` supports 5 cutoff formats (`iso|date|dateHour|epochMs|epochSeconds`) and — importantly — `collectCallLogArtifactsBefore()`, which deletes orphaned filesystem artifacts as well as rows. Without that, the DB would be cleaned while the artifact directory grew forever.

### 7.3 Defaults

| Setting | Default | Env |
|---|---|---|
| `APP_LOG_RETENTION_DAYS` | 7 | `logEnv.ts:79` |
| `CALL_LOG_RETENTION_DAYS` | 7 | `:83` |
| `CALL_LOG_MAX_ENTRIES` | 10,000 | `:111` |
| `CALL_LOGS_TABLE_MAX_ROWS` | 100,000 | `:115` |
| `PROXY_LOGS_TABLE_MAX_ROWS` | 100,000 | `:132` |

### 7.4 Spend batching

`src/lib/spend/batchWriter.ts` (208 lines) exists specifically to avoid a per-request write to the spend tables. For a router that logs every call, this is the difference between a durable WAL and a fsync storm.

### 7.5 Docker

`docker-compose.yml` ships `redis:8.6.5-alpine` alongside the app with `REDIS_URL` wired. Redis is optional — every consumer has a SQLite fallback (`quota/storeFactory.ts:83-96` logs a credential-stripped URL and warns on misconfiguration). The credential-stripping (`redisUrl.replace(/:[^:@]*@/, ":***@")`) is a nice touch: connection strings routinely end up in logs.

`Dockerfile:2` uses `node:26-trixie-slim`, `USER node`, `DATA_DIR=/app/data`, exposes **20128**, and has a healthcheck. `Dockerfile:114` sets `OMNIROUTE_MITM_STUB=1` for the Docker build — the MITM manager is deliberately absent from container images.

---

## 8. Build & Bundle

| Property | Value | Evidence |
|---|---|---|
| Output | `standalone` | `next.config.mjs:160` |
| `distDir` | custom | `next.config.mjs` |
| serverAction body limit | default `50mb` | `next.config.mjs` |
| `typescript.ignoreBuildErrors` | **`true`** | `next.config.mjs:296` |
| `serverExternalPackages` | `better-sqlite3`, `pino`, … | `next.config.mjs` |
| `outputFileTracingIncludes` | includes `./src/mitm/server.cjs` | `next.config.mjs:219` |
| `images.unoptimized` | `true` | `next.config.mjs` |
| Frozen bundle-size ratchet | **8,045** (unit) | `config/quality/quality-baseline.json` |

**`typescript.ignoreBuildErrors: true`** is mitigated by a dedicated `dashboard-typecheck` CI gate and a frozen `dashboard-typecheck-baseline.json` plus `open-sse-typecheck-baseline.json`. So type errors are caught in CI, not at build. This is a deliberate trade (a type error in a vendored or generated file should not break the build) and the compensating control is present — but it means `npm run build` succeeding is **not** evidence of type safety.

---

## 9. Test & Quality Performance Gates

| Gate | Value | Notes |
|---|---|---|
| Coverage (CI floor) | **60/60/60/60** (statements/lines/functions/branches) | `ci.yml:956` |
| Coverage (frozen ratchet, "up" only) | 80.8 / 80.8 / 86.42 / 78.1 | `quality-baseline.json` |
| Per-module coverage ratchets | 30+ modules | chatCore 72.45, combo 85.42, accountFallback 96.78, auth 92.55, routeGuard 98.73, publicCreds 99.07 |
| Test files | 4,201 unit | measured |
| Test cases | ~32,299 | measured |
| Unit test shards | 8 (CI), 4 (quality workflow) | |
| E2E shards | 9, duration-balanced (LPT) | `e2e-timings.json` |
| Integration shards | 2 | |
| Coverage merge | 8 shards combined | |
| Node matrix | 24 (CI), 24+26 (nightly compat) | |
| Mutation testing | Stryker, 9 nightly batches + `mutationScore.*` ratchets for 30+ modules | |
| Property fuzz | `FC_SEED=random`, `FC_NUM_RUNS=2000` nightly | |
| i18n UI coverage | frozen at 100% | |

`routeGuard` at 98.73% and `publicCreds` at 99.07% are the right places to spend coverage — those are the security-critical modules. `chatCore` at 72.45% is the lowest and also the highest-complexity; that combination deserves scrutiny.

**`c8` floor of 60% is the real gate; Codecov is explicitly `informational: true`** in `codecov.yml`, with a philosophy note saying so. That honesty is welcome — most projects set the informational badge as if it were the gate.

---

## 10. Findings

| ID | Sev | Finding | Section | Fix |
|---|---|---|---|---|
| **S-1** | **High** | WebSocket `sendTo` has no `bufferedAmount` check, no queue, no drain handling — server memory grows unbounded behind a slow client. | §5.7 | Check `bufferedAmount`; terminate past a threshold with a distinct code so the client reconnects and replays. |
| **P-1** | Medium | Four `readCache` maps are unbounded (`maxSize: 0`); the `semantic_cache` SQLite tier has no row cap. | §4.2 | Set explicit `maxSize`; add a `semantic_cache` budget to the existing cleanup scheduler. |
| **P-2** | Medium | Compression budget gate only fails on a rise; 5 engines score identical to the no-op control. | §5.5 | Bidirectional gate; per-engine minimum-effect assertion; extend the corpus with shell/tool-output tasks. |
| **P-6** | Medium | `OMNI_MAX_CONCURRENT_CONNECTIONS` defaults to 0 (disabled). | §2.3 | Sane default + startup warning. |
| **P-7** | Low | `semanticCache.invalidateByModel` clears the entire memory tier rather than a model index. | §4.1 | Add a per-model index. |
| **P-8** | Low | Semantic disk tier write/read ratio is 0.385 with a 1-h TTL; heavy write amplification for modest return. | §4.6 | Shorten the disk TTL or gate promotion on observed hit rate. |
| **P-9** | Low | `local` resilience profile is documented as active but is not wired. | §3.2 | Wire it, or annotate it as reserved in `AGENTS.md`. |
| **P-10** | Low | WS rate limiter uses a fixed 1-second window; burst across a boundary bypasses it. | §6 | Token bucket or sliding window. |
| **P-11** | Info | `1_000_000` internal-ingest cap is UTF-16 code units, not bytes. | §6 | Compute from `Buffer.byteLength`. |
| **P-12** | Info | `typescript.ignoreBuildErrors: true` — a green build is not type-safety evidence (CI compensates). | §8 | Document; keep the `dashboard-typecheck` gate mandatory. |
| **P-13** | Info | Fresh and migrated installs differ in compression defaults (migration 035 enables Caveman). | §5.4 | Document in the setup guide. |

---

## 11. Prioritized Actions

**Immediate (low effort, high impact)**
1. Add the `ws.bufferedAmount` guard — ~15 lines, fixes an unbounded-memory path (S-1).
2. Set `maxSize` on the four unbounded `readCache` maps (P-1) — 4 numbers.
3. Add a startup warning when `OMNI_MAX_CONCURRENT_CONNECTIONS` is unset (P-6) — 3 lines.
4. Fix the `AGENTS.md` breaker thresholds to 8/12/2 and annotate `local` as unwired (P-9) — docs.

**Short term**
5. Make the compression budget gate bidirectional and add a per-engine minimum effect (P-2).
6. Add `AbortSignal.timeout()` to the MITM upstream fetch and the handler `fetchRouter` (S-7).
7. Add a `semantic_cache` row budget to `cleanup.ts` (P-1).
8. Add a regression test that counts `request.json()` calls, to protect the single-parse property.
9. Build a fresh-install vs migrated-install comparison test for compression config (P-13).

**Medium term**
10. Extend the benchmark corpus with shell-heavy and tool-output-heavy tasks so RTK is actually exercised, then re-measure and either substantiate or retract the 15–95% claim.
11. Token-bucket the WS rate limiter (P-10).
12. Per-model index for the semantic memory cache (P-7).
13. Investigate `chatCore` coverage (72.45%) against its complexity.

---

## 12. Reproduction

```bash
cd /root/OmniRoute

# Finding S-1 — no backpressure
sed -n '263,268p' src/server/ws/liveServer.ts

# Admission before parse
sed -n '104,145p' src/app/api/v1/chat/completions/route.ts

# Backpressure disabled by default
sed -n '3,13p;54,57p' src/sse/utils/backpressure.ts

# Unbounded caches
sed -n '26,28p;68,131p' src/lib/db/readCache.ts

# Compression defaults are inert
sed -n '386,408p' open-sse/services/compression/types.ts

# The five no-op engines
cat scripts/check/compression-budget-baseline.json

# The gate is one-directional
sed -n '45,70p' open-sse/services/compression/budget/budgetGate.ts

# Breaker thresholds (source of truth, not AGENTS.md)
sed -n '222,268p' open-sse/config/constants.ts

# ignoreBuildErrors
grep -n "ignoreBuildErrors" next.config.mjs

# Coverage gates
sed -n '950,960p' .github/workflows/ci.yml
grep -A2 '"coverage"' config/quality/quality-baseline.json | head -10
```
