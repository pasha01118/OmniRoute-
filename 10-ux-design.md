# 10 — UX & Design Review

> **Verification basis.** OmniRoute `5764027`, version `3.8.50`. Every font, icon, colour, and component claim was verified against source. **Three headline assumptions in the original design report are wrong** — see §2.

---

## 1. Design System: Actual State

### 1.1 What is really installed

| Asset | Reality | Evidence |
|---|---|---|
| **Typeface** | **Inter** via `next/font/google` | `src/app/layout.tsx:1,14-17,138` |
| `next/font` usage | **exactly one file repo-wide** | `layout.tsx:1` |
| **Icon set** | **Material Symbols**, self-hosted | `material-symbols ^0.45.2`; `@import "material-symbols/outlined.css"` in `globals.css` (comment at `layout.tsx:97-99`, #3695) |
| `framer-motion` | **NOT a declared dependency**; zero imports in `src/` | `package.json`; `grep -rn "from \"framer-motion\"" src/` → no output |
| `@phosphor-icons/react` | **NOT installed** | `node_modules/@phosphor-icons` does not exist |
| **DM Sans** | **Absent** | `grep -rniE "dm.?sans" src/` → 0 files |
| Theming | Zustand `persist` + CSS custom properties + `dark` class | `src/store/themeStore.ts` |
| Colour presets | **7** | `themeStore.ts:56-63` |
| Default colour theme | `coral` `#e54d5e` | `themeStore.ts` |
| Colour modes | `light` / `dark` / `system` | `themeStore.ts:66-76` |
| Global CSS | **590 lines** | `src/app/globals.css` |
| Locales | **43** | `config/i18n.json` |
| RTL locales | `ar`, `fa`, `he`, `ur` | `config/i18n.json` |

### 1.2 The theme store

`src/store/themeStore.ts` — `create<ThemeState>()(persist(...))` persisted to `localStorage`.

```ts
theme:        "light" | "dark" | "system"   default THEME_CONFIG.defaultTheme
colorTheme:   7 named presets               default "coral"
customColor:  "#rrggbb"                     default "#3b82f6"
```

`applyTheme()` (`:66-76`) toggles `documentElement.classList` `dark`; `"system"` resolves via `matchMedia("(prefers-color-scheme: dark)")`.

`applyColorTheme()` (`:79-88`) sets `--color-primary` and `--color-primary-hover` (base shaded by `-0.14` via `shadeHexColor`) — runtime accent switching with a validated hex fallback to `#3b82f6`.

**The 7 presets:**

| Preset | Hex |
|---|---|
| coral | `#e54d5e` |
| blue | `#3b82f6` |
| red | `#ef4444` |
| green | `#22c55e` |
| violet | `#8b5cf6` |
| orange | `#f97316` |
| cyan | `#06b6d4` |

`normalizeHexColor` validates `/^#([0-9a-fA-F]{6})$/` — strict 6-digit, no 3-digit shorthand, no named colours.

### 1.3 App shell

`src/app/layout.tsx` (155 lines) is an **async server component** (no `"use client"`).

Provider nesting, outermost → innermost:

```
NextIntlClientProvider(locale, messages)
  └ BasePathNetworkProvider
      ├ PwaRegister
      ├ LocaleAutoDetect
      └ ThemeProvider → {children}
```

`src/shared/components/ThemeProvider.tsx` is 14 lines and a **passthrough** — no React context, just `useEffect(() => initTheme(), [initTheme])`. Worth knowing if you plan to consume theme state: there is no context to hook; components read CSS variables or the Zustand store directly.

**Metadata** (`:24-54`) is async and reads DB settings: `instanceName` (default `"OmniRoute"`), `customFaviconUrl`/`customFaviconBase64` → `/api/settings/favicon`, `manifest`, `appleWebApp` with `statusBarBar: "black-translucent"`.

**Viewport** (`:19-22`): `themeColor: "#0b0f1a"`, `viewportFit: "cover"`.

**i18n/RTL** (`:57-63`): `<html lang={locale} dir={isRtl ? "rtl" : "ltr"} suppressHydrationWarning>`.

**Accessibility** (`:139-144`): skip-to-content link, `href="#main-content"`, `sr-only focus:not-sr-only`.

**Two `dangerouslySetInnerHTML` blocks** (`:71-96`, `:100-136`):
1. A pre-hydration scrubber stripping browser-extension attributes (`bis_skin_checked`, `data-google-query-id`, `data-new-gr-c-s-check-loaded`, `data-gr-ext-installed`, `data-lt-installed`, `data-lt-tmp-id`) via `MutationObserver`, auto-`disconnect()` after 5 s. Cites Bitdefender, Grammarly, LanguageTool.
2. `window.crypto.randomUUID` polyfill (with a `Math.random` degraded branch) plus persisted-theme application from `localStorage` before hydration, honouring `prefers-color-scheme`.

Both are legitimate uses — extension attributes cause hydration mismatches, and `next/font` needs a pre-paint theme. The `Math.random` fallback for `randomUUID` is the one weak point: it is not cryptographically random, and any consumer treating its output as an identifier should be checked.

### 1.4 Component inventory (measured)

| Component | Path | Lines | Status |
|---|---|---|---|
| `DashboardLayout` | `src/shared/components/layouts/DashboardLayout.tsx` | 139 | exists |
| `Sidebar` | `src/shared/components/Sidebar.tsx` | **758** | exists — the "758 lines" claim is **correct** |
| `Header` | `src/shared/components/Header.tsx` | 286 | exists |
| `Breadcrumbs` | `src/shared/components/Breadcrumbs.tsx` | 182 | exists |
| `Card` | `src/shared/components/Card.tsx` | 141 | exists |
| `Button` | `src/shared/components/Button.tsx` | 88 | exists |
| `Input` | `src/shared/components/Input.tsx` | 157 | exists |
| `Select` | `src/shared/components/Select.tsx` | 115 | exists |
| `Modal` | `src/shared/components/Modal.tsx` | 267 | exists |
| `Badge` | `src/shared/components/Badge.tsx` | 68 | exists |
| `Avatar` | `src/shared/components/Avatar.tsx` | 81 | exists |
| **`DataTable`** | `src/shared/components/DataTable.tsx` | — | **exists — not in the report inventory** |
| `NotificationToast` | `src/shared/components/NotificationToast.tsx` | 208 | exists |
| `Table` | — | — | **absent** |
| `Tabs` | — | — | **absent** (only `components/docs/Tabs.tsx`) |
| `Toast` | — | — | **absent** |
| `Icon` | — | — | **absent** |
| `src/shared/lib/animations.ts` | — | — | **absent** (directory exists) |

Also present and unmentioned: `Checkbox`, `Collapsible`, `CollapsibleSection`, `ColumnToggle`, `CommandPalette`, `EmptyState`, `ErrorPageScaffold`, `FilterBar`, `Footer`, `InfoTooltip`, `Loading`, `MonacoEditor`, `OAuthModal`, `PresetSlider`, `PricingModal`, `ProviderIcon`, `ReasoningRoutingRules`, `NavigationProgress`, `MaintenanceBanner`, `CloudSyncStatus`, `DegradationBadge`, `RequestLoggerV2`, `ConsoleLogViewer`, and ~20 more.

**135 shared component files** in total, against the report's "100+".

---

## 2. Three Corrections to the Original Design Report

### 2.1 CORRECTION 1 — DM Sans is not installed

The design report specifies DM Sans as the target typeface. Measured: the app uses **Inter**, and `grep -rniE "dm.?sans" src/ --include=*.ts --include=*.tsx --include=*.css` returns **zero files**.

Adopting DM Sans is a real change, not a config toggle. Via `next/font/google` it is a small diff, but it invalidates the entire type scale in the 590-line `globals.css` and every fixed-height UI element.

**Recommendation:** keep Inter. It is already correctly wired through `next/font` with zero layout-shift risk, and the design rationale for a geometric sans is satisfied. If DM Sans is a hard brand requirement, treat it as a Phase 2 token change with a full re-audit, not a Day-3 task.

### 2.2 CORRECTION 2 — Phosphor is not installed; Material Symbols is

The design report specifies `@phosphor-icons/react: ^2.1.0` and Phosphor duotone at 16–24 px. Measured: `material-symbols ^0.45.2` is a **declared dependency**, self-hosted via `@import` in `globals.css` (deliberately self-hosted per #3695), and `@phosphor-icons` does not exist in `node_modules`.

Adopting Phosphor means: a new runtime dependency, a dual-icon-set migration across 135 components, and a re-audit of every icon's accessible name. Material Symbols is already self-hosted, already offline-friendly, and already integrated.

**Recommendation:** keep Material Symbols and add a thin `Icon.tsx` abstraction so the icon system has a single seam. Revisit Phosphor only if a design review genuinely requires the duotone treatment.

### 2.3 CORRECTION 3 — Framer Motion is not a dependency

The design report lists Framer Motion as a stack component and `^11.0.0` as a new dependency. Measured: **not declared** in `package.json`, and **zero** `from "framer-motion"` imports in `src/`. It appears in `node_modules` only transitively.

So Framer Motion is a legitimate *new* dependency for the redesign — but the architecture report's tech-stack grid lists it under the **current** frontend stack, which is wrong. Any bundle-size budget must account for it as an addition.

Note also that Motion's current major is well past 11, and OmniRoute pins React `19.2.8` exactly. Verify Motion/React 19 compatibility before adopting.

### 2.4 CORRECTION 4 — "dark-only" is a removal, not an addition

The design direction says dark-only. The product currently ships **light + dark + system** with 7 colour presets and a full `applyTheme()` implementation. A dark-only design is therefore a **removal of capability**, including the 43-locale RTL experience's light variant.

That may be the right call, but it must be stated as a decision with its accessibility consequences (users who need high-contrast light themes, and the `prefers-color-scheme: light` preference), not presented as a default.

---

## 3. Existing Design Assets to Build On

### 3.1 `globals.css` — 590 lines

Single global stylesheet. Currently holds the Material Symbols `@import`, the theme variables, and the Tailwind entry. A full rewrite is a high-risk operation because every component references the existing custom-property names.

**Recommendation:** additive migration. Introduce the new token names alongside the old ones, migrate component-by-component, then remove the old names once `grep` finds no references. A big-bang rewrite of 590 lines that 135 components depend on is the single most likely way to produce a visually broken, hard-to-bisect regression.

### 3.2 The `DataTable` discovery

`src/shared/components/DataTable.tsx` already exists and is not in the report's inventory, which proposes creating `Table.tsx`. Before writing a new table, read `DataTable.tsx` plus `ColumnToggle.tsx`, `FilterBar.tsx`, `Pagination`-equivalent components, and the per-page tables in `providers`, `costs`, `logs`, and `usage`.

The dashboard has heavy table usage; a new generic `Table` that duplicates `DataTable` will be adopted by some pages and not others, producing exactly the inconsistency the redesign is meant to remove.

### 3.3 Larger page files than the roadmap assumes

| Page | Roadmap estimate | Measured |
|---|---|---|
| `HomePageClient.tsx` | ~400 | **1,385** |
| `providers/page.tsx` | ~200 | **1,951** |
| `analytics/page.tsx` | ~150 | 155 |
| `settings/page.tsx` | ~200 | **33** |
| `Sidebar.tsx` | ~300 (delta implied) | 758 (total) |
| `Header.tsx` | ~100 (delta implied) | 286 (total) |

`providers/page.tsx` at 1,951 lines is nearly 5× the budget. `HomePageClient.tsx` at 1,385 is 3.5×. The roadmap must be rebudgeted from measured sizes, and the two large pages need **component extraction before restyling** — restyling a 1,951-line file in one pass is unreviewable.

### 3.4 Page routes that do not exist

The roadmap's Phase 6 lists `resilience/page.tsx` and `system/page.tsx`. Neither exists:

| Roadmap target | Actual |
|---|---|
| `resilience/page.tsx` | `resilience/connections/page.tsx` |
| `system/page.tsx` | `system/1proxy/page.tsx`, `system/proxy/page.tsx`, `system/mitm-proxy/page.tsx` |

The other 19 Phase 6 targets exist. `system/mitm-proxy/page.tsx` (40 lines) is a **redirect stub** with a 2.5 s auto-`router.replace` to `/dashboard/tools/agent-bridge` (`page.tsx:15-20`), and a dev-only error boundary (`error.tsx`, 38 lines) that shows the message only when `NODE_ENV === "development"`.

### 3.5 MITM settings UI

`src/app/(dashboard)/dashboard/settings/components/MitmProxyTab.tsx` (429 lines):
- `TRANSPARENT_MITM_PORT = 443` — **read-only**, value coerced back to `"443"` (`:7,85,113,129`).
- `apiKey` and `sudoPassword` password inputs; the sudo placeholder shows `status.hasCachedPassword ? cachedPassword : sudoPassword` (`:271`).
- States: `status`, `loading`, `saving`, `feedback {type, message}` (`:68-76`).
- `regenerateCertificate` uses a **native `confirm()`** (`:147-170`) — an inconsistency against the app's own modal system, and an accessibility gap.
- Cert download is an `<a href="/api/settings/mitm?download=cert">`, enabled only when `certExists` (`:304-314`).
- Stats cards: interceptedRequests, activeConnections, dnsConfigured, pid, lastInterceptAt (`:327-372`).

`useMitmSudoPrompt.tsx` (157 lines) — `canRunWithoutPassword = isWin || hasCachedPassword || !needsSudoPassword` (`:108`); otherwise stashes the action in a ref and opens a modal (`:116-127`). On error the action is restored to the ref and the modal reopens with the message (`:129-145`). The password is cleared on confirm and on close (`:31,42`) and is never stored in the hook.

### 3.6 Accessibility patterns already present

| Pattern | Location |
|---|---|
| Skip-to-content | `layout.tsx:139-144` |
| `EmptyState` component | `src/shared/components/EmptyState.tsx` |
| `ErrorPageScaffold` | `src/shared/components/ErrorPageScaffold.tsx` |
| `Loading` | `src/shared/components/Loading.tsx` |
| `InfoTooltip` | `src/shared/components/InfoTooltip.tsx` |
| HTTP error pages | `src/app/{400,401,403,408,429,500,502,503}/` |
| `error.tsx` / `global-error.tsx` | `src/app/` |
| i18n UI coverage gate | frozen at **100%** in `quality-baseline.json` |

The 100% i18n UI coverage ratchet is a strong accessibility-and-i18n property: it means no user-facing string can be added without a translation. Any redesign must maintain that gate, which means **every new component needs 43 translations** — a real cost that the roadmap does not account for.

---

## 4. Maintainability Signals

Measured across all of `src/`:

| Pattern | Files | Matches |
|---|---|---|
| `eslint-disable` | 33 | 46 |
| `dangerouslySetInnerHTML` | 2 | 3 |
| `TODO` | 4 | 4 |
| `@ts-ignore` | 2 | 2 |
| `FIXME` | 0 | 0 |
| `HACK` | 0 | 0 |
| `ponytail` (in-house "deliberately weird" marker) | 4 | 6 |

**4 TODOs in an 888-file `src/lib` plus 114 dashboard pages is a remarkably clean codebase.** The `ponytail` convention is notable — the team has an explicit marker for intentional weirdness, which is a healthier practice than silent deviation.

### 4.1 ESLint policy

`eslint.config.mjs` (197 lines) — zero-warning:
- `next/core-web-vitals` plus custom **errors**: `no-eval`, `no-implied-eval`, `no-new-func` (`:61-63`).
- `restricted-imports`: localDb barrel, executors boundary, PropTypes.
- `no-explicit-any` is an **error** in `open-sse` and tests.
- `react-hooks/exhaustive-deps` is an **error**.
- Turkish-safe `matchesSearch` restriction and a new-`toNumber` bar (#7879).
- 3,342-line `config/quality/eslint-suppressions.json` — per-file frozen suppressions.

`no-eval` / `no-new-func` as **errors** is why `/api/middleware/` (arbitrary JS via `vm.Script`) is in `LOCAL_ONLY_API_PREFIXES` — the lint rule and the authz tier are consistent about the danger.

### 4.2 Where the suppressions are

| File | Count | Kind |
|---|---|---|
| `dashboard/cache/media/MediaPageClient.tsx` | 7 | 5 `no-restricted-syntax` (`:334-342`), `no-img-element` (`:391`), `exhaustive-deps` (`:489`) |
| `dashboard/costs/quota-share/components/PoolWizard.tsx` | 3 | `exhaustive-deps` (`:267,314,326`) |
| `app/layout.tsx` | 2 | `dangerouslySetInnerHTML` (`:72,101`) |
| `shared/components/ProviderIcon.tsx` | 2 | `no-img-element` (`:361,480`) — operator-supplied remote URLs, external SVG from thesvg.org |
| `dashboard/settings/components/proxy/FreePoolTab.tsx` | 2 | `set-state-in-effect` (`:51,101`) |
| 28 more files | 1 each | mostly `exhaustive-deps` / `set-state-in-effect` |

`ProviderIcon.tsx` rendering operator-supplied remote SVG URLs is a **security-relevant** suppression, not just a lint one: an operator can configure a provider icon URL, and the dashboard renders it as an image. If any code path allows *non-operator* control of that URL, it becomes a tracking-pixel vector. Worth an explicit review.

---

## 5. Design System Specification (Corrected)

### 5.1 Foundations — decided

| Decision | Choice | Rationale |
|---|---|---|
| Typeface | **Inter** (existing) | Already `next/font`-wired, zero CLS risk, 590-line type scale stays valid |
| Icon set | **Material Symbols** (existing) + new `Icon.tsx` seam | Self-hosted, offline-friendly, 135 components already consistent |
| Motion | **Add Framer Motion** as a new dependency | Genuinely new; budget it and verify React 19 compat |
| Colour mode | **Decision required** | dark-only removes light + system; see §2.4 |
| Accent | Extend the 7 existing presets | Don't replace; users may have a persisted `colorTheme` |
| i18n | **43 locales**, RTL for `ar/fa/he/ur` | Gate is at 100% coverage; every new string needs 43 translations |

### 5.2 Tokens to define

Building on `themeStore.ts`'s existing CSS-variable pattern (`--color-primary`, `--color-primary-hover`):

```
/* surfaces */
--surface-base        /* charcoal ground */
--surface-raised      /* card */
--surface-overlay     /* modal, dropdown */
--surface-inset       /* inputs, wells */

/* text */
--text-primary        /* Neon Ivory #FFF8E7 on #0A0E14 */
--text-secondary
--text-tertiary
--text-disabled

/* borders */
--border-subtle       /* 1px, low alpha */
--border-default
--border-strong
--border-focus        /* 4px focus ring per the spec */

/* accent (7 presets × 3) */
--color-primary
--color-primary-hover
--color-primary-subtle

/* status */
--status-success --status-warning --status-error --status-info
```

### 5.3 Contrast — measured, not asserted

The original report asserts `14.2:1` for Neon Ivory `#FFF8E7` on Charcoal `#0A0E14`. That is plausible but was not verified. **Compute and record every pair in CI** rather than asserting it in a document.

Minimum targets:

| Element | Requirement |
|---|---|
| Body text | ≥ 4.5:1 (WCAG 2.1 AA) |
| Large text (≥ 18.66px bold / 24px) | ≥ 3:1 |
| UI borders / icons | ≥ 3:1 (WCAG 1.4.11) |
| Focus indicators | ≥ 3:1 against adjacent colours |
| Disabled text | exempt, but must not be the only signal |

**Neon accents on charcoal are the risk area.** A saturated neon border at 1px against a dark ground can easily fall below 3:1 while still looking correct. Each accent preset must be tested at 1px border width, not at large sizes.

### 5.4 Component work

**Restyle (exists):** `Card` (141), `Button` (88), `Input` (157), `Select` (115), `Modal` (267), `Badge` (68), `Avatar` (81), `NotificationToast` (208), `DashboardLayout` (139), `Sidebar` (758), `Header` (286), `Breadcrumbs` (182).

**Create:** `Icon.tsx` (Material Symbols wrapper), `Table.tsx` (**after** reading `DataTable.tsx`), `Tabs.tsx` (the only existing one is `docs/Tabs.tsx`, docs-scoped), `Toast.tsx` (or rename `NotificationToast.tsx`), `src/shared/lib/animations.ts`.

**Already present and unmentioned** — adopt rather than duplicate: `DataTable`, `ColumnToggle`, `FilterBar`, `EmptyState`, `ErrorPageScaffold`, `Loading`, `InfoTooltip`, `CommandPalette`, `Checkbox`, `Collapsible`, `NavigationProgress`, `DegradationBadge`.

### 5.5 Motion system

New `src/shared/lib/animations.ts`. Respect:

- `prefers-reduced-motion` — the project has a nightly a11y job and an a11y checklist item; wire it as a hard gate, not a convention.
- `View Transitions` are not used anywhere; introducing them alongside Framer Motion is a decision, not an accident.
- The existing `sseMerger` and stream machinery means the dashboard has many live-updating regions; animation on those can cause layout thrash.

---

## 6. Phased Plan (rebudgeted)

| Phase | Scope | Days (1 dev) | Gate |
|---|---|---|---|
| **0. Truth** | Fix stale numbers, reconcile README, align `AGENTS.md` | 1 | `check:docs-all` green |
| **1. Decisions** | ADR: typeface, icons, motion, colour mode | 1 | Written ADR |
| **2. Tokens** | Additive CSS-variable migration, contrast CI check | 3 | AA automated pass |
| **3. Shell** | `DashboardLayout` (139), `Breadcrumbs` (182), `Header` (286) | 2 | Visual regression |
| **4. Sidebar** | `Sidebar.tsx` (758) — staged, state-heavy | 4 | Visual regression + keyboard |
| **5. Primitives** | `Card` `Button` `Input` `Select` `Badge` `Avatar` `Modal` | 5 | Storybook coverage |
| **6. New primitives** | `Icon`, `Table`(after `DataTable` audit), `Tabs`, `Toast`, `animations.ts` | 4 | Component review |
| **7. Extract first** | Decompose `HomePageClient` (1,385) and `providers/page.tsx` (1,951) | 5 | No behaviour change |
| **8. Core pages** | Restyle the four extracted/measured pages | 6 | Per-page review |
| **9. Batch pages** | ~100 remaining `page.tsx` via one per-page pattern | 15 | lint + a11y gates |
| **10. Motion** | Framer Motion integration, reduced-motion | 3 | `prefers-reduced-motion` verified |
| **11. QA** | contrast, keyboard, screen reader, cross-browser, bundle | 5 | Ratchet green |

**Total ≈ 54 dev-days** — materially more than the 39 in the original roadmap, driven by the measured file sizes and the 43-locale translation cost that the roadmap omits entirely.

---

## 7. Findings

| ID | Sev | Finding | Section |
|---|---|---|---|
| **U-1** | High | Roadmap budgets `HomePageClient` at ~400 (actual 1,385) and `providers/page.tsx` at ~200 (actual 1,951) — 3.5× and 9.75× under. | §3.3 |
| **U-2** | High | Design system specifies DM Sans + Phosphor; neither is installed. | §2.1, §2.2 |
| **U-3** | High | Framer Motion is listed as current stack; it is not a dependency. | §2.3 |
| **U-4** | Medium | "Dark-only" removes existing light + system modes — a capability reduction, not a default. | §2.4 |
| **U-5** | Medium | `DataTable.tsx` exists but the inventory proposes creating `Table.tsx` — duplicate risk. | §3.2 |
| **U-6** | Medium | Roadmap targets `resilience/page.tsx` and `system/page.tsx`; neither exists. | §3.4 |
| **U-7** | Medium | Every new string needs **43 translations** (i18n gate at 100% coverage); unbudgeted. | §3.6 |
| **U-8** | Medium | `ProviderIcon.tsx` renders operator-supplied remote SVG URLs under a lint suppression — potential tracking-pixel vector. | §4.2 |
| **U-9** | Low | `MitmProxyTab` uses a native `confirm()` instead of the app's modal. | §3.5 |
| **U-10** | Low | `ThemeProvider` is a passthrough with no React context — no seam for theme consumption. | §1.3 |
| **U-11** | Low | Asserted `14.2:1` contrast ratio was never verified. | §5.3 |
| **U-12** | Info | ~20 shared components exist but are absent from the inventory. | §3.3 |
| **U-13** | Info | 2 `dangerouslySetInnerHTML` uses in `layout.tsx` — both legitimate, both deserve a review note. | §1.3 |

---

## 8. Recommendations

**Immediate**
1. Write the ADR for typeface / icons / motion / colour mode. Three of the four are currently mis-stated in the design report (U-2, U-3, U-4).
2. Rebudget the roadmap from measured file sizes (U-1); extract `providers/page.tsx` and `HomePageClient.tsx` **before** restyling.
3. Fix the two non-existent route targets (U-6).
4. Add `DataTable.tsx` to the inventory and decide wrap-vs-replace (U-5).

**Short term**
5. Add a contrast CI check that computes every token pair and fails below threshold, replacing the asserted `14.2:1` (U-11).
6. Account for 43 translations per new string in the estimate (U-7); a component with 10 strings costs 430 translation units.
7. Audit `ProviderIcon.tsx`'s remote-SVG path and confirm the URL is operator-only (U-8).
8. Replace the native `confirm()` in `MitmProxyTab` with the app's `Modal` (U-9).

**Medium term**
9. Migrate `globals.css` additively with a per-component sweep and a `grep`-verified removal step, rather than a big-bang rewrite.
10. Give `ThemeProvider` a real context so theme state has a consumption seam (U-10).
11. Add `prefers-reduced-motion` as a blocking gate, not a convention.
12. Run the redesign behind a feature flag with a per-page opt-in, so a visual regression on one of 114 pages does not require a full rollback.
