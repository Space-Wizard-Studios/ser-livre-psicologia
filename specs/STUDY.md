# Study: Migration from Next.js to Astro

This document records the analysis of the existing `_old` Next.js project and every aspect relevant to rebuilding it as a fully static Astro site. Topics are filled in sequentially as the analysis progresses.

---

## 1. Project Purpose and Scope

The project is a single-page marketing and institutional landing site for "Ser Livre Psicologia", a psychology private practice run by Rhayane Miranda (CRP 01/28820). The site targets women seeking online individual psychotherapy.

The page is composed of eight discrete visual sections rendered in a fixed order:

- Header: fixed navigation bar with logo and a WhatsApp CTA button
- Hero: full-viewport section with portrait photo, tagline, subtitle, and a primary WhatsApp CTA
- For Whom: checklist of target profiles (five items)
- About: biography, credentials chips, and a brand logo image
- Approach: three thematic pillars rendered as cards (Gestalt-terapia framework)
- How It Works: three-step process with icons and connector line
- CTA: full-bleed call-to-action banner with WhatsApp link
- Footer: logo, contact links (email, WhatsApp, Instagram), and copyright

There are no forms, no user accounts, no authentication, no API routes, and no dynamic data. The WhatsApp link (`https://wa.me/5561999335952`) is the sole conversion gateway. Every pixel on the page is known at build time, making this an ideal candidate for a 100% static output.

---

## 2. Current Tech Stack

Runtime and framework:
- Next.js 16.1.6 with App Router
- React (bundled with Next.js)
- TypeScript (strict mode, `noEmit`, `moduleResolution: bundler`)

Styling:
- Tailwind CSS v4 (imported via `@import 'tailwindcss'`, no config file)
- `tw-animate-css` for utility animation classes
- CSS custom properties for the full design token palette (colors, radius, sidebar, charts)
- `@theme inline` block mapping tokens to Tailwind color/font/radius utilities
- Class-based dark mode via `@custom-variant dark (&:is(.dark *))`

Component layer:
- shadcn/ui configured via `components.json` (style: default, RSC on, Tailwind CSS vars, path aliases)
- `@radix-ui/*` primitives as peer dependencies for shadcn components
- `lucide-react` for all iconography
- `next-themes` for dark/light toggle (declared but not wired into the layout)

Typography:
- Cormorant Garamond (weights 300–700) loaded via `next/font/google` and exposed as `--font-cormorant` CSS variable
- DM Sans (weights 300–500–700) loaded via `next/font/google` and exposed as `--font-dm-sans` CSS variable
- Both variables mapped to Tailwind `--font-serif` and `--font-sans` inside `@theme inline`

Images:
- `next/image` with `unoptimized: true` in `next.config.mjs` — no actual transformation occurs at build time

Analytics:
- `@vercel/analytics/next` added as the `<Analytics />` component inside `RootLayout`

Utilities:
- `clsx` and `tailwind-merge` composed into a `cn()` helper (`lib/utils.ts`)
- `class-variance-authority` as a shadcn dependency
- `@hookform/resolvers`, `react-hook-form`, `zod`, `date-fns`, `embla-carousel-react`, `input-otp`, `cmdk` — present as dependencies but not referenced by any rendered component

---

## 3. Component Inventory

Components actually referenced by `page.tsx` and rendered in the browser:

| Component | File | Interactivity | External dependencies |
|---|---|---|---|
| Header | `components/header.tsx` | None (plain anchor links) | `next/image` |
| HeroSection | `components/hero-section.tsx` | None (plain anchor links) | `next/image`, SVG decorativo |
| ForWhomSection | `components/for-whom-section.tsx` | None | `lucide-react` (Check) |
| AboutSection | `components/about-section.tsx` | None | `next/image`, `lucide-react` (5 icons) |
| ApproachSection | `components/approach-section.tsx` | None | `lucide-react` (Ear, Eye, Sparkles) |
| HowItWorksSection | `components/how-it-works-section.tsx` | None | `lucide-react` (MessageCircle, CalendarCheck, ThumbsUp) |
| CtaSection | `components/cta-section.tsx` | None (plain anchor link) | SVG decorativo |
| Footer | `components/footer.tsx` | None | `next/image`, `lucide-react` (Mail, Phone, Instagram) |

Components present in `components/ui/` (entire shadcn output) plus `ThemeProvider`, `use-mobile.tsx`, `use-toast.ts` — none of these are imported by any of the above rendered components.

Summary: the actual rendered surface uses 9 components, none of which rely on React state, lifecycle hooks, context providers, or client-side event handlers beyond anchor tag navigation.

---

## 4. Static vs Dynamic Surfaces

Every data point visible on the page is hardcoded in source files — section copy, credentials list, step descriptions, pillar cards, contact details. There is no:

- Data fetching at runtime (no `getServerSideProps`, `fetch`, SWR, React Query)
- Database or CMS integration
- Authentication or session handling
- API routes
- User-generated content
- Form submission (the contact gateway is an external WhatsApp deep-link)

The single interactive surface is the WhatsApp URL (`https://wa.me/5561999335952`), which opens an external application. No JavaScript is required to render or operate any part of the visible page.

This confirms the site is 100% suitable for static output: `output: 'static'` in Astro will produce a single `index.html` plus associated assets with zero server-side runtime dependency.

---

## 5. Typography and Font Loading

Two typefaces in use:

- Cormorant Garamond — Google Fonts, subsets: latin, weights 300/400/500/600/700. Assigned to `--font-serif` / `font-serif` utility. Used in all headings.
- DM Sans — Google Fonts, subsets: latin, weights 300/400/500/700. Assigned to `--font-sans` / `font-sans` utility. Used in body copy, labels, and UI text.

In Next.js these are loaded via `next/font/google`, which self-hosts the font files at build time, injects a preload `<link>`, and returns a CSS variable string to attach to `<body>`. This eliminates layout shift and avoids a runtime Google Fonts request.

In Astro there is no equivalent of `next/font`. Options:

1. `@fontsource` packages: `@fontsource/cormorant-garamond` and `@fontsource/dm-sans` are available on npm. They bundle woff2 files locally and provide import-ready CSS. This mirrors the Next.js self-hosting behaviour and is the most portable option.
2. Astro Font API (`astro:assets` font optimization): introduced in Astro 5.7, available by adding `experimental.fonts` in `astro.config.mjs`. This downloads and self-hosts the fonts at build time with automatic preloading, mirroring `next/font` semantics exactly. It is the most ergonomic option if Astro 5.7+ is used.
3. Google Fonts stylesheet `<link>` in `<head>`: simplest but introduces an external request, possible CLS, and a privacy-relevant third-party connection.

The CSS variable names (`--font-cormorant`, `--font-dm-sans`) are assigned in the `@theme inline` block and can be retained unchanged. Only the mechanism that loads and exposes the variables changes.

---

## 6. Image Handling

Images referenced by the components:

| Path | Used in | Dimensions rendered | Notes |
|---|---|---|---|
| `/images/brand-logo.png` | Header, AboutSection, Footer | 36×36, 320×320, 32×32 | Logo mark, appears at three sizes |
| `/images/rhayane-portrait.jpg` | HeroSection | up to 384×384 | Portrait, circular crop via `rounded-full` |

In the current project, `next/image` is used with `unoptimized: true` in `next.config.mjs`. This means no resizing, no format conversion, and no automatic WebP/AVIF is produced — the original files are served as-is. The use of `next/image` here provides only lazy loading and the `width`/`height` dimensioning, which are achievable with a plain `<img>` tag.

In Astro, the built-in `<Image>` component from `astro:assets` performs actual build-time optimisation when the source image is located in `src/` (i.e., `src/assets/`). For images in `public/`, Astro serves them verbatim (same as the current behaviour).

Moving both images to `src/assets/images/` and importing them as modules enables:
- Automatic WebP/AVIF conversion
- Width/height inference (no need to hardcode dimensions)
- Content-hashed filenames for cache-busting
- Automatic `loading="lazy"` and `decoding="async"` attribution

The portrait image (`priority` in Next.js hero) should use Astro's `loading="eager"` to prevent LCP penalty.

---

## 7. CSS, Tailwind v4, and Theming

Tailwind v4 is configured via a single CSS entry file using the `@import 'tailwindcss'` directive rather than a `tailwind.config.js` file. All design tokens live inside a `@theme inline { ... }` block. This approach is entirely framework-agnostic — the CSS file itself needs no changes for Astro.

Tailwind v4 integrates with Vite (Astro's bundler) through the official `@tailwindcss/vite` plugin. The integration does not require an `@astrojs/tailwind` wrapper; the plugin is added directly to the `vite.plugins` array in `astro.config.mjs`.

Color palette (defined as hex values in `app/globals.css`, mapped via `@theme inline`):
- Background: `#F5F0EB` (warm off-white)
- Foreground: `#1A2744` (dark navy)
- Primary: `#C17A7A` (dusty rose)
- Secondary: `#E8D5C0` (warm beige)
- Accent: `#1A2744` (navy, same as foreground — used for dark panels)
- Border: `#D9CFC5`
- Muted foreground: `#5A6478`

Dark mode variables are defined but the dark mode toggle is not implemented in the layout (no `ThemeProvider` is applied). The `@custom-variant dark (&:is(.dark *))` variant requires the `dark` class to be present on a parent element. Since no mechanism adds this class, dark mode is effectively inactive. In the Astro version this can remain as-is (future dark mode opt-in) or be replaced with a `@media (prefers-color-scheme: dark)` variant for automatic OS-based theming.

The `tw-animate-css` package provides Tailwind utility classes for CSS keyframe animations (`animate-in`, `animate-out`, `fade-in`, etc.). It is listed as a dependency and imported in `globals.css` but no component currently uses any animation utilities from it. It can be retained as a zero-cost import or removed.

---

## 8. shadcn Integration Strategy

shadcn is a code-generation tool, not an installable library. It reads a `components.json` configuration and writes component source files into the project. The generated components are Radix UI primitives wrapped in Tailwind-styled code using the `cn()` utility.

The `_old/components.json` is configured for Next.js (with RSC support enabled). For Astro, a new `components.json` must be initialised via `pnpm dlx shadcn@latest init`, which detects the Astro project structure and writes an Astro-compatible configuration.

Key differences in the Astro-flavour of shadcn:
- Components are still React (Radix UI requires a React tree)
- The `"use client"` directive is not relevant in Astro; React components rendered in `.astro` files are server-rendered by default unless a `client:*` directive is passed
- The `components.json` will point `aliases.utils` to `src/lib/utils`, `aliases.components` to `src/components/ui`, etc.

Actual usage audit: none of the 9 rendered page components import anything from `components/ui/`. The `cn()` helper from `lib/utils.ts` is used by the shadcn-generated UI files but is not used by the page section components themselves — they use raw Tailwind class strings.

Conclusion: shadcn should be initialised and configured so the project is ready for future component additions, but no shadcn components need to be added for the initial migration. The `cn()` utility is still worth keeping for consistency.

---

## 9. React Usage Scope in Astro

In Astro, components default to zero JavaScript output. A `.astro` component is compiled to static HTML at build time. A React `.tsx` component used inside an `.astro` file is also server-rendered by default (no hydration, no React runtime shipped to the browser) unless a `client:load`, `client:idle`, `client:visible`, or `client:only` directive is applied.

Review of the nine rendered components against this model:

- None use `useState`, `useEffect`, `useRef`, `useContext`, or any React hook.
- None use event handlers (no `onClick`, `onChange`, etc.). The WhatsApp links are standard `<a href>` elements requiring no JS.
- None depend on browser APIs.
- Os SVGs decorativos são markup estático sem lógica de componente reativa.

This means every component can be fully converted to `.astro` format, producing static HTML with zero JavaScript hydration. The React runtime (approximately 45 kB gzipped) would then be absent from the production bundle entirely — unless a shadcn interactive component (accordion, dialog, etc.) is added and hydrated.

The `@astrojs/react` integration is still worth including because:
- shadcn components are React-based and will need it when added
- It makes converting specific sections to interactive islands straightforward when needed
- The cost is near zero when no `client:*` directive is used

All page section components should be written as `.astro` files. SVGs decorativos podem ser componentes `.astro` ou assets `.svg` importados diretamente.

---

## 10. SEO and Metadata

Existing metadata from `layout.tsx`:

- `<title>`: "Ser Livre Psicologia | Rhayane Miranda - Psicoterapia Online"
- `<meta name="description">`: "Psicoterapia online com escuta sensível, acolhedora e atenta às vivências das mulheres. Rhayane Miranda, Psicóloga Clínica CRP 01/28820."
- `<meta name="theme-color">`: `#F5F0EB`
- `<link rel="icon">` variants: `icon-light-32x32.png` (light scheme), `icon-dark-32x32.png` (dark scheme), `icon.svg` (SVG fallback)
- `<link rel="apple-touch-icon">`: `apple-icon.png`
- `lang="pt-BR"` on `<html>`

Missing in the current project (opportunities to add during migration):
- Open Graph tags (`og:title`, `og:description`, `og:image`, `og:url`, `og:type`)
- Twitter Card tags
- Canonical URL
- Structured data (LocalBusiness or Person schema in JSON-LD)

In Astro, metadata lives in a `<head>` slot within the root layout (a `.astro` file). There is no special API — it is plain HTML `<meta>` and `<link>` tags rendered conditionally from component props. This is more explicit than Next.js's `metadata` export and equally powerful.

A base layout `Layout.astro` should accept props for `title`, `description`, and `ogImage` to allow page-level overrides, even though the project currently has only one page.

---

## 11. Analytics and Third-Party Scripts

The current project uses `@vercel/analytics/next`, which was designed specifically for Next.js and injects a tracking script tied to Vercel's infrastructure.

For a 100% static Astro site, there are two relevant paths:

1. Vercel deployment with `@astrojs/vercel` adapter (static mode): the adapter can inject the `@vercel/analytics` script automatically via its `webAnalytics` option. No manual script tag is needed.
2. Any host without the Vercel adapter: `@vercel/analytics` provides a plain `<script>` tag that can be injected in the layout `<head>` (or via Astro's `<script>` tag within the layout component). This works independently of the deployment platform as long as the Vercel Analytics dashboard is configured.

Alternatively, if the deployment moves away from Vercel entirely, the analytics dependency can be dropped or replaced with a privacy-friendly self-hosted alternative (e.g., Plausible, Umami).

For now, the correct approach is to keep `@vercel/analytics` and inject the script in the layout `<head>` using Astro's standard `<script>` mechanism, decoupled from any framework adapter.

---

## 12. Dependencies Audit

Dependencies in the old project categorised by migration status:

Retain (used by rendering components):
- `lucide-react` — icons used in five sections
- `clsx` — used by `cn()` helper
- `tailwind-merge` — used by `cn()` helper
- `class-variance-authority` — shadcn peer dependency
- `@radix-ui/react-slot` — shadcn Button's `asChild` prop

Retain but migrate (Next.js-specific implementation):
- `@vercel/analytics` — switch to the adapter-agnostic script injection
- Font packages — replace `next/font/google` with `@fontsource/*` or Astro Font API

Remove (unused on the rendered page):
- `@hookform/resolvers`, `react-hook-form`, `zod` — no forms exist
- `date-fns` — no date formatting
- `embla-carousel-react` — no carousel
- `input-otp` — no OTP input
- `cmdk` — no command palette
- `next-themes` — no toggle; theming is CSS-only
- `next` — replaced by `astro`

Remove (unused shadcn Radix primitives — can be added back on demand):
- All `@radix-ui/*` packages except `@radix-ui/react-slot`

Add (Astro stack):
- `astro`
- `@astrojs/react`
- `@astrojs/check` (TypeScript checking for `.astro` files)
- `@tailwindcss/vite` (Tailwind v4 Vite plugin)
- `tw-animate-css` (already present, keep as-is)
- `typescript`
- Font provider: `@fontsource-variable/cormorant-garamond` and `@fontsource-variable/dm-sans` (variable font variants — single file, all weights)

---

## 13. Accessibility Considerations

What the current implementation already does correctly:
- `aria-hidden="true"` on all decorative SVGs (including WhatsApp icon)
- Descriptive `alt` text on both images
- Semantic sectioning (`<header>`, `<main>`, `<section>`, `<footer>`)
- `lang="pt-BR"` declared on `<html>`
- Understandable link labels ("Agendar sessão", "Falar com a Rhayane pelo WhatsApp", etc.)
- Tailwind's `outline-ring/50` base style provides a default focus ring

Gaps to address during migration:
- No skip-to-content link: the fixed header overlays the page, making keyboard navigation start above the main content. A visually-hidden "Ir para o conteúdo" link pointing to `#conteudo-principal` should be added.
- The fixed header `pt-20` offset on the hero is a layout workaround; the target anchor for the header logo link (`href="#"`) should scroll to the top intentionally.
- Credential chip `<div>` containers in the About section have no semantic meaning — they are presentational but do not require ARIA roles since they contain visible text.
- The `HowItWorksSection` connector line between steps is purely decorative and correctly implemented as a visual `<div>` without content semantics.

---

## 14. Deployment Target

The original project is deployed on Vercel (evidenced by `@vercel/analytics` and the default Next.js deployment workflow). Vercel supports Astro static sites natively without any adapter — the `dist/` output folder is served from Vercel's CDN edge nodes.

For a 100% static Astro build:
- `output: 'static'` in `astro.config.mjs` produces a `dist/` folder containing `index.html`, hashed JS/CSS bundles, and the `public/` assets
- No adapter is required for Vercel static hosting; Vercel auto-detects the framework and runs `astro build`
- If an adapter is desired for web analytics injection, `@astrojs/vercel` can be added and configured with `webAnalytics: { enabled: true }`, which eliminates the need for a manual script tag

The site should also be deployable without modification to Netlify, Cloudflare Pages, GitHub Pages, or any static file host by pointing the host at the `dist/` directory. No environment variables or server-side configuration are required.
