# OmniRoute Dashboard Redesign - Design System Specification

## 1. Design Vision

**Theme**: Charcoal Metallic + Neon Accents + Neon Ivory Text  
**Philosophy**: True neon aesthetics require emissive light principles. Dark-only mode ensures WCAG 2.1 AA contrast on OLED/LCD screens.  
**Typography**: DM Sans (premium modern, geometric, open counters for small-size readability)  
**Icons**: Phosphor Icons (duotone support, optimal at 16-24px)  
**Animations**: Framer Motion + CSS (GPU-accelerated, reduced-motion compliant)

---

## 2. Color System (40+ CSS Custom Properties)

### 2.1 Charcoal Metallic Base

```css
/* Deep charcoal - almost black */
--color-bg: #0A0E14;

/* Slightly elevated surfaces */
--color-bg-elevated: #111822;

/* Card backgrounds */
--color-bg-card: #151D2B;

/* Metallic accent tones */
--color-metallic-light: #2A3444;    /* Brushed metal highlight */
--color-metallic-mid: #1E2838;      /* Mid metallic tone */
--color-metallic-dark: #141B2A;     /* Dark metallic groove */
--color-metallic-border: #3A4A5C;   /* Metallic edge highlight */
```

### 2.2 Neon Palette (Thin, Bright Borders)

```css
/* Primary neon - bright cyan */
--neon-cyan: #00FFFF;

/* Success/active neon */
--neon-emerald: #00FF88;

/* Warning/attention neon */
--neon-amber: #FFB800;

/* Error/critical neon */
--neon-coral: #FF4E6D;

/* Special/purple neon */
--neon-violet: #B880FF;
```

### 2.3 Neon Ivory Text (Replaces Pure White)

```css
/* Warm bright ivory - primary text */
--neon-ivory: #FFF8E7;

/* Muted text */
--neon-ivory-dim: #E8DCC8;

/* Subtle/placeholder text */
--neon-ivory-subtle: #B8A994;
```

### 2.4 Semantic Mappings

```css
--color-text-main: var(--neon-ivory);
--color-text-primary: var(--neon-ivory);
--color-text-muted: var(--neon-ivory-dim);
--color-text-subtle: var(--neon-ivory-subtle);
--color-border: var(--color-metallic-border);
--color-border-neon: var(--neon-cyan);
--color-surface: var(--color-bg-card);
--color-card: var(--color-bg-card);
--color-sidebar: var(--color-bg-elevated);
```

### 2.5 Glassmorphism

```css
--glass-blur: 20px;
--glass-opacity: 0.15;
--glass-border: 1px solid rgba(255, 255, 255, 0.08);
```

### 2.6 Shadows (Metallic Depth + Neon Glow)

```css
/* Metallic shadows with subtle border */
--shadow-metallic-sm: 0 1px 2px rgba(0, 0, 0, 0.3), 0 0 0 1px var(--color-metallic-border);
--shadow-metallic-md: 0 4px 12px rgba(0, 0, 0, 0.4), 0 0 0 1px var(--color-metallic-border);
--shadow-metallic-lg: 0 12px 28px rgba(0, 0, 0, 0.5), 0 0 0 1px var(--color-metallic-border);

/* Neon glow for live/active states */
--shadow-neon-glow: 0 0 20px var(--neon-cyan), 0 0 40px rgba(0, 255, 255, 0.15);
```

### 2.7 Radius Scale

```css
--radius-card: 16px;
--radius-control: 10px;
--radius-full: 9999px;
```

### 2.8 Typography

```css
/* DM Sans - Premium Modern */
--font-sans: 'DM Sans', -apple-system, BlinkMacSystemFont, 'SF Pro Text', system-ui, sans-serif;

/* JetBrains Mono - Code/Data */
--font-mono: 'JetBrains Mono', 'Fira Code', 'SF Mono', monospace;
```

### 2.9 Animation Tokens

```css
--duration-fast: 150ms;
--duration-normal: 250ms;
--duration-slow: 400ms;
--easing-smooth: cubic-bezier(0.4, 0, 0.2, 1);
--easing-spring: cubic-bezier(0.34, 1.56, 0.64, 1);
```

---

## 3. Tailwind Extensions (@theme inline)

```css
@theme inline {
  /* Custom Colors */
  --color-neon-cyan: var(--neon-cyan);
  --color-neon-emerald: var(--neon-emerald);
  --color-neon-amber: var(--neon-amber);
  --color-neon-coral: var(--neon-coral);
  --color-neon-violet: var(--neon-violet);
  --color-neon-ivory: var(--neon-ivory);
  --color-metallic-bg: var(--color-bg);
  --color-metallic-elevated: var(--color-bg-elevated);
  --color-metallic-card: var(--color-bg-card);
  --color-metallic-border: var(--color-metallic-border);
  
  /* Custom Shadows */
  --shadow-metallic-sm: var(--shadow-metallic-sm);
  --shadow-metallic-md: var(--shadow-metallic-md);
  --shadow-metallic-lg: var(--shadow-metallic-lg);
  --shadow-neon-glow: var(--shadow-neon-glow);
  
  /* Custom Animations */
  --animate-stagger-in: staggerIn 0.4s var(--easing-smooth) forwards;
  --animate-neon-pulse: neonPulse 2s ease-in-out infinite;
  --animate-border-glow: borderGlow 2s ease-in-out infinite;
}
```

---

## 4. Component Specifications

### 4.1 Card.tsx

```tsx
// Base card with metallic background + neon border hover
<Card 
  variant="default" | "glass" | "elevated"
  className="bg-metallic-card border-metallic-border 
    hover:border-neon-cyan hover:shadow-neon-glow 
    transition-all duration-300"
/>

// Glass variant
<Card variant="glass" className="bg-metallic-card/70 backdrop-blur-xl border-neon-cyan/20" />

// Elevated variant
<Card variant="elevated" className="bg-metallic-elevated shadow-metallic-lg" />
```

### 4.2 Button.tsx

```tsx
// Primary: Neon gradient + glow
<Button variant="primary" 
  className="bg-gradient-to-r from-neon-cyan to-neon-violet text-neutral-900 
    hover:shadow-neon-glow font-medium px-4 py-2 rounded-control" />

// Secondary: Neon border + text
<Button variant="secondary" 
  className="border-neon-cyan text-neon-cyan hover:bg-neon-cyan/10 
    font-medium px-4 py-2 rounded-control" />

// Ghost: Neon text only
<Button variant="ghost" 
  className="text-neon-ivory hover:text-neon-cyan font-medium px-4 py-2" />

// Danger: Coral accent
<Button variant="danger" 
  className="bg-neon-coral/10 border-neon-coral text-neon-coral 
    hover:bg-neon-coral hover:text-neutral-900" />
```

### 4.3 Input.tsx

```tsx
<Input 
  className="bg-metallic-card border-metallic-border 
    focus:border-neon-cyan focus:ring-2 focus:ring-neon-cyan/30 
    text-neon-ivory placeholder:text-neon-ivory-subtle
    rounded-control px-3 py-2"
  icon={<Icon name="search" className="text-neon-ivory-subtle" />}
/>
```

### 4.4 Select.tsx

```tsx
<Select>
  <SelectTrigger className="bg-metallic-card border-metallic-border 
    focus:border-neon-cyan focus:ring-2 focus:ring-neon-cyan/30 
    text-neon-ivory" />
  <SelectContent className="bg-metallic-card/95 backdrop-blur-xl 
    border-neon-cyan/30 shadow-metallic-lg">
    <SelectItem value="opt" className="text-neon-ivory 
      focus:bg-neon-cyan/10 focus:text-neon-cyan" />
  </SelectContent>
</Select>
```

### 4.5 Table.tsx (New Component)

```tsx
// Header
<thead className="bg-metallic-mid text-neon-ivory border-b border-neon-cyan">
  <th className="px-4 py-3 font-medium">Column</th>
</thead>

// Rows - alternating metallic-card / metallic-elevated
<tbody>
  <tr className="bg-metallic-card border-b border-metallic-border 
    hover:bg-metallic-light hover:border-l-3 hover:border-neon-cyan 
    transition-colors">
    <td className="px-4 py-3 text-neon-ivory">Data</td>
  </tr>
  <tr className="bg-metallic-elevated border-b border-metallic-border 
    hover:bg-metallic-light hover:border-l-3 hover:border-neon-cyan">
    <td className="px-4 py-3 text-neon-ivory">Data</td>
  </tr>
</tbody>

// Selected row
<tr className="border border-neon-cyan bg-neon-cyan/5 shadow-neon-glow">
```

### 4.6 Tabs.tsx

```tsx
// Tab list
<div className="flex border-b border-metallic-border">
  <TabsList className="flex gap-1 bg-transparent">
    <TabsTrigger className="text-neon-ivory-dim hover:text-neon-ivory 
      data-[state=active]:text-neon-cyan data-[state=active]:font-medium 
      px-4 py-2 rounded-control transition-colors" />
  </TabsList>
</div>

// Active indicator (neon gradient line)
<div className="absolute bottom-0 left-0 h-1 bg-gradient-to-r 
  from-neon-cyan to-neon-violet transition-transform duration-300" />
```

### 4.7 Modal.tsx

```tsx
// Overlay
<div className="fixed inset-0 bg-black/60 backdrop-blur-sm z-50" />

// Panel
<div className="bg-metallic-card/95 backdrop-blur-xl 
  border border-neon-cyan/30 shadow-metallic-lg rounded-card 
  max-w-lg w-full mx-4">
  <div className="p-6 border-b border-metallic-border">
    <h2 className="text-neon-ivory font-semibold text-lg">Title</h2>
  </div>
  <div className="p-6">Content</div>
  <div className="p-6 border-t border-metallic-border flex justify-end gap-2">
    <Button variant="ghost">Cancel</Button>
    <Button variant="primary">Confirm</Button>
  </div>
</div>
```

### 4.8 Badge.tsx

```tsx
// Semantic variants with neon text + subtle bg
const variants = {
  info:    "text-neon-cyan    bg-neon-cyan/10    border-neon-cyan/20",
  success: "text-neon-emerald bg-neon-emerald/10 border-neon-emerald/20",
  warning: "text-neon-amber   bg-neon-amber/10   border-neon-amber/20",
  error:   "text-neon-coral   bg-neon-coral/10   border-neon-coral/20",
  default: "text-neon-ivory-dim bg-metallic-border border-metallic-border"
};

// Usage
<Badge variant="success" className="px-2.5 py-0.5 rounded-full text-xs font-medium">
  Connected
</Badge>
```

### 4.9 Avatar.tsx

```tsx
<Avatar 
  className="border-2 border-neon-cyan/50"
  src={url}
  fallback={<span className="text-neon-ivory">AB</span>}
  size="md" // sm (32px), md (40px), lg (48px)
/>
```

### 4.10 Toast.tsx

```tsx
// Semantic left border
const toastVariants = {
  info:    "border-l-4 border-neon-cyan",
  success: "border-l-4 border-neon-emerald",
  warning: "border-l-4 border-neon-amber",
  error:   "border-l-4 border-neon-coral",
};

// Panel
<div className="bg-metallic-card/95 backdrop-blur-xl 
  border border-metallic-border shadow-metallic-lg 
  rounded-card p-4 min-w-[300px] max-w-md">
  <div className="flex items-start gap-3">
    <Icon className="text-neon-cyan shrink-0" />
    <div className="flex-1">
      <p className="text-neon-ivory font-medium">Title</p>
      <p className="text-neon-ivory-dim text-sm">Message</p>
    </div>
  </div>
</div>
```

---

## 5. Animation System (Framer Motion)

### 5.1 Animation Utilities (`src/shared/lib/animations.ts`)

```typescript
import { Variants } from 'framer-motion';

export const staggerContainer: Variants = {
  hidden: { opacity: 0 },
  show: { opacity: 1, transition: { staggerChildren: 0.05 } }
};

export const cardReveal: Variants = {
  hidden: { opacity: 0, y: 20 },
  show: { opacity: 1, y: 0, transition: { duration: 0.4, ease: 'expo.out' } }
};

export const scrollReveal: Variants = {
  hidden: { opacity: 0, y: 16 },
  show: { opacity: 1, y: 0, transition: { duration: 0.5, ease: 'power1.out' } }
};

export const neonPulse = {
  animate: { 
    boxShadow: ['0 0 5px var(--neon-cyan)', '0 0 20px var(--neon-cyan)'],
    transition: { duration: 2, repeat: Infinity, repeatType: 'mirror' }
  }
};

export const magneticButton = {
  whileHover: { scale: 1.02, transition: { duration: 0.15 } },
  whileTap: { scale: 0.98, transition: { duration: 0.1 } }
};

export const pageTransition = {
  initial: { opacity: 0, y: 10 },
  animate: { opacity: 1, y: 0, transition: { duration: 0.4, ease: 'expo.out' } },
  exit: { opacity: 0, y: -10, transition: { duration: 0.2, ease: 'power1.in' } }
};

export const useReducedMotion = () => {
  const [reduce, setReduce] = useState(false);
  useEffect(() => {
    const media = window.matchMedia('(prefers-reduced-motion: reduce)');
    setReduce(media.matches);
    const handler = (e: MediaQueryListEvent) => setReduce(e.matches);
    media.addEventListener('change', handler);
    return () => media.removeEventListener('change', handler);
  }, []);
  return reduce;
};
```

### 5.2 Usage Patterns

```tsx
// Card grid with stagger
<motion.div variants={staggerContainer} className="grid gap-4 md:grid-cols-2 lg:grid-cols-4">
  {items.map(item => (
    <motion.div key={item.id} variants={cardReveal} className="group">
      <Card>...</Card>
    </motion.div>
  ))}
</motion.div>

// Page transitions
<AnimatePresence mode="wait">
  <motion.div 
    key={pathname} 
    variants={pageTransition}
    initial="initial" 
    animate="animate" 
    exit="exit"
  >
    {children}
  </motion.div>
</AnimatePresence>

// Neon pulse for live indicators
<motion.span 
  animate={neonPulse} 
  className="size-2 rounded-full bg-neon-emerald"
  style={{ willChange: 'box-shadow' }}
/>

// Magnetic button
<motion.button 
  whileHover={magneticButton.whileHover}
  whileTap={magneticButton.whileTap}
  className="..."
>
  Click me
</motion.button>
```

---

## 6. Layout-Specific Styles

### 6.1 DashboardLayout.tsx

```tsx
// Body with metallic grid wallpaper
body::before {
  content: "";
  position: fixed;
  inset: 0;
  z-index: -1;
  pointer-events: none;
  background-image:
    linear-gradient(to right, var(--color-metallic-border) 1px, transparent 1px),
    linear-gradient(to bottom, var(--color-metallic-border) 1px, transparent 1px);
  background-size: 32px 32px;
  opacity: 0.08;
}

// Main content wrapper with glassmorphism
<main className="relative flex min-h-0 flex-1 min-w-0 flex-col">
  <div className="flex-1 min-h-0 overflow-y-auto overflow-x-hidden custom-scrollbar p-4 sm:p-6 lg:p-10">
    <div className="max-w-[3840px] mx-auto w-full h-full min-h-0 flex flex-col">
      <Breadcrumbs />
      <div className="flex-1 min-h-0">{children}</div>
    </div>
  </div>
</main>
```

### 6.2 Sidebar.tsx (Key Changes)

```tsx
// Background gradient
<aside className="bg-gradient-to-b from-metallic-elevated to-metallic-bg 
  border-r border-metallic-border transition-all duration-300" />

// Active navigation item
<Link className={cn(
  "border-l-3 border-neon-cyan bg-neon-cyan/5 text-neon-ivory shadow-neon-glow",
  "hover:bg-neon-cyan/10"
)} />

// Icons
<span className="text-neon-ivory group-hover:text-neon-cyan transition-colors" />

// Section titles
<span className="text-neon-ivory-subtle uppercase tracking-wider text-[10px]" />

// Collapsed tooltip
<div className="bg-metallic-card/90 backdrop-blur-xl border border-neon-cyan/30 
  rounded-md px-2.5 py-1.5 text-neon-ivory text-xs font-medium shadow-lg" />
```

### 6.3 Header.tsx

```tsx
// Glassmorphism header
<header className="sticky top-0 z-10 flex items-center justify-between 
  border-b border-metallic-border bg-metallic-elevated/80 backdrop-blur-xl px-8 py-4">

// Neon bottom accent line
<div className="absolute bottom-0 left-0 right-0 h-0.5 
  bg-gradient-to-r from-transparent via-neon-cyan/50 to-transparent" />

// Title text
<h1 className="text-xl font-semibold text-neon-ivory tracking-tight" />

// Focus rings on all interactive elements
<button className="focus-visible:outline-none focus-visible:ring-4 
  focus-visible:ring-neon-cyan focus-visible:ring-offset-2 
  focus-visible:ring-offset-metallic-bg" />
```

---

## 7. Accessibility Checklist

| Criteria | Target | Implementation |
|----------|--------|----------------|
| **Text Contrast** | 14.2:1 | Neon Ivory (#FFF8E7) on Charcoal (#0A0E14) |
| **Border/Icon Contrast** | 3:1+ | Neon colors on metallic backgrounds |
| **Focus Rings** | 4px visible | Neon cyan ring on all focusable |
| **Reduced Motion** | Instant fallback | `useReducedMotion()` guard on all Framer Motion |
| **Color Blind Safe** | Tested | Deuteranopia/Protanopia simulators |
| **Touch Targets** | ≥44×44pt | Minimum sizing on all interactive |
| **Keyboard** | Logical order | Skip links, focus management |

---

## 8. Performance Guidelines

| Concern | Mitigation |
|---------|------------|
| **Bundle Size** | Framer Motion ~25KB gz, Phosphor tree-shakeable |
| **Blur Performance** | Limit glassmorphism to key surfaces, `will-change: backdrop-filter` |
| **Neon Animations** | GPU-accelerated (`will-change: box-shadow, transform`) |
| **Large Lists** | Only animate mounted items (virtualization compatible) |
| **Dark Mode Only** | Single theme reduces CSS complexity |

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