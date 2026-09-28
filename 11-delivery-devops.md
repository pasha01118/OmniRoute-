# 11 — Delivery, DevOps & Quality Review

> **Verification basis.** OmniRoute `5764027`, version `3.8.50`. All workflow, gate, and baseline numbers read from `.github/workflows/`, `config/quality/`, and root config files.

---

## 1. Pipeline Inventory

**24 workflows.** CI runs on **Node 24**; nightly compatibility runs **Node 24 and 26**.

### 1.1 Blocking quality gates

| Job | Mechanism |
|---|---|
| CodeQL alert ratchet | frozen at **2**; a rise blocks (`ci.yml:260-267`) |
| Secret findings | frozen at **0** |
| Dependency vulns (osv-scanner) | frozen at **10** |
| Workflow lint (zizmor) | frozen at **190** |
| OpenAPI breaking changes (oasdiff) | frozen at **0** |
| Coverage floor | **60 / 60 / 60 / 60** (statements / lines / functions / branches) |
| i18n UI coverage | frozen at **100%** |
| ESLint | `--max-warnings 0`; `no-eval` / `no-new-func` / `no-explicit-any` are errors |
| Bundle size | frozen ratchet |
| Gitleaks secret scan | blocking ratchet |
| Dockerfile lint (hadolint) | error threshold |
| Type coverage | frozen at **92.17%** |
| Dead exports (knip) | frozen at **230** |

The ratchet pattern — a frozen baseline where only *regression* fails — is the strongest quality mechanism in this repository. It converts "quality is good" into "quality cannot silently degrade."

### 1.2 Advisory gates

| Job | Why advisory |
|---|---|
| semgrep (OWASP Top Ten + secrets) | SARIF report, no fail gate |
| DAST smoke (schemathesis + promptfoo) | `continue-on-error`; inherently flaky |
| nightly schemathesis contract fuzz | advisory |
| OpenSSF Scorecard | weekly badge |
| circular-deps | known debt, ratcheted separately |
| SonarQube | no fail gate |
| `test-bun-sqlite` on Windows | advisory |

### 1.3 Test matrix

| Tier | Shards | Count |
|---|---|---|
| Unit (Node native) | 8 (CI) / 4 (quality workflow) | **4,201 files** |
| Vitest (jsdom) | 1 | included above |
| Integration | 2 | 109 files |
| E2E (Playwright chromium) | **9, duration-balanced via LPT** | 39 specs |
| Security | 1 | `npm run test:security` |
| Coverage merge | 8 shards combined | — |

E2E shard weights live in `config/quality/e2e-timings.json` and are balanced longest-processing-time-first. For a 39-spec E2E suite this is arguably over-engineered, but it is the correct pattern if the suite grows.

`vitest.config.ts` uses a threads pool (20 workers), jsdom, and a **~50-file `exclude` list of pre-existing failures tracked by issue #8618**, plus `config/quality/test-discovery-baseline.json` allowing 13 orphan tests and `test-masking-allowlist.json`.

**Assessment:** excluding ~50 known-failing files is a legitimate triage mechanism *when tracked and shrinking*. The risk is that the list becomes permanent. It should have a ratchet (count must decrease), and `test-discovery-baseline.json` suggests the team already has that mechanism for discovery — the same treatment should apply to the exclusion list.

---

## 2. Scheduled Workflows

| Workflow | Cron | Purpose |
|---|---|---|
| `nightly-mutation.yml` | 03:17 | 9 parallel Stryker batches over `auth`, `accountFallback`, `security`, `combo`, `chatCore`; incremental cache + mutation ratchet |
| `nightly-schemathesis.yml` | 04:23 | OpenAPI contract fuzz (advisory) |
| `nightly-resilience.yml` | 04:41 | heap, chaos, k6 soak, a11y |
| `nightly-llm-security.yml` | 05:53 | `promptfoo` injection guard (**block mode**) + `garak` probes |
| `nightly-release-green.yml` | 05:23 / 12:23 / 18:23 | release-branch and main green checks; deduplicated base-red issue |
| `nightly-property.yml` | 06:00 | fast-check `FC_SEED=random FC_NUM_RUNS=2000`; files an issue with the failing seed |
| `nightly-compat.yml` | 06:47 | Node **24 and 26** × 4 shards + Node 26 webpack build |

**Mutation testing focused on `auth`** is the standout. Assertion-based mutation testing on the security-critical modules is materially stronger than coverage on those modules, and it is the correct place to spend the compute.

The `nightly-llm-security` **promptfoo in block mode** is also notable — it treats prompt-injection regression as a build-breaking condition rather than an advisory.

`nightly-compat` running Node 26 alongside 24 is forward-looking: `package.json` engines allow `<27`, and `Dockerfile:2` already uses `node:26-trixie-slim`, so nightly validates the runtime the production image actually uses.

---

## 3. Quality Ratchet Architecture

`config/quality/` holds 18 baseline files. The ratchet model is consistent:

```
baseline JSON  →  measure  →  fail if WORSE than baseline  →  --update to tighten
```

| Baseline | Frozen value |
|---|---|
| `quality-baseline.json` | ~90 metrics: eslint warnings/errors 0, coverage 80.8/80.8/86.42/78.1, typeCoverage 92.17%, deadExports 230, cognitiveComplexity 1223, codeqlAlerts 2, secretFindings 0, vulnCount 10, zizmorFindings 190, bundleSize 8045, openapiBreaking 0, i18nUiCoverage 100, 30+ `mutationScore.*` |
| `complexity-baseline.json` | 2774 |
| `file-size-baseline.json` | cap 1000 (test cap 1000) + 342 rebaseline notes |
| `duplication-baseline.json` | jscpd 5.72% |
| `eslint-suppressions.json` | 3,342 lines of per-file frozen suppressions |
| `dashboard-typecheck-baseline.json` | frozen TS error count |
| `open-sse-typecheck-baseline.json` | frozen TS error count |
| `dependency-allowlist.json` | anti-slopsquatting |
| `.license-allowlist.json` | SPDX whitelist |
| `test-discovery-baseline.json` | 13 orphan tests allowed |
| `test-masking-allowlist.json` | — |
| `install-upgrade-allowlist.json` | — |
| `forgotten-sibling-allowlist.json` | empty |
| `e2e-timings.json` | LPT shard weights |

**This is the most sophisticated quality system in the reports' subject matter.** The 3,342-line ESLint suppression file is unusual — most projects would delete those suppressions; this one freezes and ratchets them.

**Two observations worth acting on:**

1. **342 rebaseline notes** in `file-size-baseline.json` means the file-size cap has been rebaselined 342 times. Each rebaseline is a small admission that the codebase grew. The cap is enforced, but the trajectory is what matters.
2. **Frozen typecheck error baselines** exist for both the dashboard and `open-sse`, and `next.config.mjs:296` sets `typescript.ignoreBuildErrors: true`. So a green build is **not** evidence of type safety; the typecheck ratchet is the compensating control. This is documented in the baseline names but should be stated in contributor docs.

---

## 4. Build

### 4.1 Framework configuration

`next.config.mjs` (673 lines):

| Setting | Value | Line |
|---|---|---|
| `output` | `"standalone"` | `:160` |
| `distDir` | custom | — |
| `serverActions.bodySizeLimit` | default `50mb` | — |
| `serverExternalPackages` | `better-sqlite3`, `pino`, … | — |
| **`typescript.ignoreBuildErrors`** | **`true`** | `:296` |
| `outputFileTracingIncludes` | includes `./src/mitm/server.cjs` | `:219` |
| `images.unoptimized` | `true` | — |
| `MITM` alias | `@/mitm/manager` → stub, controlled by `OMNIROUTE_MITM_STUB` | `:118` |

`outputFileTracingIncludes` explicitly including `src/mitm/server.cjs` is the kind of detail that breaks standalone builds if forgotten — and `tests/unit/build/mitm-server-bundle-contents.test.ts` (2 cases) guards it.

The MITM stub alias exists because Turbopack cannot bundle native modules. `manager.stub.ts:11-13` makes `startMitm`/`stopMitm` throw with the message *"MITM manager stub reached at runtime — build alias applied incorrectly. Use --webpack for production builds or verify Turbopack is not aliasing at runtime."* A clear, actionable failure beats a confusing one. `next.config.mjs:118` and `scripts/build/mitm-stub-flag.mjs:22` control it; `tests/unit/next-config.test.ts:130` asserts the Docker profile sets the alias.

### 4.2 Toolchain

| Tool | Version | Notes |
|---|---|---|
| Next.js | `^16.2.11` | |
| React | `19.2.8` | **pinned exact** |
| TypeScript | `^6.0.3` | `strict: false` in `tsconfig.json` |
| Tailwind | `^4.0.0` | `@theme inline` in `globals.css` |
| Vitest | `^4.1.7` | |
| Playwright | `^1.60.0` | |
| ESLint | `^9.39.4` | flat config |
| Prettier | `^3.8.3` | |
| c8 | `^12.0.0` | |
| knip | `^6.18.0` | dead exports |
| Express | `^5.2.1` | |
| ws | `^8.18.0` | live sidecar |
| sharp | `^0.35.3` | |
| pino | `^10.3.1` | |
| `better-sqlite3` | `^13.0.2` | **optionalDependency** |
| `@atjsh/llmlingua-2` | `2.0.3` | compression engine |
| Bun | `1.3.14` | declared, no `bun.lockb` |
| material-symbols | `^0.45.2` | self-hosted icons |

**Two package-manager findings:**

1. **No `packageManager` field, no `pnpm-lock.yaml`, no `yarn.lock`, no `bun.lockb`** — only `package-lock.json`. The report README says `pnpm ≥ 8.0.0`; the report checklist says `npm`. **npm is correct**, and the README is wrong.
2. **Bun 1.3.14 is declared** but has no lockfile, and CI has a `test-bun-sqlite` job. So Bun is a *test* target, not the package manager.

`tsconfig.json`: `strict: false`, `target: ES2022`, `moduleResolution: bundler`, paths `@/*` → `src` and `@omniroute/open-sse/*`. `strict: false` combined with `no-explicit-any` as an ESLint error is a coherent split — TypeScript is permissive, the linter is strict about the most dangerous escape hatch.

---

## 5. Container Delivery

### 5.1 Dockerfile

`Dockerfile` (~15.5 KB), multi-stage:

| Aspect | Value | Line |
|---|---|---|
| Base | `node:26-trixie-slim` | `:2` |
| Build arg | `OMNIROUTE_USE_TURBOPACK` | `:103` |
| **MITM stub** | `OMNIROUTE_MITM_STUB=1` | `:114` |
| Exposed port | 20128 | — |
| User | `USER node` (non-root) | — |
| `DATA_DIR` | `/app/data` | — |
| Healthcheck | present | — |
| Targets | `runner-web`, `runner-cli` | — |

`OMNIROUTE_MITM_STUB=1` means **the MITM manager is deliberately absent from container images**. Correct — the MITM subsystem needs host-level DNS and trust-store access that a container cannot have — and correctly guarded by `tests/unit/next-config.test.ts:130`.

Non-root `USER node` with a `DATA_DIR` the app owns is correct hardening.

### 5.2 Compose

`docker-compose.yml` (11.6 KB): `redis:8.6.5-alpine` + `base`/`web`/`cli` services + a `chatgpt-web-codex-browser` profile, with an env block for `DATA_DIR`, `PORT 20128`, `API_PORT 20129`, `LIVE_WS`, `REDIS_URL`.

`docker-compose.prod.yml` (4 KB): `redis` + `prod` + the browser profile.

`redis:8.6.5-alpine` is a current, specific pin. Redis is **optional** — every consumer has a SQLite fallback, and `quota/storeFactory.ts:83-96` warns (with the URL credential-stripped) when `driver=redis` is set without a URL.

### 5.3 Release pipeline

| Workflow | Trigger | Behaviour |
|---|---|---|
| `docker-publish.yml` | push to main / release / `v*` / tags / `released` | 3 jobs (prepare/build/merge), multi-platform matrix → GHCR + Docker Hub, least-privilege |
| `npm-publish.yml` | `released` / manual / `workflow_call` | staged publish, 2FA, semver + dist-tag guards, **provenance** |
| `electron-release.yml` | tag `v*` / manual | 5 jobs including `verify-desktop-assets` + `publish-npm` |
| `deploy-vps.yml` | after Docker publish / manual | SSH reachability guard, gated on `DEPLOY_ENABLED` |
| `lock-released-branch.yml` | release published / push `release/v*` | locks the branch, guards against post-release pushes |

**npm provenance** means the published package is cryptographically attributable to the CI workflow — a meaningful supply-chain control.

`lock-released-branch.yml` prevents a post-release push from silently diverging a shipped version. That is a mature release practice.

`DEPLOY_ENABLED` on `deploy-vps.yml` means production deploys are opt-in per run rather than automatic, which is the safer default for a self-hosted project.

---

## 6. Security Scanning Coverage

| Scanner | Mode | Rules |
|---|---|---|
| **CodeQL** | blocking ratchet (2 alerts) | `javascript-typescript`, `security-extended` |
| **gitleaks** | blocking ratchet | ~170 default rules + project allowlists |
| **osv-scanner** | blocking ratchet (10 vulns) | dependency vulns |
| **zizmor + actionlint** | blocking ratchet (190) | GitHub Actions security |
| **semgrep** | advisory | `p/owasp-top-ten`, `p/secrets` |
| **hadolint** | blocking (error threshold) | Dockerfile |
| **promptfoo** | blocking (nightly) | prompt-injection guard |
| **garak** | nightly (needs a secret) | LLM-specific probes |
| **schemathesis** | advisory | OpenAPI contract fuzz |
| **Stryker** | nightly ratchet | mutation testing |
| **OpenSSF Scorecard** | weekly | supply-chain posture |
| **SonarQube** | advisory | code quality |

`.gitleaks.toml` uses `useDefault = true` plus project allowlists for fixtures and public OAuth credentials — with mandatory justification. Documented exceptions are the right way to handle a legitimately-public test credential.

`Semgrep` being advisory while CodeQL is blocking is a defensible split: CodeQL's dataflow analysis produces fewer false positives on a codebase this size.

---

## 7. Documentation Pipeline

`docs-sync-strict` (blocking) and `docs-lint` (advisory) jobs, plus `wiki-sync.yml` (docs-path push → repo wiki).

`check:docs-all` appears in the report checklist's PR gates. Given that **eight marketing numbers are currently wrong** (§8), the docs pipeline is not catching claim drift — only formatting and link validity. A `check:docs-truth` script that re-derives counts from the executed registries and diffs them against the docs would close the gap that caused all eight errors.

---

## 8. Documentation Drift (Delivery Risk)

These are delivery concerns because stale operational documentation causes outages:

| Claim | Documented | Measured | Impact |
|---|---|---|---|
| Provider count | 291 | **338** | Operators underestimate the catalog by 16% |
| MCP tools | 105 | **108** | — |
| MCP scopes | 31 | **18** | Operators may request scopes that do not exist |
| A2A skills | 5 | **6** | — |
| Routing strategies | 19 | **20** | — |
| Compression engines | 12 | **15** | — |
| Compression savings | 15–95% (~89% avg) | **not reproducible** | **Users size capacity against a number the project cannot substantiate** |
| Breaker thresholds (OAuth/API-key/local) | 3 / 5 / 2 | **8 / 12 / 2** | Operators tune 60% too low → premature breaker trips |
| Migrations | 130 | **144** | — |
| Package manager | `pnpm ≥ 8.0.0` | **npm** (`package-lock.json` only) | New contributors run the wrong install |
| Test runner | "Jest/Node" | **Vitest + Node native**; no jest installed | — |

**Root cause of the count drift is one line.** `scripts/docs/gen-provider-reference.ts` imports `NOAUTH_PROVIDERS` at line 8 and never uses it in `main()` (`:180-196`).

**This is a delivery defect, not a documentation nit.** An operator who sizes infrastructure against "15–95% token savings" and finds no savings will conclude the product is broken.

---

## 9. Findings

| ID | Sev | Finding | Section |
|---|---|---|---|
| **D-1** | High | Docs generator omits `NOAUTH_PROVIDERS`, so 8 marketing counts are stale. | §8 |
| **D-2** | High | Compressor savings claim is not reproducible from the project's own frozen corpus; operators may size capacity against it. | §8, `06` §5.5 |
| **D-3** | Medium | `AGENTS.md` breaker thresholds contradict source (3/5/2 vs 8/12/2) — operators tuning from docs set 60% too low. | §8 |
| **D-4** | Medium | `typescript.ignoreBuildErrors: true` plus frozen typecheck baselines means a green build is not type-safety evidence. | §2, §4.1 |
| **D-5** | Medium | README specifies `pnpm`; the tree has only `package-lock.json`. | §4.2 |
| **D-6** | Medium | `vitest.config.ts` excludes ~50 pre-existing failing files with no count ratchet. | §1.3 |
| **D-7** | Low | 342 file-size rebaseline notes indicate a growing-file trajectory. | §3 |
| **D-8** | Low | zizmor baseline is 190 findings — high, though blocking so it cannot be ignored. | §1.1 |
| **D-9** | Low | Bun 1.3.14 is declared with no lockfile, which reads as an unused dependency. | §4.2 |
| **D-10** | Info | No per-provider quality eval; routing quality regressions are undetectable. | `07` §7 |
| **D-11** | Info | Codecov is fully `informational: true`; the real gate is c8 at 60%. | §1.1 |
| **D-12** | Info | `chatCore` coverage 72.45% is the lowest of the ratcheted modules and the highest-complexity. | §1.1 |

---

## 10. Recommendations

### 10.1 Immediate — documentation truth

1. Fix `scripts/docs/gen-provider-reference.ts` to union all ten collections and **generate** the total (D-1).
2. Add `npm run check:docs-truth`: re-derive provider count, MCP tool count, MCP scope count, A2A skill count, strategy count, engine count, and migration count from the executed registries, and fail on drift. This makes D-1 structurally impossible to reintroduce.
3. Correct `AGENTS.md` breaker thresholds to 8/12/2 and annotate `local` as unwired (D-3).
4. Correct the README package manager to `npm` (D-5) and the report README likewise.

### 10.2 Immediate — compression claim

5. Make the compression budget gate **bidirectional** and add a per-engine minimum-effect assertion.
6. Extend the benchmark corpus with shell-heavy and tool-output-heavy tasks so RTK is actually exercised.
7. Re-measure, then either substantiate the savings claim with a reproducible number or **retract it**. An unsubstantiated performance claim in marketing copy is a liability, not an asset.

### 10.3 Short term — pipeline hardening

8. Add a count ratchet to the `vitest.config.ts` exclusion list so it must shrink (D-6).
9. State explicitly in contributor docs that a green build is not type-safety evidence, and that the `dashboard-typecheck` gate is mandatory (D-4).
10. Add a per-provider quality eval, starting with the providers behind default strategies (D-10).
11. Either remove the Bun declaration or document it as a test-only runtime (D-9).

### 10.4 Medium term

12. Track the zizmor baseline's trajectory and set a downward target (D-8).
13. Report `file-size-baseline` rebaseline counts as a tracked metric, so growth is visible rather than buried in a notes field (D-7).
14. Add a `chatCore` mutation or coverage target that reflects its complexity (D-12).

---

## 11. Reproduction

```bash
cd /root/OmniRoute

# Ratchet baselines
cat config/quality/quality-baseline.json | head -60
cat config/quality/complexity-baseline.json
cat config/quality/duplication-baseline.json

# The missing union (D-1)
sed -n '1,20p;175,200p' scripts/docs/gen-provider-reference.ts
grep -c "NOAUTH" scripts/docs/gen-provider-reference.ts

# Breaker thresholds — source of truth, not AGENTS.md
sed -n '222,268p' open-sse/config/constants.ts
grep -n "threshold" AGENTS.md | head

# ignoreBuildErrors (D-4)
grep -n "ignoreBuildErrors" next.config.mjs
cat config/quality/dashboard-typecheck-baseline.json | head

# Package manager (D-5)
ls package-lock.json pnpm-lock.yaml yarn.lock bun.lockb 2>&1
node -e "console.log(require('./package.json').packageManager)"

# Test exclusion list (D-6)
grep -c "\.test\." vitest.config.ts
cat config/quality/test-discovery-baseline.json

# Compression claim (D-2)
cat scripts/check/compression-budget-baseline.json
sed -n '45,70p' open-sse/services/compression/budget/budgetGate.ts

# Workflow inventory
ls .github/workflows/ | wc -l
grep -n "CI_NODE_VERSION" .github/workflows/ci.yml

# Coverage gates
sed -n '950,960p' .github/workflows/ci.yml
grep -A3 '"coverage"' config/quality/quality-baseline.json
grep -n "informational" codecov.yml

# Container hardening
grep -n "FROM node\|OMNIROUTE_MITM_STUB\|USER node\|DATA_DIR\|EXPOSE" Dockerfile

# ES2022 / strict:false
grep -n '"strict"\|"target"\|"moduleResolution"' tsconfig.json
```
