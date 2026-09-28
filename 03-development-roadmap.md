# OmniRoute Dashboard Redesign - Development Roadmap

## Overview

| Metric | Value |
|--------|-------|
| **Total Duration** | 39 developer-days (6 weeks) |
| **Parallelizable** | 3 developers → ~13 days |
| **Phases** | 7 |
| **Files Modified** | ~60 |
| **Risk Level** | Medium (phased mitigates) |

---

## Phase Overview (39 Days)

```
WEEK 1 (Days 1-3)     ████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │ Phase 1: Foundation
WEEK 1-2 (Days 4-6)   ░░████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │ Phase 2: Layout Shell
WEEK 2 (Days 7-10)    ░░░░████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │ Phase 3: Core Components
WEEK 3 (Days 11-15)   ░░░░░░██████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │ Phase 4: Core Pages
WEEK 4 (Days 16-19)   ░░░░░░░░████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │ Phase 5: Animation System
WEEK 5-6 (Days 20-34) ░░░░░░░░░░██████████████████████░░░░░░░░░░░░░░░░░░░░  │ Phase 6: Batch Pages (25)
WEEK 6 (Days 35-39)   ░░░░░░░░░░░░██████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │ Phase 7: QA & Polish
```

---

## Phase 1: Foundation & Design Tokens (Days 1-3)

### Day 1: globals.css Complete Overhaul
- **File**: `src/app/globals.css`
- **Task**: Replace entire `:root` and `.dark` blocks with 40+ new CSS custom properties
- **Deliverable**: Complete color system, shadows, radius, typography, animation tokens
- **Validation**: `npm run dev` → verify no build errors, dark mode renders

### Day 2: Tailwind Extensions + Phosphor Install
- **File**: `src/app/globals.css` (`@theme inline` section)
- **Task**: Add custom colors, shadows, animations to Tailwind
- **Command**: `npm install @phosphor-icons/react`
- **New File**: `src/shared/components/Icon.tsx` (wrapper component)

### Day 3: DM Sans Typography + Icon Wrapper
- **File**: `src/app/globals.css`
- **Task**: Add `@import` for DM Sans, update `--font-sans`
- **File**: `src/shared/components/Icon.tsx`
- **Task**: Create consistent Phosphor wrapper with size/color props
- **Validation**: Storybook/visual check of icons at 16/20/24/28px

---

## Phase 2: Layout Shell (Days 4-6)

### Day 4: DashboardLayout.tsx
- **File**: `src/shared/components/layouts/DashboardLayout.tsx`
- **Changes**:
  - Enhanced metallic grid wallpaper (`body::before`)
  - Glassmorphism main content wrapper
  - Neon border frame on main content area
  - `AnimatePresence` for page transitions

### Day 5: Sidebar.tsx (758 lines - Major Restyle)
- **File**: `src/shared/components/Sidebar.tsx`
- **Changes**:
  - Background: `bg-gradient-to-b from-metallic-elevated to-metallic-bg`
  - Active item: `border-l-3 border-neon-cyan bg-neon-cyan/5 text-neon-ivory shadow-neon-glow`
  - Icons: `text-neon-ivory group-hover:text-neon-cyan`
  - Section titles: `text-neon-ivory-subtle uppercase tracking-wider text-[10px]`
  - Collapsed tooltip: glassmorphism + neon border
  - Framer Motion `AnimatePresence` for collapse/expand

### Day 6: Header.tsx + Breadcrumbs.tsx
- **Files**: `src/shared/components/Header.tsx`, `src/shared/components/Breadcrumbs.tsx`
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

| Day | Component | Key Neon Treatment |
|-----|-----------|-------------------|
| 7 | `Card.tsx` | `bg-metallic-card border-metallic-border hover:border-neon-cyan hover:shadow-neon-glow` |
| 7 | `Button.tsx` | Primary: gradient + glow; Secondary: border + text; Ghost: text only |
| 8 | `Input.tsx` | `focus:border-neon-cyan focus:ring-neon-cyan/30 text-neon-ivory placeholder:text-neon-ivory-subtle` |
| 8 | `Select.tsx` | Dropdown: glassmorphism + neon border |
| 9 | `Table.tsx` (NEW) | Header: `bg-metallic-mid border-b border-neon-cyan`; Hover: `border-l-3 border-neon-cyan` |
| 9 | `Tabs.tsx` | Indicator: `bg-gradient-to-r from-neon-cyan to-neon-violet h-1` |
| 10 | `Modal.tsx` | Overlay: `bg-black/60 backdrop-blur-sm`; Panel: glassmorphism + neon border |
| 10 | `Badge.tsx` | Semantic: `text-neon-{color} bg-neon-{color}/10` |
| 10 | `Avatar.tsx` | Ring: `border-2 border-neon-cyan/50` |
| 10 | `Toast.tsx` | Left border: `border-l-4 border-neon-{color}` |

### Component Migration Checklist
- [ ] All `className` props accept override
- [ ] `variant` prop for semantic styling
- [ ] `size` prop where applicable
- [ ] Forward ref for DOM access
- [ ] TypeScript interfaces updated

---

## Phase 4: Core Pages (Days 11-15)

### Day 11-12: HomePageClient.tsx
```tsx
// Hero Section
<Card className="bg-gradient-to-br from-metallic-elevated to-metallic-card 
  border-neon-cyan/20 p-8 text-center">
  <OmniRouteLogo className="text-neon-cyan" size={48} />
  <h1 className="text-4xl font-bold text-neon-ivory">Unified AI Router</h1>
  <p className="text-neon-ivory-dim">291 Providers • Auto-Fallback • Zero Config</p>
  <div className="flex gap-3 justify-center mt-6">
    <Button variant="primary">Get Started</Button>
    <Button variant="secondary">View Docs</Button>
  </div>
</Card>

// Metrics Row (4 cards, staggered)
<motion.div variants={staggerContainer} className="grid gap-4 md:grid-cols-2 lg:grid-cols-4">
  <MetricCard icon="token" value="1.2M" label="Requests" accent="neon-emerald" />
  <MetricCard icon="cloud" value="291" label="Providers" accent="neon-cyan" />
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

### Day 13: Providers Page
- Card grid with `ProviderOverviewCard` using new `Card` + `Badge` + `ProviderIcon`
- Neon status dots: connected=emerald, error=coral, idle=amber
- Provider detail drawer with glassmorphism panel

### Day 14: Analytics Pages
- Recharts with neon data lines: `stroke="var(--neon-cyan)"`
- Metric cards with stagger entrance
- New `Table` component for data grids

### Day 15: Settings Pages
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
**New File**: `src/shared/lib/animations.ts` (see Report 2 for full content)

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

---

## Phase 6: Remaining Pages - Batch Processing (Days 20-34)

### Batch 1 (Days 20-22): Data & Monitoring
| Page | Focus |
|------|-------|
| `logs/page.tsx` | Data tables, filter bars, level badges |
| `costs/page.tsx` | Budget cards, pricing tables, quota widgets |
| `cache/page.tsx` | Memory cards, trends charts, idempotency layer |
| `quota/page.tsx` | Usage bars, limit configs, pool status |

### Batch 2 (Days 23-25): Configuration & Protocols
| Page | Focus |
|------|-------|
| `combos/page.tsx` | Builder, weight bars, intelligent panel |
| `api-manager/page.tsx` | Key filters, compression toggles, scopes |
| `mcp/page.tsx` | Tool registry, server config, audit |
| `a2a/page.tsx` | Agent card, skill registry, task execution |
| `endpoint/page.tsx` | Endpoint registry, health, components |

### Batch 3 (Days 26-28): Interactive & Intelligence
| Page | Focus |
|------|-------|
| `playground/page.tsx` | Chat UI, model selector, parameter controls |
| `translator/page.tsx` | Studio, live preview, format conversion |
| `search-tools/page.tsx` | Tool registry, parameter schemas |
| `memory/page.tsx` | Conversation list, vector viz, FTS5 stats |

### Batch 4 (Days 29-31): Agents & Resilience
| Page | Focus |
|------|-------|
| `agent-skills/page.tsx` | Marketplace cards, coverage bars, MCP/A2A links |
| `cli-agents/page.tsx` | Agent cards, status badges, detail views |
| `health/page.tsx` | System health, provider status, metrics |
| `resilience/page.tsx` | Circuit breakers, cooldowns, model lockouts |

### Batch 5 (Days 32-34): Meta & System
| Page | Focus |
|------|-------|
| `profile/page.tsx` | User settings, appearance, security |
| `onboarding/page.tsx` | Wizard steps, progress, validation |
| `changelog/page.tsx` | Timeline, news viewer, version cards |
| `system/page.tsx` | Admin controls, bootstrap, version manager |

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
- [ ] Neon Ivory (#FFF8E7) on Charcoal (#0A0E14) = 14.2:1 ✓
- [ ] All borders ≥ 3:1 contrast
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
| 4 | `HomePageClient.tsx`, Providers, Analytics (3), Settings (6) | ~12 |
| 5 | `animations.ts` | 1 |
| 6 | 25 dashboard pages | 25 |
| 7 | Tests, docs, config | ~6 |
| **Total** | | **~60 files** |

---

## Risk Mitigation

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Fumadocs component conflicts | Medium | High | Wrapper components extending Fumadocs with neon overrides |
| Material Symbols dependencies | Medium | Medium | Gradual migration - keep both during transition |
| Bundle size increase | Low | Medium | Code-split animations, CSS for simple transitions |
| 50+ page regression | High | High | Visual regression tests + phased PRs |
| Dark mode only constraint | Low | Low | Document as intentional design decision |

---

## Success Metrics

| Metric | Target |
|--------|--------|
| **Visual Consistency** | 100% pages use design tokens |
| **Contrast Ratio** | ≥ 14.2:1 (text), ≥ 3:1 (borders) |
| **Animation Performance** | 60fps on all transitions |
| **Bundle Size** | Core < 250KB gz |
| **Accessibility** | WCAG 2.1 AA compliant |
| **Developer Experience** | Storybook stories for all components |