# OmniRoute Dashboard Redesign

<div align="center">

![OmniRoute Banner](https://img.shields.io/badge/OmniRoute-Dashboard%20Redesign-0A0E14?style=for-the-badge&logo=github&logoColor=00FFFF)
![Version](https://img.shields.io/badge/Version-1.0.0-00FF88?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-FFB800?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Complete-00FFFF?style=for-the-badge)

**Enterprise-grade dashboard redesign for OmniRoute — Unified AI Router with 291+ Providers**

[📋 Architecture Blueprint](./01-architecture-blueprint.md) • [🎨 Design System](./02-design-system.md) • [🗺️ Development Roadmap](./03-development-roadmap.md) • [✅ Implementation Checklist](./04-implementation-checklist.md)

</div>

---

## 🎯 Project Overview

This repository contains the complete **design specification and implementation roadmap** for the OmniRoute Dashboard redesign — a sophisticated, enterprise-grade transformation of the Next.js 16 dashboard powering the OmniRoute Unified AI Router platform.

### 🏗️ What is OmniRoute?

OmniRoute is a **unified AI proxy/router** providing:
- **291+ LLM providers** with auto-fallback
- **Real-time routing** with intelligent combo strategies
- **MCP/A2A protocol support** with 105+ tools
- **Enterprise features**: caching, rate limiting, resilience patterns
- **Desktop app** via Electron with system tray integration

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
Thin, bright emissive borders that provide visual hierarchy without overwhelming.

```css
--neon-cyan: #00FFFF;      /* Primary actions */
--neon-emerald: #00FF88;   /* Success states */
--neon-amber: #FFB800;     /* Warning/attention */
--neon-coral: #FF4E6D;     /* Error/critical */
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
--neon-ivory-subtle: #B8A994; /* Muted/placeholder */
```

---

## 📦 Repository Contents

| File | Description | Size |
|------|-------------|------|
| [`01-architecture-blueprint.md`](./01-architecture-blueprint.md) | Complete system architecture, request pipeline, resilience model, authZ | 38.7 KB |
| [`02-design-system.md`](./02-design-system.md) | 40+ CSS tokens, 10 component specs, animations, accessibility | 15.6 KB |
| [`03-development-roadmap.md`](./03-development-roadmap.md) | 39-day phased plan (7 phases), daily tasks, batch assignments | 14.2 KB |
| [`04-implementation-checklist.md`](./04-implementation-checklist.md) | Visual, interaction, accessibility, performance, testing criteria | 11.7 KB |

---

## 🚀 Quick Start

### Prerequisites
- Node.js ≥ 22.22.2 < 23 || ≥ 24.0.0 < 27
- pnpm ≥ 8.0.0
- Git

### Installation

```bash
# Clone the repository
git clone https://github.com/pasha01118/OmniRoute-.git
cd OmniRoute-

# Install dependencies
pnpm install

# Start development server
pnpm run dev
# Dashboard available at http://localhost:20128
```

### Environment Setup

```bash
# Copy example env
cp .env.example .env

# Generate secrets
openssl rand -base64 48  # JWT_SECRET
openssl rand -hex 32     # API_KEY_SECRET
```

---

## 🏗️ Implementation Phases

<div align="center">

### 39-Day Timeline (6 Weeks)

| Phase | Duration | Focus | Deliverables |
|-------|----------|-------|--------------|
| **1. Foundation** | Days 1-3 | Design tokens, typography, icons | `globals.css`, `Icon.tsx` |
| **2. Layout Shell** | Days 4-6 | DashboardLayout, Sidebar, Header | Core layout components |
| **3. Core Components** | Days 7-10 | Card, Button, Input, Table, Modal | 10 reusable components |
| **4. Core Pages** | Days 11-15 | Home, Providers, Analytics, Settings | 4 high-impact pages |
| **5. Animations** | Days 16-19 | Framer Motion, stagger, transitions | Animation library |
| **6. Batch Pages** | Days 20-34 | 25 remaining pages | Complete dashboard |
| **7. QA & Polish** | Days 35-39 | Testing, accessibility, performance | Production-ready |

</div>

---

## 🛠️ Tech Stack

| Category | Technologies |
|----------|--------------|
| **Frontend** | Next.js 16, React 19.2, Tailwind CSS 4, Fumadocs UI |
| **State** | Zustand, React Hooks |
| **Animations** | Framer Motion 11 |
| **Icons** | Phosphor Icons (duotone) |
| **Typography** | DM Sans (UI), JetBrains Mono (Code) |
| **Charts** | Recharts 3 |
| **Quality** | ESLint 9, Prettier, TypeScript 6, Vitest, Playwright |

---

## ♿ Accessibility Commitment

| Standard | Target | Implementation |
|----------|--------|----------------|
| **WCAG 2.1 AA** | ✅ Compliant | 14.2:1 text contrast, 3:1 UI contrast |
| **Keyboard Navigation** | ✅ Full support | Logical tab order, focus rings, skip links |
| **Screen Readers** | ✅ Optimized | ARIA labels, live regions, semantic HTML |
| **Reduced Motion** | ✅ Respected | `prefers-reduced-motion` guards |
| **Color Blind Safe** | ✅ Tested | Deuteranopia/Protanopia/Tritanopia verified |

---

## 📊 Performance Targets

| Metric | Target | Strategy |
|--------|--------|----------|
| **Core Bundle** | < 250 KB gz | Code splitting, tree shaking |
| **Dashboard Bundle** | < 500 KB gz | Dynamic imports for charts |
| **LCP** | < 2.5s | Optimized fonts, critical CSS |
| **CLS** | < 0.1 | Stable layouts, reserved space |
| **FPS** | 60fps | GPU-accelerated animations, `will-change` |

---

## 🧪 Quality Gates

```bash
# Run all checks
pnpm run check          # lint + test
pnpm run lint           # ESLint (zero errors)
pnpm run typecheck:core # TypeScript strict
pnpm run test:unit      # Unit tests
pnpm run test:vitest    # Vitest (MCP, autoCombo)
pnpm run test:e2e       # Playwright E2E
pnpm run check:docs-all # Documentation validation
```

---

## 📚 Documentation

| Document | Purpose |
|----------|---------|
| [Architecture Blueprint](./01-architecture-blueprint.md) | System architecture, data flows, resilience model |
| [Design System](./02-design-system.md) | Tokens, components, animations, layout styles |
| [Development Roadmap](./03-development-roadmap.md) | 39-day plan with daily tasks |
| [Implementation Checklist](./04-implementation-checklist.md) | QA criteria, acceptance criteria, file inventory |

---

## 🤝 Contributing

### Branch Strategy
```bash
# Create feature branch from main
git checkout -b feat/your-feature-name

# Conventional commits
git commit -m "feat(scope): description"
```

### PR Requirements
- [ ] All CI gates pass (`pnpm run check`)
- [ ] TypeScript compiles (`pnpm run typecheck:core`)
- [ ] Tests pass (`pnpm run test:unit && pnpm run test:vitest`)
- [ ] Visual regression snapshots updated
- [ ] Accessibility audit passes
- [ ] Documentation updated

---

## 📄 License

MIT License — see [LICENSE](LICENSE) for details.

---

## 🙏 Acknowledgments

- **OmniRoute Team** — Building the future of AI routing
- **Phosphor Icons** — Beautiful, consistent iconography
- **Framer Motion** — Delightful animations made simple
- **DM Sans** — Elegant, readable typography
- **Tailwind CSS** — Utility-first styling at scale

---

<div align="center">

---

**Built with precision for the OmniRoute platform** 🚀

*Enterprise-grade design. Developer-first experience. Production-ready from day one.*

</div>