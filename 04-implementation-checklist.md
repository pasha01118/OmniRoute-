# OmniRoute Dashboard Redesign - Implementation Checklist

> ## ⚠️ Verification Notice — the file inventory in §8 is partly wrong
>
> The checklist QA criteria in §1–§7 and §9–§12 remain valid and should be used as written. **§8 (File Inventory) required correction** after measuring the OmniRoute tree at commit `5764027`.
>
> ### §8 corrections
>
> | Entry | This checklist says | Measured reality |
> |---|---|---|
> | `src/shared/components/Icon.tsx` | New | **Correct** — does not exist. Must wrap Material Symbols (the installed set), not Phosphor. |
> | `src/shared/components/Table.tsx` | New | **Partly wrong** — **`DataTable.tsx` already exists.** Audit it, then extend or replace. Creating a parallel `Table` guarantees the inconsistency this project is meant to remove. |
> | `src/shared/components/Tabs.tsx` | Modified | **Wrong** — does not exist. Only `src/shared/components/docs/Tabs.tsx` (docs-scoped) exists. |
> | `src/shared/components/Toast.tsx` | Modified | **Wrong** — does not exist. Only `NotificationToast.tsx` (208 lines) exists. |
> | `src/shared/lib/animations.ts` | New | **Correct** — the file does not exist, and `src/shared/lib/` is a valid directory. |
> | `globals.css` ~600 | Modified (rewrite) | **590 lines measured.** Also: rewrite **additively**, not big-bang — 135 components depend on the current custom-property names. |
> | `HomePageClient.tsx` ~400 | Modified (major) | **1,385 lines.** Extract components before restyling. |
> | `providers/page.tsx` ~200 | Modified | **1,951 lines.** Same — this is the single largest file in scope. |
> | `settings/page.tsx` ~200 | Modified | **33 lines.** Trivial. |
> | `Sidebar.tsx` ~300 | Modified (major) | 758 total; "~300" was a delta estimate. Ambiguous — restate as "758 lines, staged". |
> | `Header.tsx` ~100 | Modified | 286 total. |
> | `Breadcrumbs.tsx` ~30 | Modified | 182 total. |
> | — | not listed | **Already exists, adopt not duplicate:** `DataTable` · `ColumnToggle` · `FilterBar` · `EmptyState` · `ErrorPageScaffold` · `Loading` · `InfoTooltip` · `CommandPalette` · `Checkbox` · `Collapsible` · `NavigationProgress` |
>
> ### §9 (New Dependencies) corrections
>
> | Dependency | Status |
> |---|---|
> | `@phosphor-icons/react ^2.1.0` | **Optional, not required.** The app ships Material Symbols `^0.45.2` self-hosted. Adopting Phosphor is a 135-component migration plus a new dependency. Recommend keeping Material Symbols behind a thin `Icon.tsx` seam. |
> | `framer-motion ^11.0.0` | **Genuinely new — not installed.** Zero imports in `src/`. Verify Motion/React 19.2.8 compatibility and budget it in the bundle ratchet. Note the current major is well past 11. |
> | DM Sans | **Not a dependency — a `next/font` swap.** Currently Inter via `layout.tsx:1`. Keep Inter unless there is a hard brand requirement; if changed, the full 590-line type scale needs re-audit. |
>
> ### §6.1 (Testing) correction
>
> "Unit Tests (Jest/Node)" → the project runs **Vitest 4.1.7 + Node-native** runners. **Jest is not installed** and no script references it. §11's `npm run test:vitest` is correct; §6.1's heading is not.
>
> ### §11 (Pre-PR) correction
>
> `npm install` and the `npm run …` commands are **correct** — the tree has `package-lock.json` and no pnpm/yarn/bun lockfile. The project **README** says pnpm; that is the error, not this checklist. Also add: `npm run dashboard-typecheck` is **mandatory** — `next.config.mjs:296` sets `typescript.ignoreBuildErrors: true`, so a green build is **not** evidence of type safety.
>
> ### Missing acceptance criteria
>
> The checklist has no item for the **i18n coverage gate**, which is frozen at **100%** across **43** locales. Add: "every new user-facing string has 43 translations, and `npm run check:i18n-ui-coverage` passes." Without it, Phase 1 will ship untranslated UI.
>
> **Full analysis:** [`10-ux-design.md`](./10-ux-design.md) §3 and §7.

---

## 1. Visual Quality Checklist

### 1.1 Color & Theme
- [ ] **No pure white text** - All text uses Neon Ivory (`#FFF8E7`) variants
- [ ] **No emoji icons** - All icons from the icon set (Material Symbols, behind an `Icon.tsx` seam — see notice above)
- [ ] **Charcoal metallic backgrounds** - `#0A0E14` base with subtle gradients
- [ ] **Thin neon borders** - 1-2px on ALL interactive surfaces
- [ ] **Semantic neon colors** - Cyan (primary), Emerald (success), Amber (warning), Coral (error), Violet (special)
- [ ] **Glassmorphism consistency** - `backdrop-blur-xl` + `bg-metallic-card/70` + neon border

### 1.2 Component Consistency
- [ ] **Card**: Metallic bg + metallic border + neon hover glow
- [ ] **Button**: 3 variants (primary/secondary/ghost) with neon treatment
- [ ] **Input/Select**: Neon focus ring (4px cyan) + neon ivory text
- [ ] **Table**: Metallic header + zebra rows + neon left accent on hover
- [ ] **Tabs**: Neon gradient indicator line
- [ ] **Modal**: Glassmorphism panel + neon border frame
- [ ] **Badge**: Semantic neon text + 10% opacity bg
- [ ] **Toast**: 4px left border in semantic neon color
- [ ] **Avatar**: 2px neon cyan/50 ring

### 1.3 Layout Integrity
- [ ] **DashboardLayout**: Metallic grid wallpaper visible through transparent areas
- [ ] **Sidebar**: Gradient bg, neon active indicator (3px left), glassmorphism tooltip
- [ ] **Header**: Glassmorphism + neon bottom accent line
- [ ] **Breadcrumbs**: Neon ivory dim text, metallic separators
- [ ] **Max-width**: 3840px cap, centered beyond 4K

---

## 2. Interaction Checklist

### 2.1 Feedback & Timing
- [ ] **Pressed feedback** < 150ms on all buttons/links
- [ ] **Hover transitions** 150-300ms smooth
- [ ] **Stagger animations** 50ms delay, 400ms duration
- [ ] **Scroll reveals** trigger at 85% viewport
- [ ] **Page transitions** 400ms expo.out
- [ ] **Magnetic buttons** scale 1.02 hover / 0.98 tap

### 2.2 Touch & Accessibility Targets
- [ ] **Desktop touch targets** ≥ 32×32px
- [ ] **Mobile touch targets** ≥ 44×44pt
- [ ] **Focus rings** 4px neon cyan on ALL focusable elements
- [ ] **Skip links** functional (Sidebar, Header, Main)
- [ ] **Logical tab order** through all interactive elements

### 2.3 State Management
- [ ] **Loading states** skeleton screens (CardSkeleton)
- [ ] **Empty states** illustrative + action button
- [ ] **Error states** inline + toast notification
- [ ] **Disabled states** 50% opacity + not-interactive
- [ ] **Live regions** for dynamic updates (toast, topology)

---

## 3. Accessibility Checklist (WCAG 2.1 AA)

### 3.1 Contrast & Color
- [ ] **Text contrast** ≥ 4.5:1 (Neon Ivory on Charcoal) — **measure it; the `14.2:1` figure previously stated here was never verified**
- [ ] **Large text** ≥ 3:1
- [ ] **UI components** (borders, icons) ≥ 3:1 — **test at 1px border width; neon on charcoal is the risk area**
- [ ] **Focus indicators** ≥ 3:1 against adjacent
- [ ] **Color not sole indicator** - Icons + text for status
- [ ] **CI contrast check exists** and fails below threshold

### 3.2 Keyboard Navigation
- [ ] **All interactive** reachable via Tab
- [ ] **Focus visible** at all times
- [ ] **Focus order** matches visual order
- [ ] **Focus trapping** in modals/drawers
- [ ] **Escape key** closes modals/dropdowns
- [ ] **Arrow keys** navigate menus/tabs

### 3.3 Screen Readers
- [ ] **Semantic HTML** (nav, main, aside, header, footer)
- [ ] **ARIA labels** on icon-only buttons
- [ ] **Live regions** for toasts/alerts (`aria-live="polite"`)
- [ ] **Heading hierarchy** h1 → h2 → h3
- [ ] **Form labels** explicit + associated
- [ ] **Table headers** `<th scope="col">`

### 3.4 Reduced Motion
- [ ] **All Framer Motion** wrapped in `useReducedMotion()`
- [ ] **CSS animations** respect `@media (prefers-reduced-motion: reduce)`
- [ ] **Auto-playing** content has pause controls
- [ ] **Scroll animations** instant final state
- [ ] **`prefers-reduced-motion` is a blocking gate**, not a convention

### 3.5 Color Blind Safe
- [ ] **Deuteranopia** (Coblis/Color Oracle) - verified
- [ ] **Protanopia** - verified
- [ ] **Tritanopia** - neon cyan/amber distinguishable
- [ ] **Monochrome** - patterns/text distinguishable

### 3.6 Internationalization
- [ ] **UI coverage** 100% across all **43** locales (`npm run check:i18n-ui-coverage`)
- [ ] **RTL** verified for `ar`, `fa`, `he`, `ur`
- [ ] Every new user-facing string has **43** translations before the build passes

---

## 4. Performance Checklist

### 4.1 Bundle & Load
- [ ] **Core bundle** < 250KB gzipped
- [ ] **Dashboard bundle** < 500KB gzipped
- [ ] **Code splitting** by route (Next.js automatic)
- [ ] **Dynamic imports** for heavy components (charts, editors)
- [ ] **Icon set** tree-shaken (only used glyphs bundled)
- [ ] **Framer Motion** addition visible in the frozen bundle-size ratchet (`config/quality/quality-baseline.json`, currently 8,045)

### 4.2 Runtime Performance
- [ ] **60fps scrolling** on 1000+ item virtualized lists
- [ ] **Animations** GPU-accelerated (`will-change: transform, box-shadow`)
- [ ] **Blur effects** limited to key surfaces
- [ ] **Neon glow** `will-change: box-shadow` on animated elements
- [ ] **Stagger** only mounted items (virtualization compatible)
- [ ] **Live-updating regions** (WS sidecar, `sseMerger`) do not cause layout thrash when animated

### 4.3 Core Web Vitals
- [ ] **LCP** < 2.5s
- [ ] **TTI** < 3.5s
- [ ] **CLS** < 0.1
- [ ] **FID** < 100ms
- [ ] **INP** < 200ms

### 4.4 Monitoring
- [ ] **Bundle analyzer** `npm run check:bundle-size`
- [ ] **Lighthouse CI** in PR pipeline
- [ ] **Web Vitals** tracking in production

---

## 5. Cross-Browser & Device Checklist

| Browser | Version | Status |
|---------|---------|--------|
| Chrome | 120+ | [ ] |
| Firefox | 115+ | [ ] |
| Safari | 17+ | [ ] |
| Edge | 120+ | [ ] |

| Device | Breakpoint | Tests |
|--------|------------|-------|
| iPhone SE | 375px | [ ] Sidebar collapse, header, card stack |
| iPhone 15 Pro | 393px | [ ] Safe areas, touch targets |
| iPad Mini | 768px | [ ] Sidebar toggle, 2-col grid |
| iPad Pro | 1024px | [ ] Full sidebar, 4-col grid |
| Desktop HD | 1920px | [ ] Hover states, max-width cap |
| 4K Monitor | 3840px | [ ] Centered content, no stretching |

---

## 6. Testing Checklist

### 6.1 Unit Tests (Vitest + Node-native)
- [ ] All new components have tests
- [ ] Design token utilities tested
- [ ] Animation utilities tested (reduced motion)
- [ ] Color contrast utilities tested
- [ ] Icon wrapper component tested
- ⚠️ **Not Jest.** The project runs Vitest 4.1.7 and Node-native runners; Jest is not installed. See the notice above.

### 6.2 Integration Tests
- [ ] DashboardLayout renders correctly
- [ ] Sidebar collapse/expand works
- [ ] Theme toggle persists
- [ ] Command palette opens/closes
- [ ] Page transitions complete

### 6.3 E2E Tests (Playwright)
- [ ] **Critical Path 1**: Login → Home → Providers → Settings
- [ ] **Critical Path 2**: Home → Analytics → Combo Builder → Save
- [ ] **Critical Path 3**: Providers → New Connection → Test → Save
- [ ] **Critical Path 4**: Settings → Appearance → Theme Toggle → Persist

### 6.4 Visual Regression
- [ ] **Percy/Chromatic** snapshots for all **114** pages
- [ ] **Component stories** in Storybook (if available)
- [ ] **Diff threshold** < 0.1% pixel difference
- [ ] **Per-page feature flag** so a regression on one page does not require a full rollback

### 6.5 Accessibility Tests
- [ ] **axe-core** in Playwright (`@axe-core/playwright`)
- [ ] **Lighthouse CI** accessibility score ≥ 95
- [ ] **Manual testing** with NVDA/VoiceOver

---

## 7. Acceptance Criteria (Definition of Done)

### Per Component
- [ ] TypeScript compiles without errors (`npm run typecheck:core`)
- [ ] ESLint passes (`npm run lint`)
- [ ] Unit tests pass (`npm run test:unit`)
- [ ] Storybook story exists (if applicable)
- [ ] All variants documented in component file
- [ ] Props interface exported
- [ ] 43 translations added for every new string

### Per Page
- [ ] Renders without console errors
- [ ] All interactive elements functional
- [ ] Data fetching handles loading/error/empty
- [ ] Responsive at 375/768/1024/1440px
- [ ] Animations smooth (or instant with reduced motion)
- [ ] Focus management correct
- [ ] No console errors in production build

### Per Phase
- [ ] **Phase 1**: `globals.css` valid, no build errors, dark mode renders
- [ ] **Phase 2**: Layout shell complete, navigation works
- [ ] **Phase 3**: All 10 components in Storybook, tests pass
- [ ] **Phase 4**: 4 core pages visually complete, interactive
- [ ] **Phase 5**: Animations work, reduced motion tested
- [ ] **Phase 6**: 25 pages consistent, no regressions
- [ ] **Phase 7**: All QA checks pass, PR ready

---

## 8. File Inventory (Modified/New)

### Phase 1: Foundation
| File | Status | Measured Size |
|------|--------|---------------|
| `src/app/globals.css` | Modified (additive migration) | **590 lines** |
| `src/shared/components/Icon.tsx` | **New** | ~50 |

### Phase 2: Layout
| File | Status | Measured Size |
|------|--------|---------------|
| `src/shared/components/layouts/DashboardLayout.tsx` | Modified | 139 |
| `src/shared/components/Sidebar.tsx` | Modified (major, **staged**) | **758** |
| `src/shared/components/Header.tsx` | Modified | 286 |
| `src/shared/components/Breadcrumbs.tsx` | Modified | 182 |

### Phase 3: Components
| File | Status | Measured Size |
|------|--------|---------------|
| `src/shared/components/Card.tsx` | Modified | 141 |
| `src/shared/components/Button.tsx` | Modified | 88 |
| `src/shared/components/Input.tsx` | Modified | 157 |
| `src/shared/components/Select.tsx` | Modified | 115 |
| `src/shared/components/DataTable.tsx` | **EXISTS — audit before creating `Table.tsx`** | — |
| `src/shared/components/Table.tsx` | **New** (or extend `DataTable`) | ~200 |
| `src/shared/components/docs/Tabs.tsx` | Exists but **docs-scoped** | — |
| `src/shared/components/Tabs.tsx` | **New** (or promote the docs one) | ~80 |
| `src/shared/components/Modal.tsx` | Modified | 267 |
| `src/shared/components/Badge.tsx` | Modified | 68 |
| `src/shared/components/Avatar.tsx` | Modified | 81 |
| `src/shared/components/NotificationToast.tsx` | Modified (or rename to `Toast.tsx`) | 208 |

**Also present — adopt, do not duplicate:** `ColumnToggle` · `FilterBar` · `EmptyState` · `ErrorPageScaffold` · `Loading` · `InfoTooltip` · `CommandPalette` · `Checkbox` · `Collapsible` · `CollapsibleSection` · `NavigationProgress` · `DegradationBadge` · `MonacoEditor` · `PresetSlider` · `PricingModal` · `RequestLoggerV2` — 135 shared component files in total.

### Phase 4: Core Pages
| File | Status | Measured Size |
|------|--------|---------------|
| `src/app/(dashboard)/dashboard/HomePageClient.tsx` | **Extract first**, then modify | **1,385** |
| `src/app/(dashboard)/dashboard/providers/page.tsx` | **Extract first**, then modify | **1,951** |
| `src/app/(dashboard)/dashboard/analytics/page.tsx` | Modified | 155 |
| `src/app/(dashboard)/dashboard/settings/page.tsx` | Modified (trivial) | **33** |

### Phase 5: Animations
| File | Status | Lines |
|------|--------|-------|
| `src/shared/lib/animations.ts` | **New** (directory exists) | ~80 |

### Phase 6: Batch Pages (25 pages)
| Batch | Pages | Exists? |
|-------|-------|---------|
| 1 | Logs, Costs, Cache, Quota | all yes |
| 2 | Combos, API Manager, MCP, A2A, Endpoint | all yes |
| 3 | Playground, Translator, Search, Memory | all yes |
| 4 | Agent Skills, CLI Agents, Health, **Resilience** | ⚠️ `resilience/page.tsx` **does not exist** → `resilience/connections/page.tsx` |
| 5 | Profile, Onboarding, Changelog, **System** | ⚠️ `system/page.tsx` **does not exist** → `system/1proxy/`, `system/proxy/`, `system/mitm-proxy/` (the last is a 40-line redirect stub to `/dashboard/tools/agent-bridge`) |

### Phase 7: QA
| File | Status |
|------|--------|
| Test updates | Modified |
| `CHANGELOG.md` | Modified |
| Documentation | Modified |

---

## 9. New Dependencies

```json
{
  "dependencies": {
    "framer-motion": "^11.0.0"
  }
}
```

> **`@phosphor-icons/react` removed — optional.** The app already ships Material Symbols `^0.45.2`, self-hosted. Adopting Phosphor is a 135-component migration plus a new dependency and a full accessible-name re-audit. **Recommendation:** keep Material Symbols behind a thin `Icon.tsx` seam.
>
> **Framer Motion is genuinely new.** It is not a declared dependency today and appears nowhere in `src/`. Verify Motion/React 19.2.8 compatibility (React is pinned exact) and check the frozen bundle-size ratchet so the addition is visible.

---

## 10. Breaking Changes Summary

| Area | Before | After | Migration |
|------|--------|-------|-----------|
| **Icons** | Material Symbols | *unchanged* (recommended) | None, if `Icon.tsx` wraps Material Symbols |
| **Typography** | Inter | *unchanged* (recommended) | None, if Inter is kept |
| **Colors** | Coral/Indigo palette | Charcoal + Neon | globals.css only (additively) |
| **Button** | 2 variants | 3 variants (+ghost) | Add `variant` prop |
| **Card** | 1 style | 3 variants | Add `variant` prop |
| **Badge** | 4 colors | 5 semantic | Update `variant` values |
| **Colour mode** | light + dark + system | dark-only *(proposed)* | ⚠️ **This removes capability.** Document as a decision with its accessibility consequences |

---

## 11. PR Checklist

### Pre-PR
- [ ] Branch: `feat/charcoal-neon-redesign`
- [ ] Worktree: `.claude/worktrees/charcoal-neon-redesign`
- [ ] Base branch confirmed with operator
- [ ] `npm install` run in worktree
- [ ] `cp -al ../node_modules node_modules` (hard links)

### PR Content
- [ ] Title: `feat: charcoal metallic neon redesign (phased)`
- [ ] Description: Screenshots of Home, Providers, Analytics, Settings
- [ ] Testing notes: Commands run, coverage results
- [ ] Breaking changes documented
- [ ] Migration guide for icon/typography changes

### CI Gates
- [ ] `npm run lint` passes
- [ ] `npm run typecheck:core` passes
- [ ] `npm run test:unit` passes
- [ ] `npm run test:vitest` passes
- [ ] `npm run check:docs-all` passes
- [ ] `npm run build` succeeds
- [ ] `npm run dashboard-typecheck` passes — **mandatory**, because `next.config.mjs:296` sets `typescript.ignoreBuildErrors: true`, so a green build is **not** evidence of type safety
- [ ] `npm run check:i18n-ui-coverage` passes (100% across 43 locales)

---

## 12. Post-Merge

- [ ] Visual regression baseline updated
- [ ] Storybook deployed (if applicable)
- [ ] Documentation site updated
- [ ] Team notification with screenshots
- [ ] Monitor error rates 24h post-deploy