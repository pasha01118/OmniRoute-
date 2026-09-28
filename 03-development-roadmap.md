# OmniRoute Dashboard Redesign - Development Roadmap

> ## ⚠️ Verification Notice — the estimates below are optimistic by ~40%
>
> This roadmap was written before the OmniRoute tree was measured. It is **directionally correct** but **arithmetically wrong** in several places, and two of its targets do not exist.
>
> | This roadmap assumes | Measured reality | Correction |
> |---|---|---|
> | 39 dev-days total | — | **≈ 54 dev-days** once the 43-locale translation cost and the two page extractions are included |
> | `HomePageClient.tsx` ~400 lines | **1,385 lines** | **3.5× under.** Extract components *before* restyling. |
> | `providers/page.tsx` ~200 lines | **1,951 lines** | **9.75× under.** Same. |
> | `settings/page.tsx` ~200 lines | **33 lines** | 6× over. Trivial. |
> | `Sidebar.tsx` 758 lines (major restyle) | **758 lines — correct** | Keep. |
> | DM Sans + Phosphor as Day-2/3 tasks | Neither is installed | These are **new dependencies**, not configuration. See [`10-ux-design.md`](./10-ux-design.md) §2. |
> | 60 files modified | **114** dashboard `page.tsx` + 135 shared components | Scope is larger than estimated. |
> | Zero translation cost | i18n coverage gate at **100%** across **43** locales | **43 translations per new string.** Unbudgeted. |
> | `resilience/page.tsx` | **Does not exist** → `resilience/connections/page.tsx` | Correct the path (Batch 4). |
> | `system/page.tsx` | **Does not exist** → `system/1proxy/`, `system/proxy/`, `system/mitm-proxy/` | Correct the path (Batch 5). |
> | `Table.tsx` is new | **`DataTable.tsx` already exists** | Audit before creating. |
> | 7 phases | 8 in the corrected plan | A **Phase 0 (truth)** and **Phase 1 (decisions)** are required first. |
>
> The other 19 Phase 6 page targets **do** exist and need no correction.
>
> **The corrected, gated phase plan is in [`10-ux-design.md`](./10-ux-design.md) §6 and [`00-comprehensive-master-plan.md`](./00-comprehensive-master-plan.md) §4.4.** The day-by-day structure below is preserved as the *sequencing intent* that plan implements.

---

## Overview

| Metric | Value |
|--------|-------|
| **Total Duration** | 39 developer-days (6 weeks) — **rebudgeted to ≈54** |
| **Parallelizable** | 3 developers → ~13 days (original) / **~18 days** (corrected) |
| **Phases** | 7 (original) / **8** (corrected, incl. truth + decisions) |
| **Files Modified** | ~60 (original) / **~100+** (corrected) |
| **Risk Level** | Medium (original) / **Medium-High** (restyling two ~1,500-line files without extracting first) |

---

## Phase Overview (39 Days)

```
WEEK 1 (Days 1-3)     ████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │ Phase 1: Foundation
WEEK 1-2 (Days 4-6)   ░░████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │ Phase 2: Layout Shell
WEEK 2 (Days 7-10)    ░░░░████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │ Phase 3: Core Components
WEEK 3 (Days 11-15)   ░░░░░░██████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │ Phase 4: Core Pages
WEEK 4 (Days 16-19)   ░░░░░░░░████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │ Phase 5: Animation System
WEEK 5-6 (Days 20-34) ░░░░░░░░░░██████████████████████░░░░░░░░░░░░░░░░░░░  │ Phase 6: Batch Pages (25)
WEEK 6 (Days 35-39)   ░░░░░░░░░░░░██████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │ Phase 7: QA & Polish
```

> **Before starting, add two phases that this plan omits.** **Phase 0 — Truth:** correct the eight stale upstream numbers and the `AGENTS.md` breaker-threshold contradiction (see [`11-delivery-devops.md`](./11-delivery-devops.md) §8). **Phase 1 — Decisions:** write an ADR for typeface, icon set, motion, and whether dark-only is the right call. Three of those four are currently mis-stated as if already done.

---

## Phase 1: Foundation & Design Tokens (Days 1-3)

### Day 1: globals.css Additive Token Migration
- **File**: `src/app/globals.css` (590 lines)
- **Task**: Introduce 40+ new CSS custom properties **alongside** the existing `:root` / `.dark` blocks
- **Deliverable**: Complete color system, shadows, radius, typography, animation tokens
- **Validation**: `npm run dev` → verify no build errors, dark mode renders
- ⚠️ **Do not replace the blocks wholesale.** All 135 shared components reference the current custom-property names. Add the new names, migrate component-by-component, then remove the old names once `grep` finds zero references.

### Day 2: Tailwind Extensions + Icon Wrapper
- **File**: `src/app/globals.css` (`@theme inline` section)
- **Task**: Add custom colors, shadows, animations to Tailwind
- **New File**: `src/shared/components/Icon.tsx` (wrapper component)
- ⚠️ **Phosphor is optional.** The app already ships **Material Symbols** `^0.45.2`, self-hosted via `@import` in `globals.css`. Recommendation: write `Icon.tsx` against Material Symbols and skip `npm install @phosphor-icons/react` entirely. Adopting Phosphor means a new dependency, a dual-set migration across 135 components, and a full accessible-name re-audit.

### Day 3: Typeface Decision + Icon Wrapper
- **File**: `src/app/globals.css`
- **Task**: Decide the typeface, then update `--font-sans` accordingly
- **File**: `src/shared/components/Icon.tsx`
- **Task**: Create a consistent icon wrapper with size/color props
- **Validation**: Visual check of icons at 16/20/24/28px
- ⚠️ **The app currently loads Inter** via `next/font/google` (`src/app/layout.tsx:1,14-17,138` — the only `next/font` call in the repo). DM Sans is **not installed**. Recommendation: keep Inter unless there is a hard brand requirement. If you switch, budget a re-audit of every fixed-height element and the whole type scale — it is not a config toggle.

---

## Phase 2: Layout Shell (Days 4-6)

### Day 4: DashboardLayout.tsx
- **File**: `src/shared/components/layouts/DashboardLayout.tsx` (139 lines)
- **Changes**:
  - Enhanced metallic grid wallpaper (`body::before`)
  - Glassmorphism main content wrapper
  - Neon border frame on main content area
  - `AnimatePresence` for page transitions

### Day 5: Sidebar.tsx (758 lines - Major Restyle)
- **File**: `src/shared/components/Sidebar.tsx` (758 lines — measured; the estimate is correct)
- **Changes**:
  - Background: `bg-gradient-to-b from-metallic-elevated to-metallic-bg`
  - Active item: `border-l-3 border-neon-cyan bg-neon-cyan/5 text-neon-ivory shadow-neon-glow`
  - Icons: `text-neon-ivory group-hover:text-neon-cyan`
  - Section titles: `text-neon-ivory-subtle uppercase tracking-wider text-[10px]`
  - Collapsed tooltip: glassmorphism + neon border
  - Framer Motion `AnimatePresence` for collapse/expand
- ⚠️ This is the largest shared component in the redesign and it is state-heavy. **Restyle it in stages behind a visual-regression gate** rather than in one pass.

### Day 6: Header.tsx + Breadcrumbs.tsx
- **Files**: `src/shared/components/Header.tsx` (286), `src/shared/components/Breadcrumbs.tsx` (182)
- **Header Changes**:
  - Glassmorphism: `bg-metallic-elevated/80 backdrop-blur-xl`
  - Neon bottom accent: `bg-gradient-to-r from-transparent via-neon-cyan/50 to-transparent`
  - Title text: `text-neon-ivory`
  - Focus rings: `focus-visible:ring-4 focus-visible:ring-neon-cyan`
- **Breadcrumbs Changes**:
  - Text: `text-neon-ivory-dim`
  - Separators: `text-metallic-border`
  - Current: `text-neon-ivory`

---

## Phase 3: Core Component Library (Days 7-10)

| Day | Component | Exists? | Key Neon Treatment |
|-----|-----------|---------|-------------------|
| 7 | `Card.tsx` (141) | yes | `bg-metallic-card border-metallic-border hover:border-neon-cyan hover:shadow-neon-glow` |
| 7 | `Button.tsx` (88) | yes | Primary: gradient + glow; Secondary: border + text; Ghost: text only |
| 8 | `Input.tsx` (157) | yes | `focus:border-neon-cyan focus:ring-neon-cyan/30 text-neon-ivory placeholder:text-neon-ivory-subtle` |
| 8 | `Select.tsx` (115) | yes | Dropdown: glassmorphism + neon border |
| 9 | `Table.tsx` | **no** | ⚠️ `DataTable.tsx` already exists — audit it first |
| 9 | `Tabs.tsx` | **no** | ⚠️ only `docs/Tabs.tsx` (docs-scoped) exists |
| 10 | `Modal.tsx` (267) | yes | Overlay: `bg-black/60 backdrop-blur-sm`; Panel: glassmorphism + neon border |
| 10 | `Badge.tsx` (68) | yes | Semantic: `text-neon-{color} bg-neon-{color}/10` |
| 10 | `Avatar.tsx` (81) | yes | Ring: `border-2 border-neon-cyan/50` |
| 10 | `Toast.tsx` | **no** | ⚠️ `NotificationToast.tsx` (208) exists — extend or rename |

### Component Migration Checklist
- [ ] All `className` props accept override
- [ ] `variant` prop for semantic styling
- [ ] `size` prop where applicable
- [ ] Forward ref for DOM access
- [ ] TypeScript interfaces updated
- [ ] **43 translations per new user-facing string** (i18n UI coverage gate is frozen at 100%)
- [ ] **Adopt, do not duplicate:** `DataTable` · `ColumnToggle` · `FilterBar` · `EmptyState` · `ErrorPageScaffold` · `Loading` · `InfoTooltip` · `CommandPalette` · `Checkbox` · `Collapsible` · `NavigationProgress` · `DegradationBadge`

---

## Phase 4: Core Pages (Days 11-15)

> ⚠️ **Extract before restyling.** `HomePageClient.tsx` is **1,385 lines** and `providers/page.tsx` is **1,951 lines** — 3.5× and 9.75× the estimates in the original plan. Decomposing them into components first costs ~5 days and is the only way the restyle is reviewable. A single-pass restyle of a 1,951-line file will not survive code review.

### Day 11-12: HomePageClient.tsx (1,385 lines — extract first)
```tsx
// Hero Section
<Card className="bg-gradient-to-br from-metallic-elevated to-metallic-card 
  border-neon-cyan/20 p-8 text-center">
  <OmniRouteLogo className="text-neon-cyan" size={48} />
  <h1 className="text-4xl font-bold text-neon-ivory">Unified AI Router</h1>
  <p className="text-neon-ivory-dim">338 Providers • Auto-Fallback • Zero Config</p>
  <div className="flex gap-3 justify-center mt-6">
    <Button variant="primary">Get Started</Button>
    <Button variant="secondary">View Docs</Button>
  </div>
</Card>

// Metrics Row (4 cards, staggered)
<motion.div variants={staggerContainer} className="grid gap-4 md:grid-cols-2 lg:grid-cols-4">
  <MetricCard icon="token" value="1.2M" label="Requests" accent="neon-emerald" />
  <MetricCard icon="cloud" value="338" label="Providers" accent="neon-cyan" />
  <MetricCard icon="model" value="4,847" label="Models" accent="neon-violet" />
  <MetricCard icon="speed" value="42ms" label="Latency" accent="neon-amber" />
</motion.div>

// Provider Topology (Live Pulse)
<HomeProviderTopologySection enabled={true} />

// Quick Start (4 steps)
<Card className="bg-metallic-card border-neon-cyan/20">
  <ol className="grid gap-4 md:grid-cols-2">
    {steps.map((step, i) => (
      <motion.li key={step.title} variants={cardReveal}>
        <div className="flex gap-4">
          <div className={`size-12 rounded-xl bg-${step.accent}/10 
            border border-${step.accent}/30 flex items-center justify-center`}>
            <Icon name={step.icon} className={`text-${step.accent}`} size={24} />
          </div>
          <div>
            <h4 className="text-neon-ivory font-semibold">{step.title}</h4>
            <p className="text-neon-ivory-dim">{step.desc}</p>
          </div>
        </div>
      </motion.li>
    ))}
  </ol>
</Card>
```

### Day 13: Providers Page (1,951 lines — extract first)
- Card grid with `ProviderOverviewCard` using new `Card` + `Badge` + `ProviderIcon`
- Neon status dots: connected=emerald, error=coral, idle=amber
- Provider detail drawer with glassmorphism panel
- ⚠️ `ProviderIcon.tsx` currently renders **operator-supplied remote SVG URLs** under a `no-img-element` lint suppression. Confirm the URL is operator-only before restyling, or you may propagate a tracking-pixel vector. See [`10-ux-design.md`](./10-ux-design.md) §4.2.

### Day 14: Analytics Pages (155 lines)
- Recharts with neon data lines: `stroke="var(--neon-cyan)"`
- Metric cards with stagger entrance
- `Table` for data grids (after the `DataTable` audit)

### Day 15: Settings Pages (33 lines — trivial)
- Form sections in glassmorphism `Card`
- All `Input`/`Select` with neon focus
- Toggles with neon thumb: `bg-neon-cyan`
- `Tabs` with neon indicator

---

## Phase 5: Animation System (Days 16-19)

### Day 16: Framer Motion Install + Utilities
```bash
npm install framer-motion
```
**New File**: `src/shared/lib/animations.ts` (see [`02-design-system.md`](./02-design-system.md) §5.1 for full content)

⚠️ `framer-motion` is **not currently a declared dependency** — this is a genuine addition, not a config change. Verify Motion/React 19.2.8 compatibility (React is pinned exact) and check the frozen bundle-size ratchet in `config/quality/quality-baseline.json` (currently 8,045) so the addition is visible.

### Day 17: Stagger + Scroll Reveal
- Card grids: `staggerContainer` + `cardReveal`
- Sections: `scrollReveal` trigger at 85% viewport
- Lists: 50ms stagger, 400ms duration

### Day 18: Magnetic + Pulse + Page Transitions
- Buttons: `magneticButton` (scale 1.02 hover / 0.98 tap)
- Live indicators: `neonPulse` (2s, infinite, mirror)
- Pages: `pageTransition` in `DashboardLayout`

### Day 19: Reduced Motion + Performance
- All animations wrapped in `useReducedMotion()`
- `matchMedia('(prefers-reduced-motion: reduce)')` listener
- Performance audit: `will-change` on animated properties
- Bundle size check: `npm run build && npm run check:bundle-size`
- ⚠️ `prefers-reduced-motion` should be a **blocking gate**, not a convention. The dashboard has many live-updating regions (`sseMerger`, the WS sidecar), and animating those causes layout thrash.

---

## Phase 6: Remaining Pages - Batch Processing (Days 20-34)

### Batch 1 (Days 20-22): Data & Monitoring
| Page | Exists? | Focus |
|------|---------|-------|
| `logs/page.tsx` | yes | Data tables, filter bars, level badges |
| `costs/page.tsx` | yes | Budget cards, pricing tables, quota widgets |
| `cache/page.tsx` | yes | Memory cards, trends charts, idempotency layer |
| `quota/page.tsx` | yes | Usage bars, limit configs, pool status |

### Batch 2 (Days 23-25): Configuration & Protocols
| Page | Exists? | Focus |
|------|---------|-------|
| `combos/page.tsx` | yes | Builder, weight bars, intelligent panel |
| `api-manager/page.tsx` | yes | Key filters, compression toggles, scopes |
| `mcp/page.tsx` | yes | Tool registry (108 tools), server config, audit |
| `a2a/page.tsx` | yes | Agent card, skill registry (6 skills), task execution |
| `endpoint/page.tsx` | yes | Endpoint registry, health, components |

### Batch 3 (Days 26-28): Interactive & Intelligence
| Page | Exists? | Focus |
|------|---------|-------|
| `playground/page.tsx` | yes | Chat UI, model selector, parameter controls |
| `translator/page.tsx` | yes | Studio, live preview, format conversion |
| `search-tools/page.tsx` | yes | Tool registry, parameter schemas |
| `memory/page.tsx` | yes | Conversation list, vector viz, FTS5 stats |

### Batch 4 (Days 29-31): Agents & Resilience
| Page | Exists? | Focus |
|------|---------|-------|
| `agent-skills/page.tsx` | yes | Marketplace cards, coverage bars, MCP/A2A links |
| `cli-agents/page.tsx` | yes | Agent cards, status badges, detail views |
| `health/page.tsx` | yes | System health, provider status, metrics |
| `resilience/page.tsx` | **NO** | ⚠️ The real route is **`resilience/connections/page.tsx`** |

### Batch 5 (Days 32-34): Meta & System
| Page | Exists? | Focus |
|------|---------|-------|
| `profile/page.tsx` | yes | User settings, appearance, security |
| `onboarding/page.tsx` | yes | Wizard steps, progress, validation |
| `changelog/page.tsx` | yes | Timeline, news viewer, version cards |
| `system/page.tsx` | **NO** | ⚠️ The real routes are **`system/1proxy/page.tsx`**, **`system/proxy/page.tsx`**, and **`system/mitm-proxy/page.tsx`** (the last is a 40-line redirect stub to `/dashboard/tools/agent-bridge` with a 2.5 s auto-`router.replace`) |

### Per-Page Pattern (Apply to All)
```tsx
// 1. Replace containers
<Card className="bg-metallic-card border-metallic-border 
  hover:border-neon-cyan hover:shadow-neon-glow transition-all duration-300">

// 2. Forms
<Input className="focus:border-neon-cyan focus:ring-neon-cyan/30" />
<Select className="focus:border-neon-cyan" />
<Button variant="primary|secondary|ghost" />

// 3. Tables
<Table data={data} columns={columns} />

// 4. Status indicators
<Badge variant="success|warning|error|info" />

// 5. Add animations
<motion.div variants={cardReveal} />
<motion.div variants={scrollReveal} />
```

---

## Phase 7: QA & Polish (Days 35-39)

### Day 35: Contrast & Focus Audit
- [ ] Add a CI contrast check that **computes** every token pair — do not assert a ratio
- [ ] Neon Ivory `#FFF8E7` on Charcoal `#0A0E14` ≥ 4.5:1 (verify, do not assume `14.2:1`)
- [ ] All borders ≥ 3:1 contrast **at 1px width** — neon on charcoal is the risk area
- [ ] Focus rings: 4px neon cyan on ALL interactive (buttons, links, inputs, tabs)
- [ ] Skip links functional

### Day 36: Reduced Motion + Color Blind
- [ ] `prefers-reduced-motion`: all animations instant
- [ ] Deuteranopia simulation (Coblis/Color Oracle)
- [ ] Protanopia simulation
- [ ] Tritanopia check (neon cyan/amber distinguishable)

### Day 37: Mobile Viewport Testing
| Breakpoint | Tests |
|------------|-------|
| 375px (iPhone SE) | Sidebar collapse, header layout, card stacking |
| 768px (iPad) | Sidebar toggle, grid 2-col, table scroll |
| 1024px (Desktop) | Full sidebar, 4-col grids, hover states |
| 1440px (Wide) | Max-width 3840px cap, centered content |

### Day 38: Performance + Cross-Browser
- [ ] Bundle: `< 250KB gz` (core), `< 500KB` (full dashboard)
- [ ] LCP `< 2.5s`, TTI `< 3.5s`, CLS `< 0.1`
- [ ] 60fps scrolling on 1000+ item virtualized lists
- [ ] Chrome 120+, Firefox 115+, Safari 17+, Edge 120+

### Day 39: Final Polish + PR Prep
- [ ] E2E critical paths: login → home → providers → settings
- [ ] Visual regression: Percy/Chromatic snapshots
- [ ] Documentation updates: `CHANGELOG.md`, `docs/`
- [ ] PR description with screenshots + testing notes
- [ ] Branch: `feat/charcoal-neon-redesign`

---

## File Inventory Summary

| Phase | Files | Est. Count |
|-------|-------|------------|
| 1 | `globals.css`, `Icon.tsx` | 2 |
| 2 | `DashboardLayout.tsx`, `Sidebar.tsx`, `Header.tsx`, `Breadcrumbs.tsx` | 4 |
| 3 | `Card.tsx`, `Button.tsx`, `Input.tsx`, `Select.tsx`, `Table.tsx`, `Tabs.tsx`, `Modal.tsx`, `Badge.tsx`, `Avatar.tsx`, `Toast.tsx` | 10 |
| 4 | `HomePageClient.tsx`, Providers, Analytics, Settings (+ 2 extractions) | ~12 + **2 new** |
| 5 | `animations.ts` | 1 |
| 6 | 25 dashboard pages (**+3 in `system/`, +1 in `resilience/`**) | 25 |
| 7 | Tests, docs, config | ~6 |
| **Total** | | **~60 original / ~100+ corrected** |

---

## Risk Mitigation

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Fumadocs component conflicts | Medium | High | Wrapper components extending Fumadocs with neon overrides |
| Icon-set migration | Medium | Medium | ⚠️ **Keep Material Symbols** and add an `Icon.tsx` seam rather than dual-set migration |
| Bundle size increase | Low | Medium | Code-split animations, CSS for simple transitions; check the frozen 8,045 ratchet |
| 114-page regression | **High** | High | Visual regression tests + phased PRs behind a feature flag with per-page opt-in |
| Dark mode only constraint | **Medium** | **Medium** | ⚠️ This **removes** existing light + system modes. Document as an intentional decision with its accessibility consequences |
| Unreviewable large-file restyle | **High** | High | ⚠️ Extract `HomePageClient` (1,385) and `providers/page.tsx` (1,951) **before** restyling |
| i18n coverage gate breakage | **High** | High | ⚠️ 43 translations per new string; the gate is frozen at 100% and blocks the build |

---

## Success Metrics

| Metric | Target |
|--------|--------|
| **Visual Consistency** | 100% pages use design tokens |
| **Contrast Ratio** | ≥ 4.5:1 (text), ≥ 3:1 (borders) — **measured in CI, not asserted** |
| **Animation Performance** | 60fps on all transitions |
| **Bundle Size** | Core < 250KB gz |
| **Accessibility** | WCAG 2.1 AA compliant |
| **i18n** | 100% UI coverage across all **43** locales |
| **Developer Experience** | Storybook stories for all components |