---
name: subpage-redesign-protocol
description: A protocol for LLMs to safely redesign subpages using the Vecten Web Design Language without altering text.
---

# Subpage Redesign Protocol

This skill dictates how to redesign existing subpages using the Vecten Web Design Language. You are to act as an expert frontend engineer focused on pixel-perfect, premium UI execution.

## 🔴 CRITICAL RULE: READ-ONLY TEXT vs. STYLING/LAYOUT

**1. CONTENT (STRICTLY PRESERVED):**
**NEVER ALTER ANY TEXT CONTENT IN THE COMPONENTS.** All headings (H1, H2, H3), paragraphs, labels, data values, and structural logic of sections MUST remain absolutely identical to the original subpage content. Do not shorten, summarize, or invent text.

**2. STYLING & LAYOUT (MODIFIED):**
You are EXPECTED to change the UI, layout, styling, and animations. This includes:
- Adapting plain text blocks into structured visual components (e.g., transforming a text-heavy list into a `Timeline`, or sections into Bento-style cards).
- Changing typography (weights, sizes) as long as the exact text is preserved.
- Adding ambient backgrounds, decorations, or visuals (using standard meshes and glows from the Design Language).

**3. DESIGN LANGUAGE LOYALTY (ANTI-INVENTION):**
Do not invent styles outside of the Vecten Web Design Language. For instance, **NEVER introduce new font families** (like Inter or Roboto) if Geist is defined. If you feel a subpage requires a fundamentally new visual pattern or font that isn't captured in the Design Language, you MUST pause and use `notify_user` to ask for confirmation before proceeding.

**4. UNCERTAINTY & EFFICIENCY:**
If a transformation is proving exceedingly difficult, or if you are unsure how to map old content into the Design Language without breaking layout shifts (e.g., timeline getting offset), STOP. Do not waste time making multiple failing attempts. Use `notify_user` to explain the blocker concisely and ask for guidance. Prioritize quality and correctness over speed.

## Design Language Fundamentals

### Core Principles
- **Aesthetics First:** High-end, hyper-modern, tech-noir aesthetic.
- **Dark Mode Native:** The default and only theme is dark.
- **Micro-interactions:** Elements should respond gracefully to user interaction (hover states, subtle scaling, glowing effects).
- **Fluid Motion:** Implement smooth scrolling (Lenis) and entry animations (GSAP/Framer Motion).

### Typography
- **Primary Font:** Geist Sans (use `font-sans`).
- **Headings:**
  - Font weight: `font-medium` or `font-semibold`.
  - Leading: `leading-[1.1]` or `leading-tight`.
  - Tracking: `-0.03em` or `tracking-tight`.
- **Heading 4:** `text-lg font-medium` — used in clickable blocks and card subtitles.
- **Small / Label:** `text-sm text-gray-400 font-light` — supporting descriptions, block subtitles.
- **Paragraphs:**
  - Font weight: `font-light` or `font-normal`.
  - Color: `text-gray-400` or `text-zinc-400`.
  - Leading: `leading-relaxed`.

### Colors & Gradients
- **Backgrounds:** `bg-black`, `bg-[#0a0a0a]` (rich black). For alternating sections use `bg-[#050505]`.
- **Core Accent:** `#00E599` (Vibrant Neon Green). Used for accents, highlights, button hovers, and selections.
- **Secondary Accents:** `#009DFF` (Neon Blue), `cyan` hues.
- **Text:** `text-white` (headings), `text-gray-400` / `text-gray-500` (body text/labels).
- **Gradients (Text):** `<span className="text-transparent bg-clip-text bg-gradient-to-r from-[#B7F6D9] via-[#00E599] to-[#009DFF]">...</span>`
- **Surfaces (Cards):** Use glassmorphism or very subtle borders on pure black. e.g., `bg-[#0A0A0A] border border-white/10` or `bg-white/[0.02]`.

### Layout & Spacing
- Container max-width: commonly `max-w-[1400px]` or `max-w-7xl` with `mx-auto px-4 md:px-8`.
- Section padding: `py-24 md:py-32`.
- Gap structures: `gap-8`, `gap-16` for main layout elements; `gap-4` for internal component spacing.

### Animation & Motion (Framer Motion / GSAP)
- **Standard Easing:** `[0.16, 1, 0.3, 1]` (custom cubic-bezier).
- **Fade Up Entry:** `initial={{ opacity: 0, y: 20 }} animate={{ opacity: 1, y: 0 }} transition={{ duration: 0.8, ease: [0.16, 1, 0.3, 1] }}`.
- **Hover States:** Buttons should have subtle scale up, background color transition, and potentially an icon translation mechanism (e.g., arrow sliding right).

### Component Patterns
Refer to `src/app/pages/DesignLanguagePage.tsx` for live previews of each pattern.

- **Deliverable Card** — Feature/deliverable card with title + icon. Props: `title`, `description`, `icon`.
- **Timeline** — Scroll-linked vertical timeline with outcome cards. Each step has number, title, description, image, outcomes[], and optional `button`.
- **Problem Scan List** — Animated problem rows with closing solution row. Props: `problems[]`, `solution`.
- **Before & After** — Contrast comparison cards with red X / green check icons. Props: `beforeItems[]`, `afterItems[]`, `afterImage`.
- **Solution CTA (Fact Pills)** — Closing CTA with badge + H2 gradient + Fact Pills on the right. Content varies per solution page.
- **Solution CTA (Clickable Blocks)** — Same layout but right column has interactive `<a>` blocks with H4 title, small description, and arrow icon.
- **Stats Bar** — 3-column metrics row with gradient values, labels, and sublabels.
- **CEO Quote Section** — Full-width blockquote with CEO portrait, animated concentric rings, and hover blend effects.
- **Split Hero** — 2-column hero with badge + H1 gradient + CTA on left, PNG with drop-shadow on right.
- **Image + Copy** — Alternating 2-column layout with ambient glow image and inline gradient text highlights.
- **Buttons:** Pill-shaped (`rounded-full`), `h-[52px] px-8`. Default light inverted (`bg-white text-black`) or subtle dark (`border border-white/20 bg-transparent text-white hover:bg-white/10`).
- **Badges:** Use text badges: e.g., `text-[#00E599] font-mono text-xs tracking-widest uppercase bg-[#00E599]/10 px-3 py-1.5 rounded-full border border-[#00E599]/20`.
- **Background Ambiance:** Use large absolute divs with heavy blur and low opacity to create ambient lighting behind content (e.g., `bg-[#00E599]/10 mt-[20%] blur-[120px] rounded-full absolute pointer-events-none`).
- **Glass Cards with Hover Glow:** Use `relative overflow-hidden group border border-white/10 hover:border-[#00E599]/30 transition-all duration-500` wrapping a gradient glow `absolute top-0 right-0 w-64 h-64 bg-gradient-to-br from-[#00E599]/20 to-transparent opacity-40 rounded-full blur-3xl transform translate-x-1/3 -translate-y-1/3 group-hover:opacity-80 transition-opacity duration-700`.
- **Section Separators:** `<div className="absolute top-0 inset-x-0 h-px bg-gradient-to-r from-transparent via-white/10 to-transparent"></div>` instead of hard borders.

## Workflow Execution Steps
When triggered to redesign a subpage, follow these exact steps:

1. **Read Original File:** Use `view_file` to capture the EXACT content structure of the target page.
2. **Draft Component Structure:** Plan the new layout using the Design Language patterns defined above.
3. **Map Content (Strictly):** Map the original text into the new structure. Do not invent headers or shorten text.
4. **Implement UI:** Rewrite the React component file, heavily utilizing Tailwind CSS, Framer Motion, and the specific color hexes/patterns defined here.
5. **Add Motion:** Ensure smooth transitions on mount and interactable elements.
6. **Verify:** Check that no text was altered. Review the component for responsive behavior (mobile stack -> desktop flex/grid).
