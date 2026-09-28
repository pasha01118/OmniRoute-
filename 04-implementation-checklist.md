# OmniRoute Dashboard Redesign - Implementation Checklist

## 1. Visual Quality Checklist

### 1.1 Color & Theme
- [ ] **No pure white text** - All text uses Neon Ivory (`#FFF8E7`) variants
- [ ] **No emoji icons** - All icons from Phosphor (`@phosphor-icons/react`)
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
- [ ] **Text contrast** 14.2:1 (Neon Ivory on Charcoal) ✓
- [ ] **Large text** ≥ 4.5:1
- [ ] **UI components** (borders, icons) ≥ 3:1
- [ ] **Focus indicators** ≥ 3:1 against adjacent
- [ ] **Color not sole indicator** - Icons + text for status

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

### 3.5 Color Blind Safe
- [ ] **Deuteranopia** (Coblis/Color Oracle) - verified
- [ ] **Protanopia** - verified
- [ ] **Tritanopia** - neon cyan/amber distinguishable
- [ ] **Monochrome** - patterns/text distinguishable

---

## 4. Performance Checklist

### 4.1 Bundle & Load
- [ ] **Core bundle** < 250KB gzipped
- [ ] **Dashboard bundle** < 500KB gzipped
- [ ] **Code splitting** by route (Next.js automatic)
- [ ] **Dynamic imports** for heavy components (charts, editors)
- [ ] **Phosphor Icons** tree-shaken (only used icons bundled)

### 4.2 Runtime Performance
- [ ] **60fps scrolling** on 1000+ item virtualized lists
- [ ] **Animations** GPU-accelerated (`will-change: transform, box-shadow`)
- [ ] **Blur effects** limited to key surfaces
- [ ] **Neon glow** `will-change: box-shadow` on animated elements
- [ ] **Stagger** only mounted items (virtualization compatible)

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

### 6.1 Unit Tests (Jest/Node)
- [ ] All new components have tests
- [ ] Design token utilities tested
- [ ] Animation utilities tested (reduced motion)
- [ ] Color contrast utilities tested
- [ ] Icon wrapper component tested

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
- [ ] **Percy/Chromatic** snapshots for all 50+ pages
- [ ] **Component stories** in Storybook (if available)
- [ ] **Diff threshold** < 0.1% pixel difference

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

### Per Page
- [ ] Renders without console errors
- [ ] All interactive elements functional
- [ ] Data fetching handles loading/error/empty
- [ ] Responsive at 375/768/1024/1440px
- [ ] Animations smooth (or instant with reduced motion)
- [ ] Focus management correct

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
| File | Status | Lines Changed |
|------|--------|---------------|
| `src/app/globals.css` | Modified (rewrite) | ~600 |
| `src/shared/components/Icon.tsx` | New | ~50 |

### Phase 2: Layout
| File | Status | Lines Changed |
|------|--------|---------------|
| `src/shared/components/layouts/DashboardLayout.tsx` | Modified | ~50 |
| `src/shared/components/Sidebar.tsx` | Modified (major) | ~300 |
| `src/shared/components/Header.tsx` | Modified | ~100 |
| `src/shared/components/Breadcrumbs.tsx` | Modified | ~30 |

### Phase 3: Components
| File | Status | Lines Changed |
|------|--------|---------------|
| `src/shared/components/Card.tsx` | Modified | ~80 |
| `src/shared/components/Button.tsx` | Modified | ~120 |
| `src/shared/components/Input.tsx` | Modified | ~80 |
| `src/shared/components/Select.tsx` | Modified | ~100 |
| `src/shared/components/Table.tsx` | New | ~200 |
| `src/shared/components/Tabs.tsx` | Modified | ~80 |
| `src/shared/components/Modal.tsx` | Modified | ~120 |
| `src/shared/components/Badge.tsx` | Modified | ~60 |
| `src/shared/components/Avatar.tsx` | Modified | ~40 |
| `src/shared/components/Toast.tsx` / `NotificationToast.tsx` | Modified | ~80 |

### Phase 4: Core Pages
| File | Status | Lines Changed |
|------|--------|---------------|
| `src/app/(dashboard)/dashboard/HomePageClient.tsx` | Modified (major) | ~400 |
| `src/app/(dashboard)/dashboard/providers/page.tsx` | Modified | ~200 |
| `src/app/(dashboard)/dashboard/analytics/page.tsx` | Modified | ~150 |
| `src/app/(dashboard)/dashboard/settings/page.tsx` | Modified | ~200 |

### Phase 5: Animations
| File | Status | Lines |
|------|--------|-------|
| `src/shared/lib/animations.ts` | New | ~80 |

### Phase 6: Batch Pages (25 pages)
| Batch | Pages | Est. Total Lines |
|-------|-------|------------------|
| 1 | Logs, Costs, Cache, Quota | ~800 |
| 2 | Combos, API Manager, MCP, A2A, Endpoint | ~1000 |
| 3 | Playground, Translator, Search, Memory | ~800 |
| 4 | Agent Skills, CLI Agents, Health, Resilience | ~800 |
| 5 | Profile, Onboarding, Changelog, System | ~600 |

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
    "@phosphor-icons/react": "^2.1.0",
    "framer-motion": "^11.0.0"
  }
}
```

---

## 10. Breaking Changes Summary

| Area | Before | After | Migration |
|------|--------|-------|-----------|
| **Icons** | Material Symbols | Phosphor | Update all imports |
| **Typography** | System fonts | DM Sans | globals.css only |
| **Colors** | Coral/Indigo palette | Charcoal + Neon | globals.css only |
| **Button** | 2 variants | 3 variants (+ghost) | Add `variant` prop |
| **Card** | 1 style | 3 variants | Add `variant` prop |
| **Badge** | 4 colors | 5 semantic | Update `variant` values |

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

---

## 12. Post-Merge

- [ ] Visual regression baseline updated
- [ ] Storybook deployed (if applicable)
- [ ] Documentation site updated
- [ ] Team notification with screenshots
- [ ] Monitor error rates 24h post-deploy