# Landing Page Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Redesign the BreedSmart landing page to use glassmorphic design, Geist typography, and scroll-driven parallax — unifying it with the dashboard's visual identity.

**Architecture:** Replace the existing organic/nature-themed components in `breeding-landing/` with new glassmorphic components. Rewrite `globals.css` with dashboard-aligned design tokens, swap fonts from Fraunces/Outfit to Geist, remove 4 sections (Problems, Workflow, Testimonials, FloatingElements), and restyle the remaining 5 sections + footer. The AI section becomes a floating dark rounded card instead of a full-bleed dark section.

**Tech Stack:** Next.js 16, React 19, Tailwind CSS 4, Framer Motion 12, Lucide React, TypeScript

**Spec:** `docs/superpowers/specs/2026-03-25-landing-page-redesign-design.md`
**Visual mockup:** `.superpowers/brainstorm/31296-1774371263/final-design.html`

---

## File Map

| Action | File | Responsibility |
|--------|------|----------------|
| Rewrite | `src/app/globals.css` | Design tokens, glassmorphic variables, remove organic styles |
| Modify | `src/app/layout.tsx` | Swap Fraunces/Outfit fonts for Geist |
| Rewrite | `src/app/page.tsx` | Remove deleted sections, update imports |
| Rewrite | `src/components/navbar.tsx` | Glassmorphic navbar with scroll-triggered opacity |
| Rewrite | `src/components/hero.tsx` | Full-bleed photo hero with parallax scroll + scale |
| Rewrite | `src/components/features.tsx` | Glassmorphic card grid |
| Rewrite | `src/components/ai-section.tsx` | Floating dark card with chat mockup |
| Rewrite | `src/components/stats.tsx` | Glass cards with count-up animation |
| Rewrite | `src/components/cta-section.tsx` | Centered text on warm background |
| Rewrite | `src/components/footer.tsx` | Dark minimal footer |
| Modify | `src/lib/animations.ts` | Remove `useMouseParallax`, add hero parallax variants |
| Rewrite | `src/lib/i18n.ts` | Simplified translations for reduced sections |
| Delete | `src/components/problems.tsx` | No longer needed |
| Delete | `src/components/workflow.tsx` | No longer needed |
| Delete | `src/components/testimonials.tsx` | No longer needed |
| Delete | `src/components/floating-elements.tsx` | No longer needed |
| Create | `public/images/` | Directory for user-provided photos |

---

### Task 1: Foundation — Fonts, Tokens, and Cleanup

**Files:**
- Modify: `src/app/layout.tsx` (lines 1-51)
- Rewrite: `src/app/globals.css` (207 lines → ~80 lines)
- Modify: `src/lib/animations.ts` (remove `useMouseParallax` at lines 152-172)
- Delete: `src/components/problems.tsx`
- Delete: `src/components/workflow.tsx`
- Delete: `src/components/testimonials.tsx`
- Delete: `src/components/floating-elements.tsx`

- [ ] **Step 1: Replace fonts in layout.tsx**

Replace Fraunces and Outfit imports with Geist Sans. Update the `<body>` className to use the new font variable. Keep metadata but update description if needed.

```tsx
// Replace font imports at top of layout.tsx
import { Geist, Geist_Mono } from "next/font/google";

const geistSans = Geist({
  variable: "--font-geist-sans",
  subsets: ["latin"],
});

const geistMono = Geist_Mono({
  variable: "--font-geist-mono",
  subsets: ["latin"],
});

// In the body tag:
<body className={`${geistSans.variable} ${geistMono.variable} antialiased`}>
```

- [ ] **Step 2: Rewrite globals.css with glassmorphic design tokens**

Replace the entire file. Remove all organic colors (sage, terracotta, earth, forest scales), blob animations, noise textures, leaf patterns, mesh gradients. Add dashboard-aligned CSS variables:

```css
@import "tailwindcss";

:root {
  /* Background */
  --background: #f0eeeb;
  --background-gradient: linear-gradient(135deg, #f0eeeb 0%, #e8e6f0 40%, #f2eff5 70%, #eef0eb 100%);

  /* Text */
  --text-primary: #1a1a1a;
  --text-muted: #aaa;
  --text-on-dark: #e8e8e8;

  /* Accent */
  --accent-green: #3d8c5c;
  --accent-green-dark: #2d6b44;
  --accent-green-light: #5cb87a;
  --accent-green-hero: #7ddfaa;

  /* Glass */
  --glass-bg: rgba(255, 255, 255, 0.75);
  --glass-border: rgba(255, 255, 255, 0.6);
  --glass-shadow: 0 4px 20px rgba(0, 0, 0, 0.04);
  --glass-blur: 20px;

  /* Dark card (AI section) */
  --dark-bg-start: #161622;
  --dark-bg-mid: #111828;
  --dark-glass-bg: rgba(255, 255, 255, 0.035);
  --dark-glass-border: rgba(255, 255, 255, 0.06);

  /* Radius */
  --radius-card: 18px;
  --radius-button: 12px;
  --radius-navbar: 16px;
  --radius-icon: 11px;
  --radius-ai-card: 28px;

  /* Font */
  font-family: var(--font-geist-sans), system-ui, -apple-system, sans-serif;
}

html {
  scroll-behavior: smooth;
}

body {
  background: var(--background);
  color: var(--text-primary);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

/* Custom scrollbar */
::-webkit-scrollbar { width: 8px; }
::-webkit-scrollbar-track { background: transparent; }
::-webkit-scrollbar-thumb {
  background: rgba(61, 140, 92, 0.2);
  border-radius: 4px;
}
::-webkit-scrollbar-thumb:hover { background: rgba(61, 140, 92, 0.35); }

::selection {
  background-color: rgba(61, 140, 92, 0.15);
  color: var(--text-primary);
}
```

- [ ] **Step 3: Remove useMouseParallax from animations.ts**

Delete the `useMouseParallax` hook (lines 152-172) and its import of `useRef, useState, useEffect` if no longer used by other hooks. Keep `useCountUp` and `useScrollProgress`. Also update `staggerContainer` to use `staggerChildren: 0.1` (from 0.12) to match spec.

- [ ] **Step 4: Delete removed component files**

```bash
cd breeding-landing
rm src/components/problems.tsx
rm src/components/workflow.tsx
rm src/components/testimonials.tsx
rm src/components/floating-elements.tsx
```

- [ ] **Step 5: Create public/images directory**

```bash
mkdir -p public/images
```

Add a placeholder note: the user will add their photos (hero-chick.webp, etc.) here.

- [ ] **Step 6: Verify build passes**

```bash
cd breeding-landing && npm run build
```

This will fail because `page.tsx` still imports deleted components — that's expected. Verify the error is only about missing imports, not CSS or font issues.

- [ ] **Step 7: Commit**

```bash
git add -A && git commit -m "refactor(landing): replace design tokens and fonts, remove organic theme

- Swap Fraunces/Outfit fonts for Geist Sans/Mono
- Rewrite globals.css with glassmorphic design tokens
- Remove useMouseParallax hook
- Delete Problems, Workflow, Testimonials, FloatingElements components
- Create public/images/ for photo assets"
```

---

### Task 2: Page Shell and i18n

**Files:**
- Rewrite: `src/app/page.tsx` (34 lines)
- Rewrite: `src/lib/i18n.ts` (448 lines → ~250 lines)

- [ ] **Step 1: Rewrite page.tsx with new section structure**

```tsx
"use client";

import { LanguageProvider } from "@/components/language-provider";
import { Navbar } from "@/components/navbar";
import { Hero } from "@/components/hero";
import { Features } from "@/components/features";
import { AISection } from "@/components/ai-section";
import { Stats } from "@/components/stats";
import { CTASection } from "@/components/cta-section";
import { Footer } from "@/components/footer";

export default function Home() {
  return (
    <LanguageProvider>
      <Navbar />
      <main>
        <Hero />
        <Features />
        <AISection />
        <Stats />
        <CTASection />
      </main>
      <Footer />
    </LanguageProvider>
  );
}
```

- [ ] **Step 2: Rewrite i18n.ts with simplified translations**

Remove all translations for deleted sections (problems, workflow, testimonials). Update remaining section translations to match new copy from the spec/mockup. Keep both `id` and `en` locales. Key structure:

```typescript
export type Locale = "id" | "en";

export const translations = {
  id: {
    nav: {
      features: "Fitur",
      ai: "AI",
      about: "Tentang",
      cta: "Mulai Gratis",
      langToggle: "ID / EN",
    },
    hero: {
      badge: "MANAJEMEN PETERNAKAN, REIMAGINED",
      headline: "Kelola peternakan",
      headlineAccent: "dengan jelas",
      subheadline: "ERP lengkap untuk operasi peternakan unggas dan ternak modern. Dari kandang hingga keuangan.",
      cta: "Mulai Uji Coba Gratis",
      ctaSecondary: "Lihat Demo",
      scroll: "Scroll",
    },
    features: {
      label: "YANG ANDA DAPATKAN",
      title: "Semua untuk menjalankan operasi Anda.",
      titleMuted: "Tidak ada yang tidak Anda butuhkan.",
      items: [
        { icon: "bird", title: "Manajemen Flock", description: "Lacak batch, catatan kesehatan, mortalitas, dan metrik pertumbuhan di semua peternakan secara real-time." },
        { icon: "package", title: "Kontrol Inventaris", description: "Pelacakan pakan otomatis, manajemen pasokan, dan peringatan pemesanan ulang." },
        { icon: "shopping-cart", title: "Penjualan & Pesanan", description: "Manajemen pesanan dari PO pelanggan hingga pengiriman. Faktur dan pembayaran terintegrasi." },
        { icon: "factory", title: "Pengolahan", description: "Kelola jadwal pemotongan, pelacakan hasil, dan output produk dengan ketertelusuran penuh." },
        { icon: "bar-chart-3", title: "Analitik", description: "Dashboard real-time, benchmark kinerja, dan analisis tren di seluruh operasi Anda." },
        { icon: "wallet", title: "Keuangan", description: "Pelacakan biaya, analisis margin, dan laporan keuangan yang dibangun khusus untuk ekonomi peternakan." },
      ],
    },
    ai: {
      label: "AI-POWERED",
      title: "Penasihat peternakan",
      titleAccent: "tercerdas Anda",
      description: "Ajukan pertanyaan dalam bahasa sehari-hari. Dapatkan wawasan dari data Anda secara instan. Tanpa SQL, tanpa laporan — cukup jawaban.",
      features: [
        "Kueri bahasa alami di seluruh data peternakan Anda",
        "Peringatan prediktif untuk mortalitas dan efisiensi pakan",
        "Laporan kinerja otomatis dan benchmarking",
        "Saran optimasi biaya berdasarkan data Anda",
      ],
      chatName: "BreedSmart AI",
      chatStatus: "Online",
      chatMessages: [
        { role: "user", text: "Bagaimana tren FCR untuk Kandang A3 dalam 3 batch terakhir?" },
        { role: "ai", text: "FCR untuk Kandang A3 meningkat secara stabil:\n\nBatch 12: 1.72\nBatch 13: 1.65\nBatch 14: 1.58\n\nItu peningkatan 8.1%. Pergantian pakan di Batch 13 tampaknya menjadi titik infleksi." },
        { role: "user", text: "Bandingkan dengan rata-rata peternakan" },
        { role: "ai", text: "Rata-rata FCR peternakan adalah 1.68. Kandang A3 berkinerja 5.9% lebih baik dari rata-rata. Ini kandang terbaik Anda kuartal ini." },
      ],
    },
    stats: {
      label: "DIPERCAYA PETERNAK",
      title: "Angka yang",
      titleAccent: "berbicara",
      items: [
        { value: "500", suffix: "+", label: "Peternak Aktif" },
        { value: "2K", suffix: "+", label: "Batch Dikelola" },
        { value: "Rp 15M", suffix: "+", label: "Transaksi" },
        { value: "99.9", suffix: "%", label: "Uptime" },
      ],
    },
    cta: {
      title: "Siap mengelola peternakan",
      titleAccent: "lebih cerdas?",
      subtitle: "Mulai uji coba gratis hari ini. Tanpa kartu kredit. Akses penuh ke semua fitur.",
      button: "Mulai Gratis",
    },
    footer: {
      tagline: "Manajemen peternakan cerdas untuk peternak modern.",
      product: "Produk",
      company: "Perusahaan",
      legal: "Legal",
      links: {
        features: "Fitur",
        aiAssistant: "Asisten AI",
        pricing: "Harga",
        about: "Tentang",
        blog: "Blog",
        contact: "Kontak",
        privacy: "Privasi",
        terms: "Ketentuan",
      },
      copyright: "© 2026 BreedSmart. Hak cipta dilindungi.",
    },
  },
  en: {
    nav: {
      features: "Features",
      ai: "AI",
      about: "About",
      cta: "Get Started",
      langToggle: "ID / EN",
    },
    hero: {
      badge: "FARM MANAGEMENT, REIMAGINED",
      headline: "Run your farm",
      headlineAccent: "with clarity",
      subheadline: "The complete ERP for modern poultry and livestock breeding operations. From flock to finance.",
      cta: "Start Free Trial",
      ctaSecondary: "Watch Demo",
      scroll: "Scroll",
    },
    features: {
      label: "WHAT YOU GET",
      title: "Everything to run your operation.",
      titleMuted: "Nothing you don't need.",
      items: [
        { icon: "bird", title: "Flock Management", description: "Track batches, health records, mortality, and growth metrics across all your farms in real-time." },
        { icon: "package", title: "Inventory Control", description: "Automated feed tracking, supply management, and reorder alerts. Never run out unexpectedly." },
        { icon: "shopping-cart", title: "Sales & Orders", description: "End-to-end order management from customer PO to delivery. Integrated invoicing and payments." },
        { icon: "factory", title: "Processing", description: "Manage slaughter schedules, yield tracking, and product output with full traceability." },
        { icon: "bar-chart-3", title: "Analytics", description: "Real-time dashboards, performance benchmarks, and trend analysis across your entire operation." },
        { icon: "wallet", title: "Finance", description: "Cost tracking, margin analysis, and financial reporting built specifically for breeding economics." },
      ],
    },
    ai: {
      label: "AI-POWERED",
      title: "Your smartest",
      titleAccent: "farm advisor",
      description: "Ask questions in plain language. Get insights from your data instantly. No SQL, no reports — just answers.",
      features: [
        "Natural language queries across all your farm data",
        "Predictive alerts for mortality and feed efficiency",
        "Automated performance reports and benchmarking",
        "Cost optimization suggestions based on your data",
      ],
      chatName: "BreedSmart AI",
      chatStatus: "Online",
      chatMessages: [
        { role: "user", text: "What's the FCR trend for Coop A3 over the last 3 batches?" },
        { role: "ai", text: "FCR for Coop A3 has improved steadily:\n\nBatch 12: 1.72\nBatch 13: 1.65\nBatch 14: 1.58\n\nThat's an 8.1% improvement. The feed switch in Batch 13 appears to be the inflection point." },
        { role: "user", text: "Compare that to the farm average" },
        { role: "ai", text: "Farm average FCR is 1.68. Coop A3 is performing 5.9% better than average. It's your best-performing coop this quarter." },
      ],
    },
    stats: {
      label: "TRUSTED BY BREEDERS",
      title: "Numbers that",
      titleAccent: "speak",
      items: [
        { value: "500", suffix: "+", label: "Active Breeders" },
        { value: "2K", suffix: "+", label: "Batches Managed" },
        { value: "$1M", suffix: "+", label: "Transactions" },
        { value: "99.9", suffix: "%", label: "Uptime" },
      ],
    },
    cta: {
      title: "Ready to run your farm",
      titleAccent: "smarter?",
      subtitle: "Start your free trial today. No credit card required. Full access to all features.",
      button: "Get Started Free",
    },
    footer: {
      tagline: "Smart farm management for modern breeders.",
      product: "Product",
      company: "Company",
      legal: "Legal",
      links: {
        features: "Features",
        aiAssistant: "AI Assistant",
        pricing: "Pricing",
        about: "About",
        blog: "Blog",
        contact: "Contact",
        privacy: "Privacy",
        terms: "Terms",
      },
      copyright: "© 2026 BreedSmart. All rights reserved.",
    },
  },
} as const;
```

- [ ] **Step 3: Verify TypeScript compiles (build will still fail — components not yet rewritten)**

```bash
cd breeding-landing && npx tsc --noEmit 2>&1 | head -20
```

Expect errors only from component files that haven't been rewritten yet, not from i18n or page.tsx.

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "refactor(landing): rewrite page shell and i18n for new section structure

- Simplify page.tsx to 5 sections (Hero, Features, AI, Stats, CTA)
- Rewrite i18n with updated copy for glassmorphic redesign
- Remove translation keys for deleted sections"
```

---

### Task 3: Navbar Component

**Files:**
- Rewrite: `src/components/navbar.tsx` (183 lines)

- [ ] **Step 1: Rewrite navbar.tsx**

Glassmorphic fixed navbar with scroll-triggered opacity change. Uses Framer Motion `useScroll` + `useMotionValueEvent` for scroll detection. Includes language toggle from `useLanguage()` hook. Nav links: Features (#features), AI (#ai), About (#stats). Green gradient CTA button.

Key styles (Tailwind):
- Container: `fixed top-4 left-1/2 -translate-x-1/2 w-[calc(100%-32px)] max-w-[1200px] z-50`
- Glass: `bg-white/[0.12] border border-white/20 rounded-[16px] backdrop-blur-[24px]`
- Scrolled state: `bg-white/85 border-white/60 shadow-[0_4px_20px_rgba(0,0,0,0.06)]`
- Logo icon: `w-8 h-8 bg-gradient-to-br from-[#3d8c5c] to-[#2d6b44] rounded-[9px]`
- CTA button: `bg-gradient-to-br from-[#3d8c5c] to-[#2d6b44] rounded-[10px] shadow-[0_2px_12px_rgba(61,140,92,0.35)]`
- Mobile: hamburger menu with `AnimatePresence` for mobile nav overlay

Reference the mockup at `.superpowers/brainstorm/31296-1774371263/final-design.html` for exact styling.

- [ ] **Step 2: Verify it renders**

```bash
cd breeding-landing && npm run dev
```

Open browser, check navbar appears with glass effect and scroll behavior works.

- [ ] **Step 3: Commit**

```bash
git add src/components/navbar.tsx && git commit -m "feat(landing): glassmorphic navbar with scroll-triggered opacity"
```

---

### Task 4: Hero Section

**Files:**
- Rewrite: `src/components/hero.tsx` (412 lines → ~120 lines)
- Modify: `src/lib/animations.ts` (add hero parallax)

- [ ] **Step 1: Add hero parallax animation to animations.ts**

Add a `heroFadeIn` variant and export it:

```typescript
export const heroFadeIn = {
  hidden: { opacity: 0, y: 30 },
  visible: {
    opacity: 1,
    y: 0,
    transition: { duration: 1.2, ease: [0.22, 1, 0.36, 1] },
  },
};
```

- [ ] **Step 2: Rewrite hero.tsx**

Full-bleed photo background with Framer Motion parallax scroll. Uses `useScroll` + `useTransform` for:
- Background `translateY` at 0.3x scroll speed
- Background scale from 1.0 → 1.1 over first viewport

Text content at bottom-left with gradient overlay for readability. Badge, headline with accent color, subheadline, two CTA buttons, scroll hint.

For now, use a placeholder gradient background (since user photos aren't added yet). Add a comment showing where to put the `<Image>` or background URL once photos are available.

Key structure:
```tsx
export function Hero() {
  const { ref: sectionRef } = ... // section ref
  const { scrollYProgress } = useScroll({ target: sectionRef, offset: ["start start", "end start"] });
  const bgY = useTransform(scrollYProgress, [0, 1], ["0%", "30%"]);
  const bgScale = useTransform(scrollYProgress, [0, 1], [1, 1.1]);
  // ... render with motion.div for parallax bg
}
```

Reference the mockup for exact styling — bottom-left text alignment, badge glass pill, CTA button styles, scroll hint animation.

- [ ] **Step 3: Verify hero renders with parallax effect**

```bash
cd breeding-landing && npm run dev
```

Check: hero fills viewport, text at bottom-left, scroll creates parallax motion on background.

- [ ] **Step 4: Commit**

```bash
git add src/components/hero.tsx src/lib/animations.ts && git commit -m "feat(landing): full-bleed hero with parallax scroll and scale effect"
```

---

### Task 5: Features Section

**Files:**
- Rewrite: `src/components/features.tsx` (158 lines → ~100 lines)

- [ ] **Step 1: Rewrite features.tsx**

Glassmorphic card grid with 6 feature cards. Uses `staggerContainer` and `staggerItem` variants from animations.ts. Section label + heading + 3x2 responsive grid.

Key styles:
- Section bg: `bg-gradient-to-b from-[#f0eeeb] to-[#e8e6f0]`
- Cards: `bg-white/75 border border-white/60 rounded-[18px] backdrop-blur-[20px] shadow-[0_4px_20px_rgba(0,0,0,0.04)]`
- Card hover: `hover:-translate-y-1.5 hover:shadow-[0_12px_40px_rgba(0,0,0,0.08)]`
- Grid: `grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4`
- Icon container: 42x42px with gradient bg, rounded-[11px]
- Section label: `text-[10px] uppercase tracking-[3px] text-black/25 font-medium`
- Heading: `text-[clamp(30px,4.5vw,50px)] font-light tracking-[-2px]`

Map icons from Lucide: Bird, Package, ShoppingCart, Factory, BarChart3, Wallet.

Use `whileInView` with `viewport: { once: true, margin: "-100px" }` for scroll-triggered stagger animation.

- [ ] **Step 2: Verify features render**

```bash
cd breeding-landing && npm run dev
```

Check: 6 glass cards, stagger animation on scroll, hover effects, responsive grid.

- [ ] **Step 3: Commit**

```bash
git add src/components/features.tsx && git commit -m "feat(landing): glassmorphic feature cards with stagger animation"
```

---

### Task 6: AI Section (Floating Dark Card)

**Files:**
- Rewrite: `src/components/ai-section.tsx` (211 lines → ~180 lines)

- [ ] **Step 1: Rewrite ai-section.tsx**

This is the most complex section. Structure:
- **Outer wrapper:** warm background matching page, centers the dark card
- **Dark card:** max-w-[1600px], rounded-[28px], dark gradient bg, inset border, shadow, radial glows
- **Inner grid:** 2 columns — left (text + features list), right (chat mockup)

Left side: section label, heading with accent, description, 4 AI feature items in dark glass cards.

Right side: Chat mockup with header (avatar, name, status), and chat messages (user/ai bubbles). Parse `\n` in chat messages to `<br>` tags. Bold text with `<strong>`.

Animation: Use `slideFromLeft` for left content, `slideFromRight` for chat mockup (both from animations.ts).

Key styles:
- Wrapper: `bg-[#e8e6f0] px-5 flex justify-center`
- Card: `max-w-[1600px] w-full rounded-[28px] bg-gradient-to-b from-[#161622] via-[#111828] to-[#161622] shadow-[0_8px_60px_rgba(0,0,0,0.15)] p-[60px_60px] md:p-[80px_60px]`
- AI feature items: `bg-white/[0.035] border border-white/[0.055] rounded-[14px] backdrop-blur-[16px]`
- Chat mockup: `bg-white/[0.035] border border-white/[0.06] rounded-[22px] backdrop-blur-[20px] shadow-[0_8px_48px_rgba(0,0,0,0.3)]`
- User bubble: `bg-[rgba(61,140,92,0.14)] rounded-[16px] rounded-br-[5px]`
- AI bubble: `bg-white/[0.055] rounded-[16px] rounded-bl-[5px]`

Responsive: On mobile (`md:` breakpoint), stack to single column with chat below text.

- [ ] **Step 2: Verify AI section renders**

```bash
cd breeding-landing && npm run dev
```

Check: floating dark card with rounded corners, warm background visible around it, two-column layout, chat mockup functional, responsive stacking.

- [ ] **Step 3: Commit**

```bash
git add src/components/ai-section.tsx && git commit -m "feat(landing): floating dark glassmorphic AI section with chat mockup"
```

---

### Task 7: Stats Section

**Files:**
- Rewrite: `src/components/stats.tsx` (91 lines → ~80 lines)

- [ ] **Step 1: Rewrite stats.tsx**

Centered layout with section label, heading, and 4 glassmorphic stat cards. Uses existing `useCountUp` hook from animations.ts for animated numbers.

Key styles:
- Section: `min-h-screen bg-gradient-to-b from-[#f0eeeb] via-[#e8e6f0] to-[#f0eeeb] flex flex-col justify-center items-center text-center`
- Grid: `grid grid-cols-2 lg:grid-cols-4 gap-5 max-w-[1000px] w-full`
- Cards: Same glass style as feature cards (18px radius, backdrop blur)
- Number: `text-[44px] font-light tracking-[-2px]`
- Suffix (+ / %): `text-[#3d8c5c]`
- Label: `text-[10px] uppercase tracking-[1.5px] text-[#aaa] font-medium`

Each stat card uses `useCountUp` with the numeric portion of the value. The suffix (+, %) is rendered separately in green.

- [ ] **Step 2: Verify stats render with count-up**

```bash
cd breeding-landing && npm run dev
```

Check: 4 glass cards, numbers animate on scroll, green accent on suffixes.

- [ ] **Step 3: Commit**

```bash
git add src/components/stats.tsx && git commit -m "feat(landing): glassmorphic stats section with count-up animation"
```

---

### Task 8: CTA Section and Footer

**Files:**
- Rewrite: `src/components/cta-section.tsx` (127 lines → ~50 lines)
- Rewrite: `src/components/footer.tsx` (127 lines → ~80 lines)

- [ ] **Step 1: Rewrite cta-section.tsx**

Simple centered section on warm background. Heading with accent, subheadline, single CTA button. Fade-in animation on scroll.

Key styles:
- Section: `min-h-[70vh] bg-gradient-to-b from-[#e8e6f0] via-[#ebe9f0] to-[#f0eeeb] flex flex-col justify-center items-center text-center`
- Heading: `text-[clamp(34px,5.5vw,64px)] font-light tracking-[-2px] text-[#1a1a1a] text-center`
- Accent: `text-[#3d8c5c] font-normal`
- Subheadline: `text-base text-[#999] max-w-[380px] mx-auto`
- Button: `bg-gradient-to-br from-[#3d8c5c] to-[#2d6b44] rounded-[12px] px-11 py-4 shadow-[0_4px_24px_rgba(61,140,92,0.4)]`

- [ ] **Step 2: Rewrite footer.tsx**

Dark footer with logo/tagline on left, 3 link columns on right.

Key styles:
- Section: `bg-[#0e0e0e] py-14 px-10`
- Logo icon: same green gradient 28x28
- Group headers: `text-[10px] uppercase tracking-[1.5px] text-white/25 font-medium`
- Links: `text-xs text-white/45 hover:text-white/70`
- Copyright: `border-t border-white/[0.06] text-[11px] text-white/[0.18]`

- [ ] **Step 3: Verify full page renders end-to-end**

```bash
cd breeding-landing && npm run dev
```

Scroll through entire page: Hero → Features → AI (floating card) → Stats → CTA → Footer. Check all sections render, animations trigger, i18n toggle works.

- [ ] **Step 4: Commit**

```bash
git add src/components/cta-section.tsx src/components/footer.tsx && git commit -m "feat(landing): glassmorphic CTA section and dark footer"
```

---

### Task 9: Responsive Polish and Build Verification

**Files:**
- Modify: All component files as needed for responsive fixes

- [ ] **Step 1: Test responsive breakpoints**

Open dev tools and test at:
- 1440px+ (desktop)
- 1024px (laptop — 3-col features, 4-col stats, 2-col AI)
- 768px (tablet — 2-col features, 2-col stats, stacked AI)
- 375px (mobile — 1-col everything, stacked CTAs, centered hero text)

Fix any layout breaks. Common issues:
- Hero text should center on mobile
- CTA buttons should stack vertically on mobile
- AI section inner grid should stack
- Feature/stat cards should reflow

- [ ] **Step 2: Test language toggle**

Click ID/EN toggle in navbar. Verify all visible text switches in both languages across all sections.

- [ ] **Step 3: Run production build**

```bash
cd breeding-landing && npm run build
```

Must succeed with zero errors. Static export to `out/` directory.

- [ ] **Step 4: Test static export**

```bash
cd breeding-landing && npx serve out
```

Open in browser, verify all sections render correctly from the static build.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "fix(landing): responsive polish and build verification"
```

---

### Task 10: Final Cleanup

**Files:**
- Verify all files

- [ ] **Step 1: Remove any unused imports across all files**

Check each component file for unused imports (Lucide icons, Framer Motion variants, etc.).

- [ ] **Step 2: Verify no references to deleted components remain**

```bash
grep -r "problems\|workflow\|testimonials\|floating-elements\|FloatingElements\|Problems\|Workflow\|Testimonials" breeding-landing/src/ --include="*.tsx" --include="*.ts"
```

Should return zero results.

- [ ] **Step 3: Verify no old font references remain**

```bash
grep -r "Fraunces\|Outfit\|fraunces\|outfit" breeding-landing/src/ --include="*.tsx" --include="*.ts" --include="*.css"
```

Should return zero results.

- [ ] **Step 4: Final build check**

```bash
cd breeding-landing && npm run build && npm run lint
```

Both must pass cleanly.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "chore(landing): final cleanup — remove unused imports and old references"
```
