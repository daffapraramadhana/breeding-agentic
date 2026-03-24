# Landing Page Redesign — Glassmorphic + Parallax

**Date:** 2026-03-25
**Status:** Draft
**Scope:** `breeding-landing/` — full visual redesign

## Summary

Redesign the BreedSmart landing page to unify its visual identity with the dashboard's glassmorphic design language while adopting a scroll-driven, editorial presentation style inspired by Reasonal (awwwards.com). The current organic/nature-themed landing page (Fraunces font, blobs, noise textures, leaf patterns) will be replaced with the dashboard's glass aesthetic, Geist typography, and sage green palette — but with marketing energy: bold photography, parallax scroll effects, and confident typography.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Design direction | Shared DNA, different energy (B) | Unify brand while keeping landing page's marketing punch |
| Typography | Full Geist (A) | Match dashboard exactly — clean, modern, consistent |
| AI section style | Glassmorphic dark variant (B) | Keeps visual rhythm without breaking glass system |
| Section structure | Curated (removed Problems, Workflow, Testimonials) | Tighter page, each section earns its viewport space |
| Hero style | Full-bleed photography | Chick photo + parallax scroll + scale effect |
| Parallax approach | Subtle scroll speed + scale-on-scroll | Suited for poultry imagery (not layered landscape parallax) |

## Section Architecture

### 1. Navbar (Fixed, Glassmorphic)
- **Position:** Fixed top, centered, max-width 1200px
- **Style:** `rgba(255,255,255,0.12)` background, `border: 1px solid rgba(255,255,255,0.2)`, `backdrop-filter: blur(24px)`, `border-radius: 16px`
- **Content:** Logo (green gradient icon + "BreedSmart"), nav links (Features, AI, About), language toggle (ID/EN), green gradient CTA button
- **Behavior:** Scroll-triggered — increases background opacity after scroll threshold

### 2. Hero (Full Viewport, Photo Background)
- **Background:** Full-bleed chick photo (user-provided), `object-fit: cover`, fills entire viewport
- **Overlay:** Subtle bottom gradient for text readability — `linear-gradient(to bottom, rgba(0,0,0,0.1) 0%, transparent 30%, rgba(0,0,0,0.5) 100%)`
- **Text position:** Bottom-left aligned
- **Badge:** Glass pill — `rgba(255,255,255,0.15)` bg, `backdrop-filter: blur(12px)`, uppercase tracking, "FARM MANAGEMENT, REIMAGINED"
- **Headline:** `clamp(48px, 7vw, 88px)`, `font-weight: 300`, `letter-spacing: -3px`, white text, accent word in `#7ddfaa`
- **Subheadline:** 17px, `rgba(255,255,255,0.75)`, max-width 460px
- **CTAs:** Primary (green gradient, 12px radius, shadow) + Secondary (glass style, white text)
- **Scroll hint:** Bottom-right, animated line pulse
- **Parallax effects (Framer Motion):**
  - Background image: `translateY` at 0.3x scroll speed (moves slower than content)
  - Scale-on-scroll: Image scales from 1.0 to 1.1 over first viewport of scroll
  - Text fade + rise: Headline fades/rises in on page load with spring easing
- **Responsive:** On mobile, text centers, font size scales down via clamp, CTAs stack vertically

### 3. Features (Glassmorphic Card Grid)
- **Background:** Warm gradient — `linear-gradient(180deg, #f0eeeb 0%, #e8e6f0 100%)`
- **Layout:** Section label ("WHAT YOU GET") + heading + 3x2 card grid
- **Heading:** `clamp(32px, 5vw, 52px)`, `font-weight: 300`, `letter-spacing: -2px`
- **Cards (6):**
  - Style: `rgba(255,255,255,0.75)` bg, `border: 1px solid rgba(255,255,255,0.6)`, `border-radius: 18px`, `backdrop-filter: blur(20px)`, `box-shadow: 0 4px 20px rgba(0,0,0,0.04)`
  - Hover: `translateY(-4px)`, stronger shadow
  - Icon: 40x40px, 11px radius, colored gradient background
  - Content: 14px bold title, 12px muted description
  - Cards:
    1. Flock Management — track batches, health, mortality, growth
    2. Inventory Control — feed tracking, supply alerts
    3. Sales & Orders — PO to delivery, invoicing
    4. Processing — slaughter schedules, yield, traceability
    5. Analytics — dashboards, benchmarks, trends
    6. Finance — cost tracking, margins, reporting
- **Animation:** Cards stagger-in on scroll (Framer Motion `whileInView` + `staggerChildren`)
- **Responsive:** 3 cols → 2 cols (tablet) → 1 col (mobile)

### 4. AI Section (Dark Glassmorphic)
- **Background:** `linear-gradient(180deg, #151520, #111828, #151520)` with subtle radial green/purple glows
- **Layout:** 2-column grid — text + features on left, chat mockup on right
- **Heading:** `clamp(32px, 4vw, 52px)`, `font-weight: 300`, white text, accent in `#5cb87a`
- **AI Feature list (4 items):**
  - Glass cards: `rgba(255,255,255,0.04)` bg, `border: 1px solid rgba(255,255,255,0.06)`, 14px radius, backdrop blur
  - Icon: 32x32px, `rgba(92,184,122,0.1)` bg
  - Features: Natural language queries, predictive alerts, automated reports, cost optimization
- **Chat mockup:**
  - Container: Dark glass — `rgba(255,255,255,0.04)`, 20px radius, backdrop blur, deep shadow
  - Header: AI avatar (green gradient) + name + green online status dot
  - Messages: User bubbles (green-tinted), AI bubbles (white-tinted), realistic FCR conversation
- **Animation:** Left side slides in from left, chat mockup slides in from right (Framer Motion)
- **Responsive:** 2 cols → stacked (mobile), chat mockup below text

### 5. Stats (Glass Cards, Count-Up)
- **Background:** `linear-gradient(180deg, #f0eeeb, #e8e6f0, #f0eeeb)`
- **Layout:** Centered — section label + heading + 4-column card grid
- **Heading:** "Numbers that speak" with "speak" in `#3d8c5c`
- **Stat cards (4):**
  - Same glass style as feature cards (18px radius, backdrop blur)
  - Number: 42px, `font-weight: 300`, `letter-spacing: -2px`, accent symbols in green
  - Label: 10px uppercase, 1.5px tracking, muted
  - Stats: 500+ Active Breeders, 2K+ Batches Managed, $1M+ Transactions, 99.9% Uptime
- **Animation:** `useCountUp` hook — numbers animate from 0 to target on scroll into view (IntersectionObserver)
- **Responsive:** 4 cols → 2 cols (tablet) → 2 cols (mobile)

### 6. CTA (Photo Background)
- **Background:** Full-bleed farm photo (free-range hens), `filter: brightness(0.4)` overlay
- **Layout:** Centered text over photo
- **Heading:** `clamp(36px, 6vw, 68px)`, white, "Ready to run your farm smarter?" with accent
- **Subheadline:** 16px, `rgba(255,255,255,0.6)`
- **CTA button:** Green gradient, 12px radius, larger padding (16px 40px), shadow
- **Responsive:** Font scales via clamp, button full-width on mobile

### 7. Footer
- **Background:** `#111`
- **Layout:** Logo/tagline on left, 3 link groups on right (Product, Company, Legal)
- **Typography:** 10px uppercase group headers, 12px muted links
- **Copyright:** Bottom with top border divider

## Parallax Photo Dividers

Between Features → AI section, use a **parallax photo divider** (50vh height):
- Poultry house photo (user-provided) with Framer Motion `useScroll` + `useTransform` for parallax (`background-attachment: fixed` is broken on iOS Safari — use transform-based parallax instead)
- Gradient fade overlay on top and bottom edges to blend into adjacent sections
- Creates a visual break and photographic depth between the light and dark sections

## Design System (Dashboard Alignment)

### Colors
| Token | Value | Usage |
|-------|-------|-------|
| `--background-gradient` | `linear-gradient(135deg, #f0eeeb 0%, #e8e6f0 40%, #f2eff5 70%, #eef0eb 100%)` | Page background |
| `--accent-green` | `#3d8c5c` | Primary accent, buttons, highlights |
| `--accent-green-light` | `#5cb87a` / `#7ddfaa` | Dark mode accent, hero accent |
| `--glass-bg` | `rgba(255,255,255,0.75)` | Card backgrounds |
| `--glass-border` | `rgba(255,255,255,0.6)` | Card borders |
| `--glass-shadow` | `0 4px 20px rgba(0,0,0,0.04)` | Card shadows |
| `--glass-blur` | `20px` | Backdrop blur |
| `--text-primary` | `#1a1a1a` | Main text |
| `--text-muted` | `#aaa` | Secondary text |
| `--dark-bg` | `#151520` to `#111828` | AI section background |

### Typography
| Element | Size | Weight | Tracking |
|---------|------|--------|----------|
| Hero headline | `clamp(48px, 7vw, 88px)` | 300 | -3px |
| Section heading | `clamp(32px, 5vw, 52px)` | 300 | -2px |
| Section label | 10px uppercase | 500 | 3px |
| Card title | 14px | 600 | -0.2px |
| Card body | 12px | 400 | 0 |
| Stat number | 42px | 300 | -2px |
| Stat label | 10px uppercase | 500 | 1.5px |

### Radius
| Element | Radius |
|---------|--------|
| Cards | 18px |
| Buttons | 10-12px |
| Navbar | 16px |
| Icon containers | 9-11px |
| Chat mockup | 20px |
| Badges | 10px |

### Font
- **Primary:** Geist Sans (via `next/font/google` or local)
- **Mono:** Geist Mono (if needed for any accent elements)

## Animation System (Framer Motion)

### Global
- Custom easing: `[0.22, 1, 0.36, 1]`
- `whileInView` with `viewport: { once: true, margin: "-100px" }` for scroll-triggered animations

### Per-section
| Section | Effect |
|---------|--------|
| Hero bg | `translateY` at 0.3x scroll speed via `useScroll` + `useTransform` |
| Hero bg | Scale from 1.0 → 1.1 over first viewport scroll |
| Hero text | Fade + rise on load, spring easing |
| Navbar | Background opacity increases on scroll |
| Feature cards | Stagger in from bottom (`staggerChildren: 0.1`) |
| AI section | Left content slides from left, chat from right |
| Stats numbers | `useCountUp` with IntersectionObserver trigger |
| CTA | Text fades in on scroll |

### Existing hooks to preserve
- `useCountUp` — animated number counters
- `useMouseParallax` — remove (was for dashboard mockup, no longer needed)
- `useScrollProgress` — keep for navbar opacity

## i18n

Preserve bilingual support (Indonesian/English) using the existing `LanguageProvider` context pattern. All visible text must have entries in both languages. The translation structure in `src/lib/i18n.ts` will be simplified to match the reduced section count.

## Photos (User-Provided Assets)

| Photo | Usage | Notes |
|-------|-------|-------|
| Chick against blue sky | Hero background | High-res, full-bleed, parallax scroll |
| Poultry house interior | Parallax divider (Features → AI) | Fixed background attachment |
| Free-range hens in grass | CTA section background | Darkened overlay for text readability |

Photos should be placed in `breeding-landing/public/images/` and optimized:
- WebP format with JPEG fallback
- Hero: max 1920px wide, quality 80
- Dividers: max 1920px wide, quality 75

## Tech Stack (Preserved)

- Next.js 16 (React 19), static export
- Tailwind CSS 4
- Framer Motion 12
- Lucide React for icons
- TypeScript strict mode
- Base path: `/breeding-landing`

## What's Being Removed

- Fraunces and Outfit font families
- Organic blob shapes and floating elements
- Noise texture overlays
- Leaf pattern decorative elements
- Topographic contour line backgrounds
- Terracotta / earth brown color palette
- Problems section (3 pain point cards)
- Workflow section (5-step pipeline)
- Testimonials section
- Hero stat counters
- Interactive dashboard mockup with mouse parallax
- `useMouseParallax` hook

## What's Being Kept

- Framer Motion animation library and scroll-driven approach
- `useCountUp` hook for stats
- `useScrollProgress` for navbar
- Bilingual i18n system (LanguageProvider + translations)
- Static export configuration
- Component-per-section file structure
- Mobile responsive design (Tailwind breakpoints)

## Responsive Breakpoints

| Breakpoint | Changes |
|------------|---------|
| `lg` (1024px+) | Full layout — 3-col features, 4-col stats, 2-col AI |
| `md` (768px) | 2-col features, 2-col stats, stacked AI |
| `sm` (640px) | 1-col features, 2-col stats, stacked AI, centered hero text |
| Mobile (<640px) | Full-width everything, stacked CTAs, reduced font sizes via clamp |
