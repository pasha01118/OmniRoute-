# 09 — Networking, Proxy & Egress Review

> **Verification basis.** OmniRoute `5764027`, version `3.8.50`. All ports, headers, and mode-switching behaviour read from source.

---

## 1. Network Surfaces

OmniRoute presents six distinct network surfaces, each with a different trust model.

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. INBOUND  /v1/*  /v1beta/*  /chat/*  /responses*  /codex/*        │
│    OpenAI / Anthropic / Gemini compatible                            │
│    client_api_key auth · CORS relaxed for token auth                │
├──────────────────────────────────────────────────────────────────────┤
│ 2. INBOUND  /dashboard/*  /api/*                                     │
│    management · session or key auth · CSRF on mutations              │
├──────────────────────────────────────────────────────────────────────┤
│ 3. SIDECAR  127.0.0.1:20132  WebSocket                               │
│    dashboard live events · header-only token or cookie               │
├──────────────────────────────────────────────────────────────────────┤
│ 4. INTERNAL /__omniroute_event                                       │
│    loopback-only event ingest · NO token                             │
├──────────────────────────────────────────────────────────────────────┤
│ 5. EGRESS   provider APIs, tunnels, webhooks                         │
│    per-account routing · circuit breakers · proxy pools             │
├──────────────────────────────────────────────────────────────────────┤
│ 6. INTERCEPT  0.0.0.0:443 or 127.0.0.1:8080  MITM                   │
│    privileged · DNS + CA trust + optional TPROXY                     │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 2. Inbound API Surface

### 2.1 OpenAI-compatible surface

| Endpoint | File | Notes |
|---|---|---|
| `GET/OPTIONS /v1` | `app/api/v1/route.ts` | 22 lines, same builder as `/v1/models` |
| `POST /v1/chat/completions` | `app/api/v1/chat/completions/route.ts` | 237 lines — primary path |
| `POST /v1/completions` | `app/api/v1/completions/route.ts` | 124 lines |
| `POST /v1/embeddings` | `app/api/v1/embeddings/route.ts` | GET returns specialty catalog |
| `GET/HEAD /v1/models` | `app/api/v1/models/route.ts` | **explicit `HEAD`** — added to stop a ~6 s hang for OpenAI SDK probes (#6400) |
| `GET /v1/models/*` | `app/api/v1/models/[...model]/route.ts` | |
| `POST /v1/responses` | `app/api/v1/responses/route.ts` | Responses API |
| `POST /v1/messages` | `app/api/v1/messages/route.ts` | Anthropic Messages |
| `POST /v1/messages/count_tokens` | `app/api/v1/messages/count_tokens/route.ts` | |
| `ALL /v1/*` | `app/api/v1/[...omnirouteCatchAll]/route.ts` | 55 lines |
| `/v1beta/models` | `app/api/v1beta/models/route.ts` | Gemini → internal converter |

`/v1/models` schedules a background catalog refresh via `after()` rather than blocking — a small but real latency win, and the `HEAD` fix (#6400) is a good example of an SDK-probe compatibility issue being solved at the source.

99 top-level `/api/*` route groups, 655 `route.ts` files.

### 2.2 Chat completions pipeline

`app/api/v1/chat/completions/route.ts:1-237`:

| Step | Line | Detail |
|---|---|---|
| Init | `:35-43` | `ensureInitialized()` → `initTranslators()` singleton |
| Content-Type | `:83-99` | **415** unless `application/json` (RFC 7231, #6414) |
| Admission | `:104-112` | capacity reservation + body buffer **before** parse |
| **Single parse** | `:141` | `request.json()` once — second parse doubled heap on 270–550 KB agent payloads (#4380/#7862) |
| Schema | `:63-70` | `chatCompletionsRouteShapeSchema.safeParse()` — deliberately permissive, `.passthrough()` |
| Alias resolve | `:155-176` | `resolveModelAliasOnBody()` |
| **Injection guard** | `:155-176` | `injectionGuard()` → `400 SECURITY_001` |
| Stream | `:195-211` | `withEarlyStreamKeepalive(...)` |
| Compression echo | `:212,:219` | `withCompressionHeaderEcho(...)` |
| Errors | — | `errorResponse()` from `open-sse/utils/error.ts`, auto-sanitized |

The schema being permissive is correct, not lax: a 338-provider router cannot strictly validate a request body whose shape varies by provider. Validation is deliberately deferred to the executor layer, while the two things that *must* be strict at the edge — content type and body size — are enforced here.

---

## 3. Egress

### 3.1 Provider routing

20 combo strategies (`open-sse/services/combo/strategyDispatch.ts:47-68`):

```
priority   weighted        round-robin     context-relay   fill-first
p2c        random          least-used      cost-optimized  reset-aware
reset-window  headroom     strict-random   auto            lkgp
context-optimized  cache-optimized   fusion   pipeline   quota-share
```

`open-sse/services/combo.ts` is **3,274 lines**, with 43 modules in `services/combo/` covering target resolution, quota scoring, headroom ranking, session stickiness, shadow routing, fusion panels, response validation, and target timeouts.

`open-sse/services/accountSelector.ts:21-37` uses **Power-of-Two-Choices** (`crypto.randomInt`) with `getAccountHealth()` comparison — a well-established load-balancing algorithm that avoids the coordination cost of exact least-connections while keeping most of its benefit. Default strategy is `fill-first` (`:76`).

### 3.2 Three fallback layers

| Layer | Trigger | Behaviour | Evidence |
|---|---|---|---|
| **Combo exhaustion** | combo exhausts all targets | reads `settings.globalFallbackModel`, logs `GLOBAL_FALLBACK`, recurses to `handleSingleModelChat` | `src/sse/handlers/chat.ts:914-957` |
| **Emergency fallback** | HTTP **402** or budget keyword match | nvidia / `openai/gpt-oss-120b`, caps `max_tokens` to 4096, `skipForToolRequests`; on failure resumes the original provider's account fallback | `open-sse/services/emergencyFallback.ts:37-62`; wiring `chat.ts:1794-1855` |
| **Declarative chains** | policy-driven | `registerFallback()` sorts by ascending priority, persists to SQLite | `src/domain/fallbackPolicy.ts:56-68`; `/api/fallback/chains` |

The emergency fallback is gated on `OMNIROUTE_EMERGENCY_FALLBACK`, cached 500 ms (`emergencyFallback.ts:20`), default **enabled** unless `"false"`/`"0"` (`:79`). It only triggers on a 402 (payment required) or an explicit budget keyword — never on a generic error, so it cannot mask real provider failures.

`skipForToolRequests: true` is the right guard: silently rerouting a tool call to a different model can produce schema-invalid output.

### 3.3 Egress proxy pools

`src/lib/proxyEgress.ts` (13.5 KB), `src/lib/freeProxyProviders/` (8 files: `syncCycle`, `scheduler`, `iplocate`, `oneproxy`, `proxifly`, `webshare`), `src/lib/proxySubscription/` (`parse`, `fetchRetry`, `fetchGuard`).

`fetchGuard.ts` is the security-relevant module — a subscription-provided proxy URL is untrusted input that will receive production traffic. `config/alibaba-free-tier-allowlist.json` (dated `asOf`/`validUntil`) shows the pattern of time-bounded allowlists for provider endpoints, which is the right shape for a similar egress allowlist.

### 3.4 Tunnels

`src/lib/tailscaleTunnel.ts` (36 KB), `src/lib/cloudflaredTunnel.ts` (26 KB), plus `ngrok` referenced in the stack map. Tailscale is the privacy-preserving option and is implemented most substantially.

**Note:** the via-proxy downgrade in `authz/peerStamp.ts:89-99` exists specifically because a tunnel presents to Next as a loopback socket. Deploying OmniRoute behind a tunnel is supported, and the LOCAL_ONLY gates hold — but the operator should know that a tunnel makes the instance reachable, and the LOCAL_ONLY list is what stands between that reachability and a child-process spawn.

### 3.5 Upstream proxies

`src/shared/constants/providers.ts` includes an `UPSTREAM_PROXY` section (2 providers) and `src/lib/services/reverseProxy.ts`. `/api/management/proxies` and `/api/management/proxy-subscriptions` are the management surface.

---

## 4. Inbound Origin & CORS

### 4.1 CORS

`src/server/cors/origins.ts` (187 lines). The header comment is a rule, not a description: *"centralized allow-list, no wildcard default; per-route handlers must not set `Access-Control-Allow-Origin` themselves."*

| Property | Value | Line |
|---|---|---|
| Default allow-list | **empty** | `:27-39` |
| Allow headers | `Content-Type, Authorization, x-api-key, anthropic-version, x-omniroute-connection, x-internal-test, accept` | `:23-25` |
| Methods | `GET, POST, PUT, DELETE, PATCH, OPTIONS` | `:25` |
| Runtime setter | `setRuntimeAllowedOrigins(csv)` — persisted-settings layer | `:27-39` |
| `Vary: Origin` | appended whenever ACAO is set | `:157-160` |
| `Vary: Accept-Encoding` | when relaxed and status ≠ 204 (RFC 9110 §12.5.5, #6737) | `:168-170` |
| Header echo | `Access-Control-Allow-Headers` ← `Access-Control-Request-Headers` | `:173-176` |
| `*` handling | `CORS_ALLOW_ALL` accepts `"true"`/`"1"`, or legacy `CORS_ORIGIN === "*"` | `:60-67` |

`relaxForTokenAuth` (`:121-146`) is the one relaxation, and its documentation is precise: it opts the response into echoing the caller's origin when the explicit allow-list does not match, reserved for `/v1/*`, `/v1beta/*`, and read-only public endpoints that authenticate via headers browsers never auto-attach — and **never** paired with `Access-Control-Allow-Credentials`. Cookie-authenticated management routes must pass `false` (`:136-140`).

The pipeline enables it only for `CLIENT_API` and read-only `PUBLIC` classes (`pipeline.ts:282-284`), so a route cannot opt in by accident.

`getCorsStatus()` (`:108-119`) merges env + runtime, dedupes, sorts, and feeds `/api/settings/authz-inventory` — the CORS configuration is therefore visible in the security inventory UI.

### 4.2 Public-origin resolution

`src/server/origin/publicOrigin.ts` (270 lines).

**Env precedence** (`:21-25`): `OMNIROUTE_PUBLIC_BASE_URL`, `NEXT_PUBLIC_BASE_URL`, `NEXT_PUBLIC_APP_URL`.

**Candidate order** (`:219-237`):

```
request-url  →  configured  →  trusted-forwarded  (only if zero configured)  →  direct-local-host
```

**`trustProxyMode()`** (`:119-127`) reads `OMNIROUTE_TRUST_PROXY`:

| Value | Mode |
|---|---|
| falsy, `0`, `false`, `none`, `off`, `no`, `disable`, `disabled` | `none` |
| `true`, `1`, `loopback` | `loopback` |
| `private`, `lan` | `private` |
| **anything unrecognized** | **`none`** |

Fail-safe by default: a typo disables proxy trust rather than enabling it.

`trustsForwardedHeaders()` (`:129-140`) additionally requires the stamped peer to classify as loopback (for `loopback` mode) or loopback-or-LAN (for `private`). A forged `x-forwarded-proto` from a remote client cannot buy trust.

**Sanitizers:**
- `sanitizeForwardedProto` (`:92-96`) — literal `http`/`https` only.
- `sanitizeForwardedHost` (`:98-117`) — rejects `/ \ whitespace control-chars`, then re-parses via `new URL("http://" + host)` and rejects any result with `username`/`password`, a non-`/` pathname, a search, or a hash. This is a thorough defence against `evil.com/path` and `user:pass@host` style Host injections.
- `parseForwardedHeader` (`:75-90`) — parses only the first `Forwarded` element's `;`-separated `proto=`/`host=`. Multiple elements are ignored rather than merged, which avoids comma-ambiguity attacks.

**`Forwarded` beats `X-Forwarded-*`:** line 147 and line 150 (`??`) give the standardised header precedence.

### 4.3 The direct-LAN carve-out

`directLocalHostOrigin` (`:195-217`) is the subtlest code in this module, documented at length in the comment at `:177-194` (issue #5340). It permits direct LAN access without a configured public origin, but applies **two independent checks**:

1. The stamped peer must not classify as `"remote"` (`:200`).
2. The `Host` header itself must classify as loopback or private-LAN — never a domain name (`:205-207`).

The second check is what defeats DNS rebinding: an attacker who controls a domain resolving to `192.168.1.5` sends `Host: attacker.example`, which classifies as remote, and is rejected — even though the connection genuinely arrived from a LAN address.

The protocol is never taken from the attacker-controlled `Origin`; it comes from `x-forwarded-proto` (when trusted) or `requestUrlProtocol` (`:209-211`).

### 4.4 Browser-mutation origin

`validateBrowserMutationOrigin` (`:252-270`):

```
sec-fetch-site present and not same-origin|same-site|none  → reject (cross-site-fetch-metadata)
Origin header absent                                        → ALLOW
Origin unparseable                                          → reject (invalid-origin)
Origin not in candidate set                                 → reject (invalid-origin)
```

**No `Origin` header → allow** is correct: browsers always send `Origin` on cross-origin mutations, so its absence indicates a non-browser client (curl, CLI, server-side), which cannot be the target of a browser-based CSRF. The check that matters for CSRF is the CSRF token itself.

---

## 5. Authenticated Peer Resolution

This is the foundation everything else rests on.

`src/server/authz/headers.ts:27-54`:

```
PEER_IP_HEADER   x-omniroute-peer-ip      = <token>|<ip>     stamped pre-Next by scripts/dev/peer-stamp.mjs
VIA_PROXY_HEADER x-omniroute-via-proxy    = <token>|1         set when x-forwarded-for / x-real-ip was present
```

The header comment on `PEER_IP_HEADER` is unambiguous: *"NEVER decide locality from the Host header."* The `VIA_PROXY_HEADER` comment records the reason it exists — to close the da667836 class where a loopback socket is the proxy hop.

`peerStamp.ts`:

| Function | Line | Behaviour |
|---|---|---|
| `resolveStampedPeer(header, token)` | `:18-35` | requires `|`, non-empty IP, **equal-length** token, `timingSafeEqual`; else `null`. Pure, dependency-free. |
| `resolveStampedViaProxy(header, token)` | `:56-72` | same validation, true only when the payload is exactly `"1"` |
| `classifyStampedPeerLocality(peer, viaProxy, token)` | `:89-99` | classifies, then **downgrades to `"remote"` when via-proxy is set and locality isn't already remote** |

The equal-length check before `timingSafeEqual` prevents the throw that a length mismatch would cause, and the pure/dependency-free design makes it trivially testable.

**Both headers are stripped** before any handler runs (`pipeline.ts:306-314`), so no handler can read a client-supplied locality.

`classifyHostLocality` (`routeGuard.ts:145-150`): `null` → `"remote"` (fail closed), loopback set → `"loopback"`, `PRIVATE_LAN_PATTERNS` (`:157-164`) → `"lan"`, else `"remote"`.

`PRIVATE_LAN_PATTERNS` (`:157-164`) is worth noting for its completeness:

```
10.0.0.0/8
100.64.0.0/10   ← CGNAT
192.168.0.0/16
172.16.0.0/12
fc00::/7        ← IPv6 ULA
fe80::/10       ← IPv6 link-local
```

Including CGNAT and IPv6 ULA/link-local shows the classification was written against real deployment topologies, not just RFC1918.

`isLoopbackHost` / `isPrivateLanHost` (`:122-137`, `:172-184`) strip IPv6 brackets, strip `:port` only when there is exactly one colon, and strip `::ffff:`.

---

## 6. WebSocket Sidecar Transport

`src/server/ws/liveServer.ts` (628 lines) — separate process, `127.0.0.1:20132` by default.

| Control | Value | Line |
|---|---|---|
| `DEFAULT_PORT` / `DEFAULT_HOST` | `20132` / `127.0.0.1` | `:40,:44` |
| Heartbeat / timeout | 15,000 / 35,000 ms | `:45-46` |
| Max clients | 500 | `:47` |
| Rate limit | 100 msg/s per client, fixed 1 s window | `:48,206-216` |
| Early queue | 32 frames / 16,384 bytes | `:49-50` |
| Backlog | 500 entries | `:85` |
| Internal ingest cap | 1,000,000 (UTF-16 units) | `:330` |

**Gate order** (`:477-561`): message listener attached first (queueing early frames) → origin (`4003`) → capacity (`1013`) → auth (`4001`) → register → replay queue.

**Origin policy** (`liveServerAllowList.ts:91-106`): defaults are the three loopback dashboard origins on port 20128 (`:19-23`), extended by `LIVE_WS_ALLOWED_ORIGINS`. Critically:

```
if (origin === undefined) {
  allow ONLY if LIVE_WS_HOST resolves to 127.0.0.1 | ::1 | localhost
}
```

Without this, a non-browser client would simply omit `Origin` and skip the browser check entirely. The rule is stated in the module's own header comment (`:1-11`) as MUST-match-the-handler-exactly, and there is a unit test (`tests/unit/security/live-server-allowlist.test.ts`) that exercises the module without booting the server.

`LIVE_WS_ALLOWED_HOSTS` (`:49-51`) is the LAN/Tailscale opt-in, **empty by default**. `originHostMatches` (`:72-77`) returns `false` when the host list is empty, so LAN access requires explicit configuration.

`allowedOrigins.has(origin)` (`:103`) is an **exact string compare** — no case normalisation, no trailing-slash handling, unlike the CORS module's `normalizeOrigin` (`:52-58`). Stricter, not looser, so it fails safe.

**Auth** (`:121-191`):
- `Authorization: Bearer` or `X-Live-WS-Token` — **header only**.
- Query-string tokens explicitly rejected (`:124-126`): *"they leak into access logs/history/Referer"*.
- Cookie fallback: `auth_token` + `JWT_SECRET`, `jwtVerify` (`:179-191`).
- `getCookieValueFromHeader` (`:163-177`) escapes regex metachars and uses `(?:^|;\s*)` so a non-first cookie still matches — the `\\s`-vs-`\s` fix for issue #4004.

**The internal ingest endpoint** (`:311-359`): `POST /__omniroute_event`, loopback-only via `isLoopbackRequest` (`:311-314`, matching only `127.0.0.1`, `::1`, `::ffff:127.0.0.1` from the raw socket — no `x-forwarded-for` trust), **no token required**, body capped, event name validated against `CHANNEL_EVENTS` (`:341-346`).

The absence of a token is a deliberate sidecar-internal design: the parent posts to the child over loopback. The loopback restriction is the only control, and it is sound because the raw socket address is used.

**Finding S-1 (High): no send backpressure.** `sendTo` (`:263-267`) checks only `readyState`. See `06-performance-scalability.md` §5.7.

**Finding S-3 (High): client/server auth mismatch.** `useLiveDashboard.ts:174-178` appends the API key as `?token=`, which the server deliberately ignores. Cookie auth is the only working path today; a token-only headless client is broken.

**Finding S-4 (Medium): idle listeners are terminated.** `lastActivity` updates only inside `handleMessage` (`:226`); the server's own `pong` at `:432` does not refresh it. A client that subscribes then only listens is terminated at 35 s.

**Startup:** module-level side-effect auto-start (`:599-628`) gated by `isBuildOrTest()` (`:610-614`) and `isLiveWsEnabled()` (`:616-620`, `OMNIROUTE_ENABLE_LIVE_WS`, default **on**). `src/instrumentation-node.ts:657-669` triggers it in production, non-fatally. `scripts/start-ws-server.mjs:55-89` respawns the child with `OMNIROUTE_ENABLE_LIVE_WS="0"` to suppress the double-start.

**Bind-failure handling** (`:579-596`) is defensive: `wss.once("error")` unsubscribes and rejects, because `ws` re-emits the server `error` on `wss` and **throws synchronously** if `wss` has no listener (issue #6324 crash-loop).

---

## 7. MITM / Interception Transport

### 7.1 Modes

| Mode | Bind | TLS | Capture |
|---|---|---|---|
| **DNS-spoof** (primary) | `0.0.0.0:443` (default) | terminated locally | full body + headers |
| **CONNECT** | on-demand | for `target` verdicts, handed to the local TLS layer | full for target; **none** for bypass/passthrough |
| **HTTP proxy** | `127.0.0.1:8080` | — | CONNECT is a labelled raw tunnel |
| **TPROXY** | `0.0.0.0:<onPort>` | via `DynamicCertStore` | full, requires `CAP_NET_ADMIN` |

`server.cjs:707-729` documents this clearly: the primary flow is DNS-spoof and **CONNECT fires only for explicit HTTPS-proxy clients**; system-wide proxy mode uses `httpProxyServer.ts` on its own port.

`server.cjs:791-806` states the scope limit honestly: true on-wire bypass-without-decrypt at :443 under direct TLS would need SNI sniffing on the raw `connection` event, and is intentionally out of scope.

### 7.2 CONNECT handling

`server.cjs:807-836` — `routeBypass(connectHost)` returns:

| Verdict | Behaviour | Capture |
|---|---|---|
| `bypass` | raw TCP tunnel | **none** — no decryption, no body/header logging |
| `target` | write `200 Connection Established`, `clientSocket.unshift(head)`, re-emit `connection` into the local TLS layer | full |
| `passthrough` | raw TCP tunnel | none |

`rawTcpForward` (`:743-789`) — `net.connect(port, host)`, writes `HTTP/1.1 200 Connection Established`, pipes both directions, `setTimeout(MITM_IDLE_TIMEOUT_MS, destroyBoth)` on both sockets, cross-destroys on `error`/`close`.

Bypass connections are tunneled without decryption — so a banking or SSO session is never readable by the inspector, matching the default bypass patterns in §8.

### 7.3 Passthrough

`server.cjs:317-333`:
- `getTargetHost` — the `Host` header if in `TARGET_HOSTS`, else the hard-coded fallback `daily-cloudcode-pa.sandbox.googleapis.com`.
- `resolveTargetIP` — a **hard-coded `8.8.8.8`** `dns.Resolver` (bypasses `/etc/hosts` by design, since the whole mechanism relies on hosts-file spoofing), results cached in `cachedTargetIPs` **for process lifetime**.

`server.cjs:421-428` — `isSelfLoopDestination(targetIP, 443, LOCAL_PORT)` returns HTTP **508 "Loop Detected"** rather than forwarding into itself.

`server.cjs:432` — `rejectUnauthorized = process.env.MITM_DISABLE_TLS_VERIFY !== "1"`, i.e. **on by default**.

`server.cjs:434-448` — `https.request({hostname: targetIP, port: 443, servername: targetHost, rejectUnauthorized, headers: {...req.headers, host: targetHost}})`, then `writeHead(status, forwardRes.headers)` + `pipe(res)`. The **only** header rewrite is `Host`; `Authorization` and `Cookie` pass through verbatim, which is correct for a transparent proxy.

### 7.4 Interception path

`server.cjs:626-704` — a single catch-all HTTPS handler, no route table:

```
bump stats → collectBodyRaw → derive host from Host header → extractModel → saveRequestLog
  x-omniroute-source === "omniroute"           → passthrough   (OmniRoute's own loop)
  host not in TARGET_HOSTS                     → passthrough
  URL doesn't match routeConfig.chatUrlPatterns → passthrough
  push inspector entry with status "in-flight"  ◄── pre-mapping capture (#8656)
  no mapped override for the model              → passthrough
  else → intercept()
```

The pre-mapping capture at `:663-685` is a good design decision: passthrough traffic is still *visible* in the inspector, so an operator can see what the bridge is doing even when it decides not to act.

`intercept()` (`:499-611`):
- `agentId` from the `Host` header via `TARGET_HOST_AGENT`, fallback `"unknown"`.
- `aliasConfigShim.applyAntigravityOverride` (`:517-520`) — swaps `model`, sets `reasoningEffortOverride`.
- `standaloneRoutingShim.resolveForwardTargetForAgent` (`:528-535`) — cloudcode envelope → `/v1/antigravity`; `routerPath === "/v1/messages"` → `<base>/v1/messages`; else fallback.
- `fetch` (`:538-547`) with `Authorization: Bearer <ROUTER_API_KEY>`, `x-omniroute-source: agent-bridge`, `x-omniroute-agent: <agentId>`. **No timeout, no `AbortSignal`.**
- Response headers (`:559-564`): `text/event-stream`, `no-cache`, `keep-alive`, `X-Accel-Buffering: no` — and status **always 200**.
- SSE streaming (`:566-579`): `getReader()` loop, `TextDecoder` streaming decode, inspector buffer capped at `INGEST_MAX_BODY`, `res.write` per chunk, `res.end` on done.
- `finally` always calls `captureToInspector` with `proxyLatencyMs` / `upstreamLatencyMs` (`:594-611`), so proxy overhead is separable from provider latency.

### 7.5 Server timeouts

`server.cjs:840-842`:

```js
requestTimeout  = MITM_IDLE_TIMEOUT_MS * 5   // 300 s at the 60 s default
headersTimeout  = MITM_IDLE_TIMEOUT_MS
keepAliveTimeout = MITM_IDLE_TIMEOUT_MS
```

`server.cjs:850-863` — per-connection `socket.setTimeout(MITM_IDLE_TIMEOUT_MS, () => socket.destroy())`, with a `socket.__mitmCounted` guard because a CONNECT "target" re-emits an already-counted socket.

`server.cjs:865-874` — `EADDRINUSE` → `❌ Port N already in use`, `EACCES` → `❌ Permission denied for port N`, else `❌ <message>`, then `exit(1)`. The parent maps these via `interpretMitmStartupError` (`manager.ts:45-71`), which explicitly refuses to guess port 443 when no diagnostic was captured (`:70`).

### 7.6 System proxy configuration

`src/mitm/inspector/systemProxyConfig.ts` (294 lines):

| Platform | Mechanism |
|---|---|
| macOS | `networksetup -getwebproxy` / `-setwebproxy <service> 127.0.0.1 <port>`; revert restores prior values or sets `*-state off` (`:133-166`) |
| Linux (GNOME) | `gsettings get/set org.gnome.system.proxy[.http\|.https]` for `mode\|host\|port`; revert restores `mode` (default `'none'`) and stored host/port (`:172-229`) |
| Windows | `netsh winhttp show proxy` / `set proxy 127.0.0.1:<port>`; revert is unconditional `netsh winhttp reset proxy` (`:235-253`) |

`MAC_DEFAULT_SERVICE = "Wi-Fi"` (`:103`) is the only macOS service ever touched — correct default, but it will miss Ethernet and VPN service names. All calls use argv arrays with a swappable `execImpl` (`__setExec`).

### 7.7 TPROXY

`src/mitm/tproxy/` — 9 TS files + 4 native files.

`commands.ts:79-121` — rule specs:

```
ip rule add fwmark <mark> lookup <table>
ip route add local 0.0.0.0/0 dev lo table <table>
iptables -A mangle OUTPUT -j MARK --set-mark <mark>          [-m mark ! --mark <bypass>]
iptables -A mangle PREROUTING -j TPROXY --on-port <onPort> --tproxy-mark <mark>
```

Revert is the exact inverse in reverse order (`setup.ts:60-68`, each command individually try/catch-swallowed for idempotency). `applyTproxy` (`:40-52`) reverts fully on any failure before rethrowing.

`validateTproxyConfig` (`:62-76`) bounds ports 1–65535 and requires `bypassMark !== mark`.

**iptables only — no nftables path exists anywhere in the tree.** Relevant for distributions shipping nftables-only.

`native/transparent.c`:
- `socket(AF_INET, SOCK_STREAM)` → `SO_REUSEADDR` → **`setsockopt(SOL_IP, IP_TRANSPARENT)`** (requires `CAP_NET_ADMIN`) → `bind` → `listen(fd, 511)`.
- `SetSocketMark` = `SO_MARK`.
- `ConnectMarked` — socket → **`SO_MARK` before `connect`** → `O_NONBLOCK` → `connect()` tolerating `EINPROGRESS` → return fd.
- **IPv4 only.**

`.gitignore` excludes `build/` and `prebuilds/`, so no binary is committed. `native/README.md:23-47` documents `npm run build:native:tproxy`; `scripts/build/build-tproxy-native.mjs` runs `node-gyp rebuild` during `npm run build`; `assembleStandalone.mjs` copies the `.node` into the standalone bundle. **linux-x64 only.**

`transparentSocket.ts:33-36,59-131` — addon candidates `native/build/Release/transparent.node` then `native/prebuilds/transparent.node`, resolved both module-relative and `<cwd>/src/mitm/tproxy/…`; returns `null` off Linux; the three wrappers throw an actionable message when missing.

`dynamicCert.ts` — CA 10-year, `cA:true critical`, `keyCertSign|cRLSign critical`; leaves 1-year; `DynamicCertStore` per-host `SecureContext` map with a `SNICallback`.

`caTrust.ts` — dedicated trust-store slot `omniroute-tproxy-ca.crt` (distinct from the MITM CA `omniroute-mitm.crt`); stages the PEM to `os.tmpdir()` at mode **0644**, then `sudo -S mkdir -p` / `cp` / update command, unlinking in `finally`. Honours `OMNIROUTE_SKIP_SYSTEM_TRUST === "1"` when `deps.run` was not injected. Linux only.

---

## 8. Bypass Safety Net

`_internal/bypass.cjs:27-32` defaults (duplicated from `passthrough.ts`):

```
/\.bank\./i
/(^|\.)gov(\.|$)/i
/(^|\.)okta\.com$/i
/(^|\.)auth0\.com$/i
```

Precedence (`:82-95`): defaults → user globs → target hosts.

These exempt banking, government, and SSO hosts. That is not a convenience — those flows use certificate pinning and step-up auth that break under interception, and a captured banking session sitting in the inspector buffer would be a serious privacy incident. The defaults are the right ones and are tested (`mitm-passthrough.test.ts`, 10 cases).

`bypassGlobMatch` (`:42-64`) rejects patterns with more than 9 segments — bounding glob cost against a hostile pattern list.

`parseBypassJson` (`:109-120`) is pure: lowercased, non-empty strings, `[]` on any failure.

`parseVerboseLevel` (`:157-160`) — `parseInt`, integer ≥ 0, else defaults to 1.

---

## 9. Networking Findings

| ID | Sev | Finding | Section | Fix |
|---|---|---|---|---|
| **S-1** | **High** | WS `sendTo` has no `bufferedAmount` check — unbounded server memory behind a slow client. | §6 | Guard on `bufferedAmount`; terminate past a threshold with a distinct code. |
| **S-3** | **High** | Client sends the WS token in the query string; the server reads headers only, by design. | §6 | Use `X-Live-WS-Token`, or document cookie-only. |
| **S-4** | Medium | Idle listeners terminated: `lastActivity` is not refreshed by server `pong`. | §6 | Refresh on send, or separate liveness from activity. |
| **N-1** | Medium | MITM upstream `fetch` has no timeout or `AbortSignal`. | §7.4 | `AbortSignal.timeout()`. |
| **N-2** | Medium | MITM DNS hard-coded to `8.8.8.8`, results cached for process lifetime. | §7.3 | Configurable resolver; add a TTL. |
| **N-3** | Medium | MITM reports HTTP 200 for every SSE response regardless of upstream status. | §7.4 | Propagate a meaningful status for non-streaming failures. |
| **N-4** | Low | macOS system proxy only ever touches service `Wi-Fi`. | §7.6 | Enumerate services, or make it configurable. |
| **N-5** | Low | WS origin matching is an exact string compare with no normalisation, unlike the CORS module. | §6 | Fails safe; note the asymmetry. |
| **N-6** | Low | Internal ingest cap is UTF-16 code units, not bytes. | §6 | `Buffer.byteLength`. |
| **N-7** | Info | tproxy is iptables-only; no nftables. Native addon is linux-x64 only. | §7.7 | Document. |
| **N-8** | Info | Emergency fallback defaults to **enabled** (opt-out via `OMNIROUTE_EMERGENCY_FALLBACK`). | §3.2 | Document; it caps `max_tokens` at 4096 on 402. |

---

## 10. Recommendations

**Immediate**
1. `ws.bufferedAmount` guard (S-1) — the only unbounded-memory path in the network layer.
2. Fix the WS client to use `X-Live-WS-Token` (S-3) — 2 lines.
3. `AbortSignal.timeout()` on the MITM upstream fetch (N-1).
4. Refresh `lastActivity` on send, or track liveness separately (S-4).

**Short term**
5. Configurable DNS resolver with a TTL for the MITM passthrough path (N-2).
6. Propagate upstream status for non-streaming MITM failures (N-3).
7. Document the trust-proxy matrix: `OMNIROUTE_TRUST_PROXY` values, the peer-stamp prerequisite, and what LOCAL_ONLY does behind a tunnel.
8. Document that `CORS_ALLOW_ALL` is a genuine `*` (unsafe with credentials) and that the default allow-list is empty.

**Medium term**
9. Decide whether to support nftables in tproxy, or document the iptables-only requirement in the system-proxy setup guide (N-7).
10. Add an outbound-egress allowlist (mirroring the dated `config/alibaba-free-tier-allowlist.json` pattern) for provider hostnames, so a compromised DNS response cannot redirect provider traffic.
11. Add a `netstat`-level dashboard panel showing the six surfaces and their listeners, so operators can verify the deployed topology.
12. Make the macOS system-proxy service configurable (N-4).
