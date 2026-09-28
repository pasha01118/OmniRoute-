# OmniRoute Dashboard Redesign

<div align="center">

![OmniRoute Banner](https://img.shields.io/badge/OmniRoute-Dashboard%20Redesign-0A0E14?style=for-the-badge&logo=github&logoColor=00FFFF)
![Version](https://img.shields.io/badge/Version-2.0.0-00FF88?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-FFB800?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Verified-00FFFF?style=for-the-badge)

**Enterprise-grade dashboard redesign for OmniRoute — a unified AI router with 338 providers** <sub>(measured at `5764027`; upstream copy says 291)</sub>

[📋 Architecture](./01-architecture-blueprint.md) • [🎨 Design System](./02-design-system.md) • [🗺️ Roadmap](./03-development-roadmap.md) • [✅ Checklist](./04-implementation-checklist.md) • [🔐 Security](./05-security-compliance.md) • [⚡ Performance](./06-performance-scalability.md) • [🤖 Agentic AI](./07-agentic-ai.md)

</div>

---

## 🎯 Project Overview

This repository contains a **code-verified design specification and implementation roadmap** for the OmniRoute Dashboard redesign, plus seven engineering deep-dives that audit the underlying platform.

**Everything here was measured against the OmniRoute source tree**, not copied from its marketing copy. Where a widely-repeated claim did not survive measurement, it is corrected with a `path:line` reference and a reproduction command.

> **Start with [`00-comprehensive-master-plan.md`](./00-comprehensive-master-plan.md).** It records the verification basis, the ten findings that change how the rest of the set should be read, the verified platform facts, the corrected architecture, the redesigned phase plan, and 15 ranked engineering findings.

### 🏗️ What is OmniRoute?

OmniRoute is a **unified AI proxy/router** providing:
- **338 LLM providers** (measured; upstream copy says 291) with automatic cross-provider fallback
- **Real-time routing** across 20 combo strategies
- **MCP / A2A protocol support** — 108 MCP tools, 18 scopes, 3 transports
- **A TLS-interception bridge** capturing third-party coding-agent traffic across 10 targets
- **Enterprise features**: semantic caching, circuit breakers, model lockout, memory, guardrails
- **Desktop app** via Electron, plus PWA and Termux support

---

## ⚠️ Corrections to Upstream Copy

Every number below was measured. Eight widely-repeated claims did not survive:

| Claim | Documented | **Measured** | Why it drifted |
|---|---|---|---|
| Providers | 291 | **338** | `scripts/docs/gen-provider-reference.ts` imports `NOAUTH_PROVIDERS` at line 8 and never unions it in `main()` |
| MCP tools | 105 | **108** | The 12 tool collections overlap (122 raw → 108 after `Set` dedupe) |
| MCP scopes | 31 | **18** | `AGENTS.md` overstates `schemas/tools.ts` |
| A2A skills | 5 | **6** | `src/lib/a2a/skills/` has 6 files and 6 handlers |
| Routing strategies | 19 | **20** | `strategyDispatch.ts:47-68` |
| Compression engines | 12 | **15** | `compression/engines/index.ts:18-45` |
| SQL migrations | 130 | **144** | — |
| Breaker thresholds (OAuth/API-key/local) | 3 / 5 / 2 | **8 / 12 / 2** | `AGENTS.md` contradicts `open-sse/config/constants.ts:222-268` |
| Token savings | 15–95% (~89% avg) | **not reproducible** | 5 of 8 engines score *byte-identical* to the no-op `lite` control on the project's own frozen corpus |

**Three redesign assumptions are also wrong.** The dashboard uses **Inter** (not DM Sans), **Material Symbols** self-hosted (not Phosphor), and **Framer Motion is not a dependency at all**. See [`10-ux-design.md`](./10-ux-design.md) §2.

**One compliance defect was found and fixed:** the committed `LICENSE` was **Apache-2.0** while this README declared MIT and OmniRoute is MIT. Corrected, with owner confirmation requested.

---

## 📦 Repository Contents

### Part A — Plan & Foundation

| File | Description | Size |
|------|-------------|------|
| [`00-comprehensive-master-plan.md`](./00-comprehensive-master-plan.md) | **Entry point.** Verification basis, 10 key findings, verified platform facts, verified architecture, redesigned phase plan, engineering findings, definition of done | 33.9 KB |
| [`01-architecture-blueprint.md`](./01-architecture-blueprint.md) | System architecture, request pipeline, resilience model, authZ tiers (all counts corrected) | 39.1 KB |
| [`02-design-system.md`](./02-design-system.md) | Tokens, component specs, motion, accessibility — **with a verification notice** (DM Sans / Phosphor / Framer Motion are not installed) | 18.9 KB |
| [`03-development-roadmap.md`](./03-development-roadmap.md) | Phased plan with daily tasks — **rebudgeted** (the original was ~40% optimistic) | 16.4 KB |
| [`04-implementation-checklist.md`](./04-implementation-checklist.md) | QA and acceptance criteria — **file inventory corrected** (`DataTable.tsx` already exists; `Table`/`Tabs`/`Toast` do not) | 15.5 KB |

### Part B — Engineering Deep Dives

| File | Description | Size |
|------|-------------|------|
| [`05-security-compliance.md`](./05-security-compliance.md) | Authz model, cryptography, input validation, secret redaction, CORS/origin resolution, MITM trust boundary, 14 CI security gates, compliance posture | 36.4 KB |
| [`06-performance-scalability.md`](./06-performance-scalability.md) | Load shedding, circuit breakers, 7 caches, streaming, sidecar capacity, build/bundle — 13 findings | 31.5 KB |
| [`07-agentic-ai.md`](./07-agentic-ai.md) | MCP, A2A, skills, memory, guardrails, evals, agent bridge — 10 findings | 28.8 KB |
| [`08-mcp-integrations.md`](./08-mcp-integrations.md) | MCP transports, auth chain, 108-tool surface, scope enforcement, audit, all integration surfaces | 15.6 KB |
| [`09-networking-proxy.md`](./09-networking-proxy.md) | Six network surfaces, egress, origin resolution, peer stamping, sidecar, MITM transport, TPROXY — 12 findings | 32.5 KB |
| [`10-ux-design.md`](./10-ux-design.md) | The design system as it actually exists, 4 corrections, component inventory, rebudgeted 12-phase plan — 13 findings | 24.1 KB |
| [`11-delivery-devops.md`](./11-delivery-devops.md) | 24 CI workflows, quality ratchets, build, container, release pipeline, security scanning — 12 findings | 19.7 KB |

---

## 🎨 Design Philosophy

<table>
<tr>
<td width="50%" valign="top">

### 🌑 Charcoal Metallic Base
Deep, sophisticated dark theme optimized for OLED displays and long coding sessions.

```css
--color-bg: #0A0E14;           /* Deep charcoal */
--color-bg-elevated: #111822;  /* Elevated surfaces */
--color-bg-card: #151D2B;      /* Card backgrounds */
--color-metallic-border: #3A4A5C;
```

</td>
<td width="50%" valign="top">

### ⚡ Neon Accent Palette
Thin, bright emissive borders that create hierarchy without overwhelming.

```css
--neon-cyan: #00FFFF;      /* Primary actions */
--neon-emerald: #00FF88;   /* Success states */
--neon-amber: #FFB800;     /* Warning / attention */
--neon-coral: #FF4E6D;     /* Error / critical */
--neon-violet: #B880FF;    /* Special features */
```

</td>
</tr>
</table>

### ✨ Neon Ivory Typography
Warm, readable text replacing harsh pure white for reduced eye strain.

```css
--neon-ivory: #FFF8E7;       /* Primary text */
--neon-ivory-dim: #E8DCC8;   /* Secondary text */
--neon-ivory-subtle: #B8A994; /* Muted / placeholder */
```

> ⚠️ **Contrast must be measured, not asserted.** The `14.2:1` figure that appears in the original design report was never verified. Neon accents on a charcoal ground are the risk area — a saturated 1px border can fall below the 3:1 WCAG 1.4.11 requirement while still looking correct. Add a CI check that computes every token pair and fails below threshold.

---

## 🚀 Quick Start

### Prerequisites
- Node.js `>=22.22.2 <23 || >=24.0.0 <27`
- npm ≥ 10 — the tree ships `package-lock.json`; there is **no** pnpm, yarn, or bun lockfile
- Git

### Installation

```bash
# Clone this reports repository
git clone https://github.com/pasha01118/OmniRoute-.git
cd OmniRoute-

# This is a documentation repository — there is nothing to build.
# To work on OmniRoute itself:
git clone https://github.com/diegosouzapw/OmniRoute
```

---

## 🏗️ Implementation Phases

The original 39-day plan is preserved in [`03-development-roadmap.md`](./03-development-roadmap.md) as sequencing intent, but it needs **two extra phases and ~15 more days**. The corrected, gated plan is in [`10-ux-design.md`](./10-ux-design.md) §6.

| Phase | Duration | Focus | Gate |
|-------|----------|-------|------|
| **0. Truth** *(new)* | 1 day | Correct 8 stale numbers; align `AGENTS.md` | `check:docs-all` green |
| **1. Decisions** *(new)* | 1 day | ADR: typeface, icon set, motion, colour mode | Written ADR |
| **2. Tokens** | 3 days | Additive CSS-variable migration, contrast CI | AA automated pass |
| **3. Shell** | 2 days | `DashboardLayout` (139), `Breadcrumbs` (182), `Header` (286) | Visual regression |
| **4. Sidebar** | 4 days | `Sidebar.tsx` (758) — staged | Visual regression + keyboard |
| **5. Primitives** | 5 days | `Card` `Button` `Input` `Select` `Badge` `Avatar` `Modal` | Storybook coverage |
| **6. New primitives** | 4 days | `Icon`, `Table`*, `Tabs`, `Toast`, `animations.ts` | Component review |
| **7. Extract first** | 5 days | Decompose `HomePageClient` (1,385) and `providers/page.tsx` (1,951) | No behaviour change |
| **8. Core pages** | 6 days | Restyle the four measured pages | Per-page review |
| **9. Batch pages** | 15 days | ~100 remaining `page.tsx` via one pattern | Lint + a11y gates |
| **10. Motion** | 3 days | Framer Motion integration | `prefers-reduced-motion` verified |
| **11. QA** | 5 days | Contrast, keyboard, screen reader, cross-browser | Ratchet green |

<sub>*Audit `DataTable.tsx` before creating `Table.tsx` — it already exists.</sub>

**≈ 54 developer-days**, up from 39. The increase is driven by two measured facts the original plan missed: the two large pages need extraction *before* restyling, and the i18n gate is frozen at 100% across **43** locales, so every new string costs 43 translations.

---

## 🛠️ Tech Stack — as actually installed

| Category | Technologies |
|----------|--------------|
| **Frontend** | Next.js 16, React 19.2 (pinned exact), Tailwind CSS 4, Fumadocs UI |
| **State** | Zustand (persisted), React Hooks |
| **Charts** | Recharts 3 |
| **Typography** | **Inter** via `next/font/google` — *DM Sans is a target, not current* |
| **Icons** | **Material Symbols** `^0.45.2`, self-hosted — *Phosphor is an optional target* |
| **Animations** | *Framer Motion is a **new** dependency — not currently installed* |
| **Quality** | ESLint 9 (zero-warning), Prettier, TypeScript 6, Vitest 4, Playwright, c8, knip, Stryker |

---

## ♿ Accessibility Commitment

| Standard | Target | Implementation |
|----------|--------|----------------|
| **WCAG 2.1 AA** | Required | ≥ 4.5:1 text, ≥ 3:1 UI — **measured in CI**, not asserted |
| **Keyboard Navigation** | Full support | Logical tab order, focus rings, skip links (already in `layout.tsx:139-144`) |
| **Screen Readers** | Optimized | ARIA labels, live regions, semantic HTML |
| **Reduced Motion** | Respected | `prefers-reduced-motion` guards as a **blocking gate** |
| **Color Blind Safe** | Tested | Deuteranopia / Protanopia / Tritanopia |
| **i18n** | 100% UI coverage | Across all **43** locales (existing ratchet) |

---

## 🧪 Quality Gates

```bash
# These are OmniRoute's own commands (not this docs repo)
npm run check            # lint + test
npm run lint             # ESLint, zero warnings tolerated
npm run typecheck:core   # TypeScript
npm run test:unit        # Unit tests
npm run test:vitest      # Vitest
npm run test:e2e         # Playwright
npm run check:docs-all   # Documentation validation
npm run dashboard-typecheck  # MANDATORY — see below
```

> ⚠️ **`npm run build` succeeding is not evidence of type safety.** `next.config.mjs:296` sets `typescript.ignoreBuildErrors: true`, and there are frozen typecheck-error baselines for both the dashboard and `open-sse`. The `dashboard-typecheck` gate is the compensating control and must be run.

---

## 🤝 Contributing

### Branch Strategy
```bash
git checkout -b feat/your-feature-name
git commit -m "feat(scope): description"
```

### PR Requirements
- [ ] All CI gates pass (`npm run check`)
- [ ] TypeScript compiles (`npm run typecheck:core` **and** `npm run dashboard-typecheck`)
- [ ] Tests pass (`npm run test:unit && npm run test:vitest`)
- [ ] i18n coverage still 100% across 43 locales
- [ ] Visual regression snapshots updated
- [ ] Accessibility audit passes
- [ ] Documentation updated

---

## 📄 License

MIT License — see [LICENSE](LICENSE) for details.

> **Correction applied.** This repository previously shipped an **Apache-2.0** `LICENSE` file (11,357 bytes) while this README declared MIT, and OmniRoute itself is MIT (`package.json:77`, `LICENSE` 1,069 bytes, © 2026 diegosouzapw). The file has been corrected to the MIT text used upstream so it matches both the declaration and the source project. **Please confirm this is the intended licence** — changing licence terms is an owner decision, not a documentation fix.

---

## 🙏 Acknowledgments

- **OmniRoute Team** — Building the future of AI routing
- **Material Symbols** — the icon set OmniRoute actually ships, self-hosted
- **Inter** — the typeface OmniRoute actually loads via `next/font`
- **Tailwind CSS** — Utility-first styling at scale
- *Planned:* Framer Motion (animation), and optionally Phosphor Icons and DM Sans

---

<div align="center">

---

**Every number in this report set was measured, not copied.**

*Verified against OmniRoute `5764027` · v3.8.50*

</div>
