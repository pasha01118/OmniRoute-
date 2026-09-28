# 05 — Security & Compliance Review

> **Verification basis.** OmniRoute `5764027`, version `3.8.50`. Every claim cites `path:line` against that tree. Counts and thresholds were measured, not copied from documentation.

---

## 1. Security Model in One Page

OmniRoute exposes three trust zones and enforces them with a single middleware plus a small number of hand-written policy modules.

```
                    UNTRUSTED
                        │
        internet clients, third-party coding agents
                        │  (no Origin, forged headers, tunneled JWTs)
                        ▼
        ┌───────────────────────────────────────┐
        │  src/proxy.ts  config.matcher         │  only listed prefixes enter
        │  → runAuthzPipeline                  │
        ├───────────────────────────────────────┤
        │  1 strip every spoofable auth header  │  ◄── kills Host/peer spoofing
        │  2 classify route → PUBLIC|CAPI|MGMT │
        │  3 resolve TRUE peer from stamped IP │
        │  4 IP filter, body size, drain        │
        │  5 policy.evaluate()                 │
        │  6 CSRF / origin gate for mutations   │
        └───────────────────────────────────────┘
                        │  loopback only
        ┌───────────────▼───────────────────────┐
        │  LOCAL ONLY (26 prefixes + 2 regex)   │  spawn-capable surfaces
        │  MCP · cli-tools · services · plugins │
        │  middleware vm · db-backups · system   │
        └───────────────────────────────────────┘
                        │
        ┌───────────────▼───────────────────────┐
        │  MITM sidecar (privileged)            │  DNS + /etc/hosts + CA
        │  manager.ts → server.cjs, port 443    │  sudo, /etc/hosts, trust store
        └───────────────────────────────────────┘
```

**The central design decision** is that locality is never inferred from a request header. `src/server/authz/headers.ts:27-38` defines `PEER_IP_HEADER` as `<token>|<ip>`, stamped by `scripts/dev/peer-stamp.mjs` before Next starts, and `peerStamp.ts:18-35` validates it with a `timingSafeEqual` on the token. A client that simply sets `X-omniroute-Forwarded-For: 127.0.0.1` or a `Host: localhost` header gets nothing, because neither is consulted.

---

## 2. Authentication

### 2.1 Management password

| Property | Value | Evidence |
|---|---|---|
| Hash | bcrypt | `src/lib/auth/managementPassword.ts:4` |
| Cost factor | 12 | `managementPassword.ts:5` |
| Validation | `/^\$2[aby]\$\d{2}\$[./A-Za-z0-9]{53}$/` | `:4` |
| Non-bcrypt value | returns `false` (fails closed) | `:52-55` |
| Default-password detection | warns loudly if password is `CHANGEME` | `:9,76-83` |
| First-run migration | forces `setupComplete: true` and `requireLogin: true` when no password was stored | `:97-102` |

### 2.2 Dashboard session

- HS256 JWT in an `auth_token` cookie, verified with `jose.jwtVerify` against `JWT_SECRET`.
- Cookie flags: `httpOnly`, `path=/`, `sameSite=lax`, `secure` derived from `AUTH_COOKIE_SECURE==="true"` **or** `x-forwarded-proto: https` **or** `nextUrl.protocol === "https:"` (`pipeline.ts:136-140`).
- Auto-refresh: a token expiring within 7 days is reissued with a 30-day expiry (`pipeline.ts:159,162-172`). A stale/invalid JWT **deletes the cookie** and emits a single warning (`:174-181`).
- **Gap:** the session is refreshed purely on expiry proximity. There is no `aud`/`iss`/role claim check beyond signature and expiry, and no session revocation list — logout is client-side cookie deletion.

### 2.3 Access tokens (`oma_` prefix)

`src/server/authz/accessTokenAuth.ts`:
- Prefix `oma_` distinguishes CLI access tokens from inference API keys (`:18`).
- Bearer extraction is **header-only** (`:28-32`) — never the URL.
- Verdicts: `absent | error | invalid | insufficient | ok` (`:20-25`).
- Shared by the policy and route-level `requireManagementAuth` so the two gates cannot drift (`:5-15`) — a genuinely good design.

### 2.4 API keys

| Property | Value | Evidence |
|---|---|---|
| Reveal | gated on `ALLOW_API_KEY_REVEAL`; mask = first 8 + `****` + last 4 | `src/lib/apiKeyExposure.ts:5-19` |
| Model policy | `enforceApiKeyPolicy` runs before handler dispatch | `src/app/api/v1/embeddings/route.ts:57-65` |
| Management routes | header-only extraction, `{allowUrl:false}` | `requireManagementAuth.ts:88` (#3300) |
| Policy subject id | `key_<last4>` — never the raw key | `policies/clientApi.ts:52-55` |

### 2.5 Login brute-force guard

`src/server/auth/loginGuard.ts`:

| Property | Value | Line |
|---|---|---|
| Window | 15 min | `:18` |
| Lockout | 15 min | `:19` |
| Threshold | 5 failures | `:20` |
| State | module-global `Map`, **in-process only** | `:22-28` |
| Prune threshold | 256 entries | `:35` |
| Key | trimmed IP, or the literal `"__unknown__"` | `:54-57` |

**Findings.** (a) No username dimension — a distributed attacker rotates IPs freely. (b) All clients with an unparseable IP share one counter, so a proxy pool can lock out legitimate users. (c) State is per-process, so it is ineffective behind multiple instances. The file's own header calls this "defense-in-depth … not a substitute for Cloudflare/reverse-proxy rate limiting", which is the right framing.

---

## 3. Authorization

### 3.1 Route classification

`src/server/authz/classify.ts:50-126` — every path is normalized (leading slash, one trailing slash stripped, `/codex*`→`/api/v1/responses`, `/v1beta*`→`/api/v1beta*`) and assigned exactly one class.

**Fail-closed default:** any path that matches no rule becomes `MANAGEMENT` (`:121-125`). A newly added route that nobody classified is locked, not open. This is the single most important authorization property in the codebase.

### 3.2 Header stripping — the load-bearing step

`pipeline.ts:306-314` deletes all six `AUTHZ_TRUSTED_HEADERS` (`headers.ts:69-76`) plus `PEER_IP_HEADER` and `VIA_PROXY_HEADER` from the forwarded request **before** any handler runs. Those are:

```
x-omniroute-route-class
x-omniroute-auth-kind
x-omniroute-auth-id
x-omniroute-auth-label
x-omniroute-auth-scopes
x-omniroute-peer-locality
```

Without this step, any handler calling `assertAuth()` would read client-controlled values.

### 3.3 The via-proxy downgrade

`peerStamp.ts:89-99` — `classifyStampedPeerLocality` resolves the stamped peer IP and classifies it, but **forces `"remote"` when the via-proxy marker is present and the locality is not already remote**. `headers.ts:40-54` explains the motivation: an external tunnel appears to Next as a loopback socket, which would otherwise satisfy a LOCAL_ONLY route. This closes a real vulnerability class (referenced upstream as da667836).

### 3.4 Spawn-capable route containment

`routeGuard.ts:33-94` enumerates 26 `LOCAL_ONLY_API_PREFIXES`, each with a comment naming its child-process / `vm.Script` / `tar` / `git checkout` surface:

```
/api/mcp/                        /api/cli-tools/runtime/
/api/services/                   /api/copilot/
/api/tools/agent-bridge/         /api/tools/traffic-inspector/
/api/issue-agent/                /api/plugins/  and /api/plugins
/api/middleware/                 (arbitrary JS via new vm.Script)
/api/system/version              (git checkout + npm install)
/api/db-backups/exportAll        (spawns tar)
/api/local/  /api/headroom/*     /api/jobs
/api/oauth/cursor/auto-import    /api/skills/collect/
/api/discovery/                  VNC_ROUTE_PREFIX
/api/acp/agents                  /api/resilience/connections
/dashboard/resilience/connections
/dashboard/providers/services/   /api/providers/cursor/agent-availability
```

Two regex patterns cover dynamic segments (`:68-94`): `refresh-cursor` and `chatgpt-web-codex-doctor`.

**Three properties keep this list correct as it grows:**

1. The `GET` exemption (`/api/system/version`) applies only on an **exact** match — a new `/api/system/version/…` sub-path is immediately re-protected (`:217-228`).
2. `matchesPrefix` requires a real segment boundary, so `/api/auth` cannot accidentally match `/api/authz-inventory` (`accessScopes.ts:39-46`).
3. `isLocalOnlyBypassableByManageScope` (`:242-267`) rejects a bypass when the candidate prefix **equals, is a child of, or is a parent of** any spawn-capable prefix — a bidirectional containment check, so both over-broad and over-narrow edits fail safe.

### 3.5 LOCAL_ONLY enforcement order

`policies/management.ts:152-224`:

1. Stamped peer IP preferred; `request.ip`/`socket.remoteAddress` only for non-middleware callers such as tests (`:28-41`).
2. Both `isLoopbackRequest` and `isPrivateLanRequest` return `false` immediately when a proxy hop is stamped (`:58-73`).
3. Non-local request → attempt the manage-scope bypass with **header-only** key extraction.
4. `/api/mcp/` accepts `mcp:connect`; every other prefix requires `manage` (`:170-174`).
5. Audit records which privilege granted it — `admin` / `manage` / `mcp-connect` (`:178-188`).
6. Any backend throw → log + `503 AUTH_BACKEND_UNAVAILABLE`. **Never a silent allow.**
7. No valid bypass → `403 LOCAL_ONLY`.

### 3.6 Scope inference

`accessScopes.ts`:

| Category | Prefixes | Effect |
|---|---|---|
| `ADMIN_SCOPE_PREFIXES` | `/api/cli/tokens`, `/api/oauth`, `/api/auth`, `/api/policy`, `/api/services`, `/api/mcp` | `admin` for all methods |
| `ADMIN_MUTATION_PREFIXES` | `/api/providers`, `/api/cli-tools/apply` | `admin` only when mutating |
| Everything else | — | `write` for mutations, `read` for reads |

The default for a new mutating route is `write` (`:14-16`) — a sensible fail-safe.

### 3.7 CSRF and origin

`pipeline.ts:395-409` applies the CSRF/origin gate **only** for `MANAGEMENT` + `dashboard_session` + unsafe method. This is correctly scoped: a bearer-token API client has no ambient authority to abuse.

`csrf.ts`:
- Format `v1.<exp>.<mac>`, TTL 600 s, MAC = HMAC-SHA256 over `context \n exp \n sha256(auth_token cookie)`.
- Validation: version, exact field count, safe integer, not expired, length check + `timingSafeEqual`.
- **Property to document:** no one-time-use tracking, so a captured token is replayable within its 10-minute window. Acceptable for CSRF (the attacker still needs the cookie for the request to be meaningful), but it is a fact, not a non-issue.

`validateBrowserMutationOrigin` (`publicOrigin.ts:252-270`):
- `sec-fetch-site` present and not in `same-origin|same-site|none` → reject.
- **No `Origin` header at all → allow.** Correct: non-browser clients (curl, CLI) legitimately omit it, and a browser always sends `Origin` on cross-origin mutations.
- Otherwise the normalized `Origin` must be in the resolved candidate set.

### 3.8 A2A has no in-handler authorization — finding S-2

Every MCP route calls `requireManagementAuth` explicitly:

```
src/app/api/mcp/sse/route.ts:32,42
src/app/api/mcp/stream/route.ts:44,50,56
```

The A2A task routes (`src/app/api/a2a/tasks/route.ts`, `[id]/route.ts`, `[id]/cancel/route.ts`, `status/route.ts`) call it **nowhere**. They depend entirely on the global pipeline classifying them as MANAGEMENT.

Today that is safe. It is also one `config.matcher` edit, one route rename, or one future `publicApiRoutes` entry away from being exposed. The MCP pattern should be applied to A2A as well.

---

## 4. Cryptography

### 4.1 Field encryption

`src/lib/db/encryption.ts`:

| Property | Value | Line |
|---|---|---|
| Algorithm | AES-256-GCM | `:29` |
| IV | 16 bytes | `:30` |
| Key | 32 bytes | `:31` |
| **Auth tag** | **16 bytes, explicitly pinned** | `:40` |
| Format | `enc:v1:<iv_hex>:<ct_hex>:<tag_hex>` | — |
| Encrypted fields | `apiKey`, `accessToken`, `refreshToken`, `idToken` | `:49-54` |

Pinning `AUTH_TAG_LENGTH` closes the GCM tag-truncation forgery vector (Semgrep `gcm-no-tag-length`). Legacy dynamic-salt keys auto-migrate on decrypt.

**Finding:** if `STORAGE_ENCRYPTION_KEY` is unset, the module runs in **plaintext passthrough** mode, documented in the header (`:8-10`) but silent at runtime. This should emit a boot-time warning equivalent to the `CHANGEME` password warning.

### 4.2 CSRF MAC

HMAC-SHA256, keyed by `JWT_SECRET`. Key separation from the session JWT itself is achieved by the `context` string and the session hash inside the MAC input. Sound.

### 4.3 Peer/IP token comparison

`timingSafeEqual` in three places: `peerStamp.ts` token match, `csrf.ts` MAC compare, `management.ts` Codex bridge secret compare. Consistent.

### 4.4 CA material

`src/mitm/cert/rootCa.ts:45-73`:
- `ca.key` / `ca.crt` under `<dataDir>/mitm`.
- Generated on first use, then `fs.chmodSync(keyPath, 0o600)`.
- **No expiry check on an existing CA** — a multi-year-old CA is reused indefinitely.
- The legacy path (`generate.ts`) produces a 1-year self-signed leaf with **no chmod on the key**.

`tproxy/dynamicCert.ts:35-49` produces a **10-year** CA with `cA:true critical` and `keyCertSign|cRLSign critical`, and `issueLeafCert` produces 1-year leaves. Separate trust-store slot (`omniroute-tproxy-ca.crt`) from the MITM CA (`omniroute-mitm.crt`), which is correct hygiene.

---

## 5. Input Validation

### 5.1 Two-layer schema discipline

- **Boundary Zod schemas** reject malformed input: `cliMitmStartSchema`, `cliMitmStopSchema`, `cliMitmAliasUpdateSchema` (`src/shared/validation/schemas/cli.ts:17-41`), `updateMitmSchema`, `regenerateSchema` (`src/app/api/settings/mitm/route.ts:41-51`).
- **Canonical-value checks at the route**: `hasInvalidReasoningEffort` rejects unknown effort strings at the edge (`alias/route.ts:63-65`) even though the schema is deliberately permissive (`cli.ts:27-36`).
- **The request-pipeline schema is intentionally permissive** — `chatCompletionsRouteShapeSchema.safeParse()` only asserts `model` is a nullable string and `messages` is an array, with `.passthrough()` (`route.ts:63-70`). This is correct: strict validation of a 20-provider-shaped request body would break provider compatibility. Malformed-but-shaped input is handled downstream.

### 5.2 Prompt injection guard

`src/middleware/promptInjectionGuard.ts` is invoked in the `/v1/chat/completions` pipeline before dispatch (`route.ts:155-176`); a hit returns `400 SECURITY_001`. Backed by a dedicated CI check and nightly `promptfoo` runs.

### 5.3 No shell interpolation

The MITM code has a documented "Hard Rule #13" — no shell string interpolation. It is honoured throughout:

- `dns/dnsConfig.ts:183-193` passes the hosts file and hostname as `process.argv[1]/[2]` to `node -e <script>`.
- `cert/install.ts:42-103` passes `CERT_PATH` and `ACTION` through `env`, not interpolation, into the NSS bash script.
- `systemCommands.ts:120-187` uses `spawn` (never `exec`) with an argv array; the header comment documents the fixed allowlist (`sudo, certutil, security, update-ca-certificates, update-ca-trust, cp, mkdir, rm`).
- Windows elevation uses `Start-Process … -File <path>` with `-NoProfile -NonInteractive -ExecutionPolicy Bypass`, explicitly **not** `-EncodedCommand` (`systemCommands.ts:193-224`).
- PowerShell quoting doubles single quotes (`:189-191`).
- The TPROXY config is validated (ports 1–65535, `bypassMark !== mark`) before any command is built (`tproxy/commands.ts:62-76`).

### 5.4 Sudo-password handling

`systemCommands.ts:120-187` — the password is written to the child's stdin and `stdin.end()` is called immediately; it is never placed in argv (where it would be visible in `ps`). The password is cached in a module-level variable (`manager.ts:126-135`) and cleared on stop. `useMitmSudoPrompt.tsx` clears its input on confirm and on close. No disk persistence of the sudo password was found.

**Gap:** the cached password lives in process memory for the lifetime of the process. That is unavoidable for this feature, but it should be in the threat model documentation.

---

## 6. Secret Redaction

### 6.1 Three independent layers

| Layer | Location | Mechanism |
|---|---|---|
| Log-payload redaction | `src/lib/logPayloads.ts:3-18` | 18 `SENSITIVE_KEYS` (incl. `x-goog-api-key`) → omit, then `sanitizePII` → `redactPayload` |
| Credential masker | `src/lib/guardrails/credentialMasker.ts:20-49` | 20+ provider-specific patterns (`sk-proj-*`, `sk-ant-*`, `AIza*`, `hf_*`, `gh[pousr]_*`, `xox[bpoa]-*`, `lin_api_*`, Discord bot tokens, …). **Opt-in**: `CREDENTIAL_REDACTION_ENABLED=true` |
| PII sanitizer | `src/lib/piiSanitizer.ts:44-104` | 13 classes → `[EMAIL_REDACTED]`, `[SSN_REDACTED]`, `[CC_REDACTED]`, `[PHONE_REDACTED]`, `[CPF_REDACTED]`, `[CNPJ_REDACTED]`, `[IP_REDACTED]`, `[AWS_KEY_REDACTED]`, `[API_KEY_REDACTED]` |
| MITM mask | `src/mitm/maskSecrets.ts` | Reused by `src/lib/inspector/secretMask.ts` |

**PII redaction defaults to OFF** — `PII_REDACTION_ENABLED` and `PII_RESPONSE_SANITIZATION` both `defaultValue: "false"`, enforced by `AGENTS.md` Hard Rule #20 across three application points (`guardrails/piiMasker.ts`, `piiSanitizer.ts`, `streamingPiiTransform.ts`). This is a deliberate, documented choice; it must be surfaced in any compliance attestation.

### 6.2 Storage bounds

- `serializePayloadForStorage(payload, maxLength = 65536)` truncates to 64 KiB with a `_truncated` / `_originalSize` / `_preview` envelope (`logPayloads.ts:153-166`).
- `omitEncryptedReasoningFromLogChunks()` strips `encrypted_content` from captured SSE text with a linear (non-backtracking) regex — no ReDoS surface (`:31-41`).
- MITM inspector bodies capped by `INSPECTOR_MAX_BODY_KB` (default **1024**) with an explicit `…(truncated for performance)` marker (`inspector/buffer.ts:20,29-38`).
- MITM `server.cjs` caps captured bodies at `INGEST_MAX_BODY = 65536` and sanitized error messages at 4096 chars / first line only, replacing path-like tokens with `<path>` (`:100,115-125,151`).

### 6.3 What is never logged

**MCP tool audit stores the hash, not the input** — `open-sse/mcp-server/audit.ts:359-390` writes `input_hash` (SHA-256), `output_summary` (first 200 chars), `duration_ms`, `api_key_id`, `success`, `error_code`. The contract is stated explicitly in `schemas/audit.ts:1-9`: "Input data is never stored in clear text." Audit-write failures are caught and never propagated — "Never let audit failure break tool execution" — which is a defensible availability-over-completeness trade.

Call logs are **summary-only** in SQLite; full request/response bodies live outside the DB as filesystem artifacts with `artifact_relpath` / `artifact_size_bytes` / `artifact_sha256` (`src/lib/usage/callLogs.ts:55-89`).

### 6.4 Per-key no-log

`src/lib/compliance/noLog.ts:16-52` — `NO_LOG_API_KEY_IDS` env plus a DB `no_log` column, probed with `PRAGMA table_info` for capability, cached 30 s. Consumed by `callLogs.ts:25`. A genuine per-tenant opt-out that most projects lack.

---

## 7. Network & Transport Security

### 7.1 CORS

`src/server/cors/origins.ts`:

| Property | Value | Line |
|---|---|---|
| Default | **empty allow-list, no wildcard** | `:27-39` |
| `STANDARD_ALLOW_HEADERS` | `Content-Type, Authorization, x-api-key, anthropic-version, x-omniroute-connection, x-internal-test, accept` | `:23-25` |
| `STANDARD_ALLOW_METHODS` | `GET, POST, PUT, DELETE, PATCH, OPTIONS` | `:25` |
| `Vary` | `Origin` appended whenever the ACAO header is set; `Accept-Encoding` when relaxed and status ≠ 204 | `:157-170` |
| Header echo | `Access-Control-Allow-Headers` is **overwritten** with the request's `Access-Control-Request-Headers` when present | `:173-176` |

**`relaxForTokenAuth`** (`:121-146`) is the only path that echoes the caller's origin when the explicit allow-list does not match. It is reserved for `/v1/*`, `/v1beta/*`, and read-only public endpoints, is documented as never pairing with `Access-Control-Allow-Credentials`, and the pipeline only enables it for `CLIENT_API` and read-only `PUBLIC` classes (`pipeline.ts:282-284`).

The request-header echo (`:173-176`) is a deliberate spec-compliance choice (reflecting `Access-Control-Request-Headers` is what the Fetch spec expects) and is safe because the value only ever reaches a `Vary`-differentiated response and is not cached without it.

### 7.2 Public-origin resolution

`src/server/origin/publicOrigin.ts:219-237` orders candidates:

```
request-url  →  configured  →  trusted-forwarded (ONLY if zero configured)  →  direct-local-host
```

`trustProxyMode()` (`:119-127`) reads `OMNIROUTE_TRUST_PROXY` with values `none | loopback | private`, and **anything unrecognized resolves to `none`** — fail-safe by default.

`sanitizeForwardedHost` (`:98-117`) rejects hosts containing `/ \ whitespace control-chars`, then re-parses via `new URL("http://" + host)` and rejects any result with credentials, a non-`/` path, a query, or a fragment. `sanitizeForwardedProto` (`:92-96`) accepts only literal `http`/`https`.

`directLocalHostOrigin` (`:195-217`) is the direct-LAN carve-out, and it applies **two independent checks** — the stamped peer must not classify as `remote`, and the `Host` header itself must be a loopback/private IP literal — precisely so a DNS-rebinding attacker who controls a domain resolving to a LAN IP still fails. The protocol is never taken from the attacker-controlled `Origin`.

### 7.3 WebSocket sidecar

| Control | Value | Evidence |
|---|---|---|
| Bind | `127.0.0.1` default | `liveServer.ts:44` |
| Origin allow-list | 3 loopback defaults + `LIVE_WS_ALLOWED_ORIGINS`; exact string compare | `liveServerAllowList.ts:19-23,41-44,103` |
| **Missing-Origin policy** | if `Origin` is undefined, allow **only** when `LIVE_WS_HOST` is a loopback name | `liveServerAllowList.ts:91-106` |
| LAN opt-in | `LIVE_WS_ALLOWED_HOSTS`, empty by default | `:49-51` |
| Token source | header-only, by design | `liveServer.ts:124-127` |
| Origin enforcement precedes auth | `4003` is sent before any credential check | `liveServer.ts:501-511` |
| Internal ingest | `POST /__omniroute_event`, **loopback-only, no token** | `liveServer.ts:311-324` |

The missing-Origin rule is the security-critical one: without it, a non-browser client would skip the browser check entirely. The code comment states this explicitly.

The unauthenticated internal ingest endpoint is a deliberate sidecar-internal design (the parent app posts events to the child), and it is loopback-restricted with no proxy-trust path.

### 7.4 Request hardening

| Control | Value | Evidence |
|---|---|---|
| IP filter | `checkRequestIP`; skipped for loopback; `null` IP when a proxy hop is stamped | `pipeline.ts:361-380` |
| Body size | per-route limit via `getBodySizeLimit(guardedPathname, settings)`; enforced pre-parse | `pipeline.ts:293-304` |
| Content-Type | `/v1/chat/completions` returns **415** if not `application/json` | `route.ts:83-99` |
| Single parse | `request.json()` exactly once — a second parse doubled heap residency on 270–550 KB agent payloads and fed the OOM loop (#4380) | `route.ts:126-141` |
| Admission control | capacity reservation + body buffering **before** JSON parse | `route.ts:104-112` |
| Internal connection capacity | 429 + `Retry-After` when over cap; **default 0 = disabled** | `src/sse/utils/backpressure.ts:3-13,25-51` |

---

## 8. MITM / TLS Interception

This is the highest-privilege subsystem and deserves explicit scrutiny.

### 8.1 Trust boundary

```
manager.ts (parent, Node)
  │
  ├─ installCleanupHandlers()          process.once(SIGINT/SIGTERM)        :290-298
  ├─ runPrivilegedMitmStep(...)        gated by canRunPrivilegedMitmSteps   :548-586
  │    └─ skips silently when no password / root / no-sudo
  ├─ sudo -S tee -a /etc/hosts         DNS entries, 127.0.0.1 + ::1         dns/dnsConfig.ts:93-179
  ├─ sudo -S cp + chmod 0644 + update-*   Linux CA trust                cert/install.ts:371-392
  ├─ security add-trusted-cert         macOS System keychain              cert/install.ts:344-369
  └─ certutil -addstore Root           Windows, via elevated PowerShell   cert/install.ts:429-436
       │
       ▼
  server.cjs (child, detached:false, NODE_ENV=production)                  :614-629
```

### 8.2 Privilege gating

`privilegedMitmStep.ts:7-17` — if `canRunPrivilegedMitmSteps(sudoPassword)` is false, the step is **logged and skipped**, not failed. Callers supply their own try/catch, so a failed trust install yields a running-but-untrusted bridge with a manual-trust guide rather than a hard failure. That is the right degradation for a self-hosted desktop product.

`sudoGate.ts:25-35` — password is not required on Windows, when already root, when a non-empty password is supplied, or when `isSudoPasswordRequired()` is false. `normalizeMitmSudoPasswordInput` treats whitespace-only as missing (#7865).

`OMNIROUTE_NO_SUDO` (`systemCommands.ts:71-74`) and `resolveSudoSpawn` (`:96-118`) support a genuinely unprivileged deployment: `sudo` is stripped from the argv, so `installCaCert` etc. run as the current user.

### 8.3 Graceful degradation in containers

`dns/provision.ts:110-145` — three independently-guarded best-effort steps (default DNS, per-agent DNS, custom hosts). Each catches and logs the full error. `provisionDnsEntries` returns early when `SKIP_ANTIGRAVITY_DNS==="true"` or when neither `sudo` nor root is available, which is the slim-Docker bail-out. `addDNSEntries` honours `OMNIROUTE_SKIP_DNS_WRITE==="1"`.

In Docker builds the manager is stubbed outright: `Dockerfile:114` sets `OMNIROUTE_MITM_STUB=1`, and `manager.stub.ts:11-13` makes `startMitm`/`stopMitm` throw a message directing the operator to `--webpack`.

### 8.4 Bypass correctness

`_internal/bypass.cjs:82-95` precedence: defaults → user globs → target hosts. Default patterns duplicate `passthrough.ts`: `/\.bank\./i`, `/(^|\.)gov(\.|$)/i`, `/(^|\.)okta\.com$/i`, `/(^|\.)auth0\.com$/i`. These correctly exempt banking, government, and SSO hosts from interception — an important safety property, since those flows frequently use certificate pinning and step-up auth that would break under MITM.

`bypassGlobMatch` (`:42-64`) rejects patterns with more than 9 segments — a limit that prevents pathological glob cost, and is tested (`mitm-passthrough.test.ts`, 10 cases).

`routeConnection` precedence is `bypass → target → passthrough` (`targets/index.ts:67-79`), and `isSelfLoopDestination` (`bypass.cjs:127-147`) returns HTTP **508 Loop Detected** rather than forwarding into itself (`server.cjs:421-428`).

### 8.5 Header handling on the interception path

`handlers/base.ts:132-152` — the client's `Authorization` is **stripped and replaced** by the router key, and hop-by-hop headers are removed via `sanitizeHeaders`. `server.cjs:440` — on passthrough, only `Host` is rewritten; everything else including `Authorization`/`Cookie` is forwarded verbatim to the real upstream, which is correct because passthrough is a transparent proxy to the genuine host.

`MITM_DISABLE_TLS_VERIFY=1` disables upstream verification (`server.cjs:432`). Default is on. The tproxy path independently defaults `rejectUnauthorized: true` (`tproxy/tlsCapture.ts:302-357`).

### 8.6 Findings

| ID | Sev | Finding | Evidence |
|---|---|---|---|
| **S-7** | Medium | Upstream `fetch` has **no timeout and no `AbortSignal`**; a hung upstream holds the request indefinitely. | `server.cjs:538-547`; also `handlers/base.ts:132-152` |
| **S-8** | Medium | Upstream DNS pinned to a hard-coded `8.8.8.8`, results cached for process lifetime. Deliberately bypasses `/etc/hosts`, but breaks on networks that block 8.8.8.8 and never re-resolves. | `server.cjs:324-333` |
| **S-9** | Medium | Readiness is a 2,000 ms **timer**, not a port probe. A slow-but-healthy start is reported failed; a hung-but-alive start is reported healthy. | `manager.ts:672-701` |
| **S-10** | Low | MITM always writes HTTP **200** for SSE responses regardless of upstream status. | `server.cjs:559-564` |
| **S-11** | Low | Legacy self-signed leaf key gets **no chmod**; only the root-CA path sets `0o600`. | `cert/generate.ts` vs `cert/rootCa.ts:45-73` |
| **S-12** | Low | Root CA has **no expiry check** on reuse; the tproxy CA is generated with a 10-year validity. | `cert/rootCa.ts:45-73`; `tproxy/dynamicCert.ts:35-49` |
| **S-13** | Info | tproxy supports **iptables only** — no nftables path exists anywhere in the tree. | `tproxy/commands.ts:79-121` |
| **S-14** | Info | Trae target is a deliberate non-routable placeholder (`trae.invalid`, `viability: "investigating"`) and its handler throws "Not yet implemented". Correctly gated. | `targets/trae.ts:12-32`; `handlers/trae.ts:16-23` |

### 8.7 tproxy native boundary

`native/transparent.c` — `socket(AF_INET, SOCK_STREAM)` → `SO_REUSEADDR` → `setsockopt(SOL_IP, IP_TRANSPARENT)` → `bind` → `listen(fd, 511)`. `IP_TRANSPARENT` requires `CAP_NET_ADMIN`. `ConnectMarked` sets `SO_MARK` **before** `connect` and tolerates `EINPROGRESS`. IPv4 only. The addon is loaded from `native/build/Release` or `native/prebuilds`; `.gitignore` excludes both, so no binary is committed. `native/README.md` documents that only `linux-x64` is built.

---

## 9. CI Security Gates

### 9.1 Blocking gates

| Gate | Mechanism | Location |
|---|---|---|
| **CodeQL alert ratchet** | alert count frozen at **2**; a rise blocks | `ci.yml:260-267`, `config/quality/quality-baseline.json` |
| **Secret findings** | `check:secrets -- --ratchet`, frozen at **0** | `quality-baseline.json` |
| **Dependency vulns** | osv-scanner ratchet, frozen at **10** | `quality-baseline.json` |
| **Workflow lint** | zizmor, frozen at **190 findings** | `quality-baseline.json` |
| **Dependency audit** | `npm audit` in the lint job | `ci.yml` lint job |
| **License allowlist** | SPDX allowlist enforced | `config/quality/.license-allowlist.json` |
| **Dockerfile lint** | hadolint, error threshold | `ci.yml` lint job |
| **Gitleaks** | blocking ratchet | `quality.yml` `fast-gates` |
| **Public-credential scan** | dedicated `check:public-creds` script | `ci.yml` lint job |
| **OpenAPI breaking changes** | oasdiff, frozen at **0** | `quality-baseline.json` |
| **i18n UI coverage** | frozen at **100%** | `quality-baseline.json` |

### 9.2 Advisory gates

| Gate | Why advisory |
|---|---|
| semgrep (OWASP Top Ten + secrets) | reports SARIF; does not block |
| DAST smoke (schemathesis + promptfoo) | `continue-on-error`; flaky by nature |
| nightly schemathesis contract fuzz | advisory |
| OpenSSF Scorecard | weekly, informational |
| circular-deps | known debt, ratcheted separately |
| SonarQube | no fail-gate configured |
| `test-bun-sqlite` on Windows | advisory |

### 9.3 Scheduled security runs

| Workflow | Schedule | What it does |
|---|---|---|
| `nightly-llm-security.yml` | 05:53 | `promptfoo` injection guard (block mode) + `garak` probes (skipped without a secret) |
| `nightly-mutation.yml` | 03:17 | 9 parallel Stryker batches over `auth`, `accountFallback`, `security`, `combo`, `chatCore` |
| `nightly-resilience.yml` | 04:41 | heap, chaos, k6 soak, a11y |
| `nightly-property.yml` | 06:00 | `FC_SEED=random FC_NUM_RUNS=2000` property fuzz; files an issue with the failing seed |
| `nightly-compat.yml` | 06:47 | Node **24 and 26** × 4 shards, plus a Node 26 webpack build |
| `scorecard.yml` | weekly | OpenSSF Scorecard + SARIF artifact |
| `codeql.yml` | manual | `javascript-typescript` + `security-extended` |

The mutation-testing focus on `auth` is a strong signal: the security-critical modules are being tested by assertion, not just by execution.

### 9.4 Assessment

**This is an above-average security posture for a self-hosted project.** The blocking CodeQL/secret/vuln ratchets mean the security posture cannot silently degrade. The 190-finding zizmor baseline is the one number that deserves scrutiny — it is high, and it is blocking, so it cannot be ignored, but it should trend down.

**Gaps worth closing, in priority order:**

1. **S-2 (A2A auth)** — add `requireManagementAuth` to the A2A routes.
2. **Encryption key absence is silent** — warn loudly like the `CHANGEME` password check.
3. **Login lockout dimensions** — add per-account; document the multi-instance limitation.
4. **CSRF replayability** — document as an accepted property.
5. **PII redaction off by default** — must be disclosed in any compliance attestation; the design decision is defensible but it is a decision.
6. **MITM timeouts** — S-7, S-8, S-9 are all straightforward fixes with disproportionate reliability impact.

---

## 10. Compliance Posture

| Framework | Status | Basis |
|---|---|---|
| MIT license | **Confirmed for OmniRoute** — `package.json:77` declares `"license": "MIT"` and `/root/OmniRoute/LICENSE` is the 1,069-byte MIT text (© 2026 diegosouzapw) | measured |
| Reports-repo licence | **Was inconsistent** — the report repository had committed an **Apache-2.0** `LICENSE` (11,357 bytes) while its own README declared MIT. Corrected to MIT to match the declaration and upstream; **owner confirmation requested**. | measured |
| GDPR (right to erasure) | Partial | Per-key `no_log`; PII redaction available but off by default; `db/cleanup.ts` (760 lines) implements 8 retention-scoped cleaners plus artifact purge |
| SOC 2 | Not claimed | No control framework present; do not imply otherwise |
| Data residency | Self-hosted by design | All data in local SQLite + filesystem artifacts; no required third-party processing |
| Audit trail | Strong | MCP tool audit (hash-only inputs), call logs (summary + hashed artifacts), timeline + high-level-actions audit modules, no-log capability |
| Retention | Configurable, 7-day defaults | `logEnv.ts`: `APP_LOG_RETENTION_DAYS` 7, `CALL_LOG_RETENTION_DAYS` 7, `CALL_LOG_MAX_ENTRIES` 10,000, `CALL_LOGS_TABLE_MAX_ROWS` 100,000, `PROXY_LOGS_TABLE_MAX_ROWS` 100,000 |

**Retention defaults are short (7 days)** — appropriate for a self-hosted ops tool, but operators running compliance-sensitive workloads should raise them and should know the knobs exist.

---

## 11. Reproduction Commands

```bash
cd /root/OmniRoute

# Header stripping (the load-bearing step)
sed -n '306,332p' src/server/authz/pipeline.ts

# Fail-closed classification
sed -n '119,126p' src/server/authz/classify.ts

# Spawn-capable containment
sed -n '242,267p' src/server/authz/routeGuard.ts

# Via-proxy downgrade
sed -n '89,99p' src/server/authz/peerStamp.ts

# Pinned GCM auth tag
sed -n '29,45p' src/lib/db/encryption.ts

# Audit stores hash, not input
sed -n '355,395p' open-sse/mcp-server/audit.ts

# Missing-Origin WS policy
sed -n '91,106p' src/server/ws/liveServerAllowList.ts

# A2A has no requireManagementAuth (finding S-2)
grep -rn "requireManagementAuth" src/app/api/a2a/ || echo "NOT CALLED — confirmed"

# No shell interpolation in MITM
grep -rn "exec(" src/mitm/ || echo "no exec() in src/mitm — confirmed"
grep -n "process.argv" src/mitm/dns/dnsConfig.ts

# Login lockout has no username dimension
sed -n '54,57p;76,111p' src/server/auth/loginGuard.ts

# Quality baseline security numbers
cat config/quality/quality-baseline.json | head -40
```

---

## 12. Findings Summary

| ID | Sev | Finding | Section |
|---|---|---|---|
| S-2 | High | A2A task routes have no in-handler authorization | §3.8 |
| S-3 | High | Client sends WS token in query string; server reads headers only | §7.3 |
| S-5 | Medium | CSRF tokens replayable within 10-min TTL | §3.7 |
| S-6 | Medium | Login lockout is per-IP, in-process, no username dimension | §2.5 |
| S-7 | Medium | MITM upstream fetch has no timeout/abort | §8.6 |
| S-8 | Medium | MITM DNS hard-coded to 8.8.8.8, cached forever | §8.6 |
| S-9 | Medium | MITM readiness is a timer, not a probe | §8.6 |
| S-10 | Low | MITM always returns 200 for SSE | §8.6 |
| S-11 | Low | Legacy leaf key has no chmod | §8.6 |
| S-12 | Low | Root CA has no expiry check | §8.6 |
| S-13 | Info | tproxy is iptables-only, no nftables | §8.6 |
| S-14 | Info | Trae is a deliberate stub | §8.6 |
| — | Medium | `STORAGE_ENCRYPTION_KEY` absence is silent | §4.1 |
| — | **High** | Report repo shipped an Apache-2.0 `LICENSE` while declaring MIT — a licence-compliance defect. Corrected to MIT; owner confirmation pending. | §10 |
| — | Medium | Credential masker is opt-in (`CREDENTIAL_REDACTION_ENABLED`) | §6.1 |
| — | Info | PII redaction off by default (documented Hard Rule #20) | §6.1 |
