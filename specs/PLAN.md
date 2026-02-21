# Plan: Astro Migration of Ser Livre Psicologia

This document defines the full implementation plan derived from the study in `STUDY.md`. It covers every group of work, describes the rationale for each decision, and details exactly how each piece is to be implemented.

---

## Implementation Groups

1. Project scaffolding
2. Build tooling: Tailwind v4 and React integration
3. Design system foundation: tokens, fonts, utilities, shadcn
4. Shared layout component
5. Primitive components
6. Page section components
7. Asset handling
8. Analytics integration
9. Build verification

---

## 1. Project Scaffolding

### What

Initialise a blank Astro project (no starter template, blank canvas) inside `e:\projects\danilo\ser-livre-psicologia` at the repository root level, alongside the existing `_old` folder and `specs` folder.

### Why

The repository already has a `main` branch and a git history. The Astro project should live at the root, not inside a subdirectory, to allow a clean deployment without path configuration. The `_old` folder is a reference archive and should not be deleted until the migration is fully validated.

### How

Run `pnpm create astro@latest . --no-install --template minimal --typescript strict --git false` to scaffold a minimal Astro project in the current directory without installing dependencies or reinitialising git. Then run `pnpm install` to install the generated `package.json` (created by the Astro CLI, not written manually).

The resulting minimal project will contain:
- `astro.config.mjs`
- `tsconfig.json` (Astro's recommended strict preset)
- `src/pages/index.astro` (blank page)
- `src/env.d.ts` (Astro types reference)
- `package.json` with `astro` as the only dependency

Node version: the project requires Node 20 LTS or later (Astro 5.x minimum support). This should be documented via a `.nvmrc` or `package.json` `engines` field.

---

## 2. Build Tooling: Tailwind v4 and React Integration

### What

Add Tailwind CSS v4 (via its Vite plugin) and the `@astrojs/react` integration to enable the full target stack.

### Why

Tailwind v4 does not use a `tailwind.config.js` file. Its official integration with Vite-based frameworks is the `@tailwindcss/vite` plugin, added directly to `vite.plugins` in `astro.config.mjs`. The Astro team's `@astrojs/tailwind` wrapper targets Tailwind v3 and should not be used here.

`@astrojs/react` is necessary because shadcn components are React-based (Radix UI) and will be added for any interactive UI work. Without it, React components cannot be rendered from `.astro` files at all.

### How

Run in sequence:
```
pnpm add @tailwindcss/vite tailwindcss
pnpm astro add react
pnpm add -D @astrojs/check typescript
```

`pnpm astro add react` handles installing `@astrojs/react`, `react`, and `react-dom`, and automatically appends the integration to `astro.config.mjs`.

In `astro.config.mjs`, add the vite plugin manually (since the Astro CLI will not add it automatically):

```js
import tailwindcss from '@tailwindcss/vite'
export default defineConfig({
  vite: { plugins: [tailwindcss()] },
  integrations: [react()],
  output: 'static',
})
```

The `output: 'static'` directive ensures the build produces a portable `dist/` directory with no server runtime.

---

## 3. Design System Foundation

### What

This group establishes the design system layer that all components depend on:
- Global CSS file with design tokens and Tailwind directives
- Font loading via `@fontsource-variable` packages
- `cn()` utility (`src/lib/utils.ts`)
- shadcn initialisation

These four concerns are grouped together because they share a common dependency direction: every component imports from or is styled by these foundations. Changes in this group affect all components.

### Why

#### CSS tokens

The `globals.css` from `_old` is fully portable. It uses standard CSS custom properties and Tailwind v4 directives (`@import 'tailwindcss'`, `@theme inline`) that are framework-agnostic. The only required change is removing the `@custom-variant dark (&:is(.dark *))` class-based variant and replacing it with `@custom-variant dark (&:is([data-theme='dark'] *))` or leaving it as-is for future use. The hex color values, radius, font aliases, and all design tokens are kept exactly as-is.

The file is placed at `src/styles/globals.css` and imported inside the layout component's `<style>` tag using `@import '../styles/globals.css'` or in the `<head>` as a stylesheet import.

#### Fonts

`next/font/google` is replaced with the variable font packages from `@fontsource-variable`. Variable fonts reduce HTTP requests (single file for all weights) and the packages self-host the woff2 files inside `node_modules`, which Vite serves from the asset pipeline.

```
pnpm add @fontsource-variable/cormorant-garamond @fontsource-variable/dm-sans
```

Each package is imported once in the layout and provides a CSS file that defines the `@font-face` declarations. The CSS variable names (`--font-cormorant`, `--font-dm-sans`) in `@theme inline` remain unchanged; only the mechanism providing the font files is different.

Importing in the layout:
```ts
import '@fontsource-variable/cormorant-garamond'
import '@fontsource-variable/dm-sans'
```

Because Cormorant Garamond's variable font uses the `wght` axis (not `ital`), a separate import for italic may be needed: `@fontsource-variable/cormorant-garamond/index-italic.css`.

#### cn() utility

`src/lib/utils.ts` is identical to the old `lib/utils.ts`:

```ts
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'
export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

Dependencies: `pnpm add clsx tailwind-merge`

#### shadcn initialisation

Run `pnpm dlx shadcn@latest init` from the project root. The CLI will ask:
- Style: default
- Base color: neutral (the project uses fully custom colors via CSS vars so the base color is irrelevant)
- CSS variables: yes

The resulting `components.json` paths should be:
- `ui`: `src/components/ui`
- `utils`: `src/lib/utils`
- `components`: `src/components`
- `hooks`: `src/hooks`
- `lib`: `src/lib`

No shadcn components need to be added at this stage. The CLI only writes `components.json` and confirms the setup.

---

## 4. Shared Layout Component

### What

A single `src/layouts/Layout.astro` component that renders the full HTML shell: `<html>`, `<head>` (metadata, favicons, fonts, styles, analytics), and `<body>` with a `<slot />` for page content.

### Why

Astro layout components are the idiomatic equivalent of Next.js's `RootLayout`. Having one layout for a single-page site is appropriate. All metadata, external resources, and document-level concerns belong here, keeping page components focused on content.

Centralising the layout also means SEO additions (Open Graph, JSON-LD) have a single entry point. The layout accepts props for `title` and `description` to allow future pages to override them.

### How

```astro
---
interface Props {
  title?: string
  description?: string
}
const {
  title = 'Ser Livre Psicologia | Rhayane Miranda - Psicoterapia Online',
  description = 'Psicoterapia online com escuta sensível...',
} = Astro.props
---
<html lang="pt-BR">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta name="theme-color" content="#F5F0EB" />
    <title>{title}</title>
    <meta name="description" content={description} />
    <!-- favicons -->
    <!-- analytics script -->
  </head>
  <body>
    <a href="#conteudo-principal" class="sr-only focus:not-sr-only ...">
      Ir para o conteúdo
    </a>
    <slot />
  </body>
</html>
```

The skip-to-content link addresses the accessibility gap identified in the study. It is styled with Tailwind's `sr-only` class (visually hidden) and `focus:not-sr-only` (visible on keyboard focus).

Favicon links are written as standard HTML using the `<link rel="icon" media="(prefers-color-scheme: ...)" href="...">` pattern. The icon files (`icon-light-32x32.png`, `icon-dark-32x32.png`, `icon.svg`, `apple-icon.png`) are moved to `public/` to be served at the root path.

Font imports are placed at the top of the component script block (the `---` frontmatter) so Vite picks them up and adds the `@font-face` declarations to the output CSS.

---

## 5. Primitive Components

### What

Um componente SVG decorativo compartilhado é usado em múltiplas seções (HeroSection e CtaSection).

### How

O componente SVG decorativo deve ser implementado em `src/components/` com props para `class` (posicionamento/tamanho em Tailwind) e opção booleana para variação visual quando necessário.

In Astro, the `class` prop on a component is passed through via `Astro.props` and spread onto the SVG element. Astro also supports scoped styles, but since this component uses Tailwind utility classes passed from the outside, it needs no scoped CSS.

```astro
---
interface Props {
  class?: string
  showDiamond?: boolean
}
const { class: className = '', showDiamond = false } = Astro.props
---
<svg class={className} ...>
  ...
</svg>
```

Note: in Astro, `class` is a reserved HTML attribute and is handled correctly without renaming to `className`. The prop destructuring renames it locally to avoid conflict with the JSX convention.

---

## 6. Page Section Components

### What

All eight section components (Header, HeroSection, ForWhomSection, AboutSection, ApproachSection, HowItWorksSection, CtaSection, Footer) are converted from React `.tsx` to Astro `.astro` files.

### Why

Converting to `.astro` eliminates the React runtime from the output bundle entirely. Each section is purely static HTML with Tailwind classes — there are no hooks, state, or event handlers. The conversion is mechanical: array data stays as TypeScript in the frontmatter, `.map()` becomes `{items.map(...)}` inside expressions, `className` becomes `class`, and image tags use Astro's `<Image>` component.

### How

General rules for all component conversions:
- Move component logic (data arrays, constant definitions) into the `---` frontmatter block
- Replace JSX `className` with HTML `class`
- Replace `next/image` `<Image>` with `astro:assets` `<Image>` for images in `src/assets/`
- Replace `import { IconName } from 'lucide-react'` with `import { IconName } from 'lucide-react'` — lucide-react works in `.astro` files because Astro renders it server-side (the icons are just SVG strings)
- Replace `import Image from 'next/image'` with `import { Image } from 'astro:assets'` and update the `src` to use imported assets
- Condicionais visuais em SVGs decorativos seguem a mesma sintaxe de expressão (`{condicao && (...)}`) no Astro

Section-by-section notes:

**Header**: Fixed positioning, logo image, single anchor link. Replace `next/image` `<Image>` with native `<img>` pointing to the logo asset (the Header renders the logo at 36×36 — small enough that optimization is marginal; using `<img>` keeps it simple) or use Astro's `<Image>`. The WhatsApp anchor becomes a plain `<a>`.

**HeroSection**: Usa SVG decorativo, imagem de retrato (converter para Astro `<Image>` com `loading="eager"` por ser LCP) e link para WhatsApp. O path SVG inline do ícone de WhatsApp é copiado sem alteração.

**ForWhomSection**: Data array of strings rendered via `.map()`. The `<Check>` icon from lucide-react renders without change in Astro.

**AboutSection**: Data array of credential objects with lucide icon references. In Astro frontmatter, icon components can be stored in arrays and rendered using dynamic components: `<cred.icon class="..." />`. This works in `.astro` files because Astro resolves React components server-side. Alternatively, the icon can be stored as a string name and looked up — but the direct approach is simpler here.

**ApproachSection**: Data array of pillar objects with lucide icons, rendered as cards. Same pattern as AboutSection.

**HowItWorksSection**: Data array of step objects with lucide icons. The connector line conditional logic (`i < steps.length - 1`) works identically in Astro's expression syntax.

**CtaSection**: Usa SVG decorativo. O path SVG inline do ícone de WhatsApp é copiado sem alteração. Sem estado.

**Footer**: Three contact anchor links. The `lucide-react` icons (Mail, Phone, Instagram) are used inline. The logo image uses Astro's `<Image>`.

Files are placed at `src/components/sections/` to separate them from potential future UI components:
- `src/components/sections/Header.astro`
- `src/components/sections/HeroSection.astro`
- `src/components/sections/ForWhomSection.astro`
- `src/components/sections/AboutSection.astro`
- `src/components/sections/ApproachSection.astro`
- `src/components/sections/HowItWorksSection.astro`
- `src/components/sections/CtaSection.astro`
- `src/components/sections/Footer.astro`

The main page (`src/pages/index.astro`) imports the layout and all sections:

```astro
---
import Layout from '../layouts/Layout.astro'
import Header from '../components/sections/Header.astro'
// ... all sections
---
<Layout>
  <Header />
  <main id="conteudo-principal">
    <HeroSection />
    ...
  </main>
  <Footer />
</Layout>
```

The `id="conteudo-principal"` on `<main>` pairs with the skip-to-content link in the layout.

---

## 7. Asset Handling

### What

Move `brand-logo.png` and `rhayane-portrait.jpg` from `_old/public/images/` (or their equivalent) into `src/assets/images/` in the new Astro project.

### Why

Images in `src/assets/` are processed by Astro's image pipeline, which produces WebP/AVIF output, generates `srcset` for responsive loading, and infers `width`/`height` from the source file (eliminating layout shift). Images in `public/` bypass this pipeline entirely and are served as-is, replicating the old `unoptimized: true` behaviour without adding any value.

### How

Import each image as an asset module at the top of the component that uses it:

```ts
import brandLogo from '../../assets/images/brand-logo.png'
import portrait from '../../assets/images/rhayane-portrait.jpg'
```

Use Astro's `<Image>` component:

```astro
<Image src={portrait} alt="Rhayane Miranda, Psicóloga Clínica" loading="eager" class="..." />
```

For the logo (used at multiple sizes in three different components), import the same asset file in each component. Astro caches and deduplicates the source; the output will include one optimised variant per unique `width` used.

Favicon files (`icon-light-32x32.png`, `icon-dark-32x32.png`, `icon.svg`, `apple-icon.png`) are placed in `public/` since they are referenced in `<link>` tags with absolute paths (`/icon.svg`) and should not be renamed or hashed.

---

## 8. Analytics Integration

### What

Connect the `@vercel/analytics` script in the Astro layout without using a framework-specific wrapper component.

### Why

The `@vercel/analytics/next` module is a React component that self-injects a `<script>` tag and reports page views. In Astro, we do not use React for layout-level concerns. The Vercel Analytics script can be loaded as a plain `<script type="module">` in the layout `<head>`.

### How

Install: `pnpm add @vercel/analytics`

In `Layout.astro`, inject the script in the `<head>`:

```astro
<script type="module">
  import { inject } from '@vercel/analytics'
  inject()
</script>
```

Alternatively, if `@astrojs/vercel` adapter is added, set `webAnalytics: { enabled: true }` in the adapter options and remove the manual script. For the initial migration, the manual approach is preferred because it decouples analytics from the deployment adapter choice.

---

## 9. Build Verification

### What

After all components are written and the project is assembled, verify the static build produces correct output.

### How

Run `pnpm build` (which executes `astro build`) and inspect the `dist/` directory to confirm:
- `dist/index.html` is generated
- No JavaScript files in `dist/_astro/` reference React runtime (confirming no unintended client hydration)
- Images in `dist/_astro/` are WebP/AVIF (confirming the asset pipeline ran)
- The HTML contains valid `<title>`, `<meta description>`, and favicon `<link>` tags
- No TypeScript errors reported by `astro check`

Run `pnpm preview` (which executes `astro preview`) to serve the `dist/` folder locally and manually verify the rendered page matches the original visually.

The build should produce zero JavaScript shipped to the browser beyond the Vercel Analytics module (which is small and deferred).

---

## Directory Structure

The final project structure after migration:

```
/
  astro.config.mjs
  tsconfig.json
  package.json
  components.json          (shadcn configuration)
  .nvmrc                   (Node version pin)
  public/
    icon.svg
    icon-light-32x32.png
    icon-dark-32x32.png
    apple-icon.png
  src/
    assets/
      images/
        brand-logo.png
        rhayane-portrait.jpg
    components/
      sections/
        Header.astro
        HeroSection.astro
        ForWhomSection.astro
        AboutSection.astro
        ApproachSection.astro
        HowItWorksSection.astro
        CtaSection.astro
        Footer.astro
      ui/                  (populated by shadcn CLI on demand)
    layouts/
      Layout.astro
    lib/
      utils.ts
    pages/
      index.astro
    styles/
      globals.css
    env.d.ts
  specs/
    STUDY.md
    PLAN.md
  _old/                    (archived reference, not included in build)
```

---

## Decisions Summary

| Decision | Choice | Rationale |
|---|---|---|
| Astro output mode | `static` | 100% static, deployable anywhere |
| Tailwind integration | `@tailwindcss/vite` plugin | Native v4 integration, no config file |
| Font loading | `@fontsource-variable/*` | Self-hosted, zero layout shift, portable |
| Dark mode | CSS vars retained, no toggle | Not implemented in original; future opt-in |
| React integration | `@astrojs/react`, zero `client:*` | shadcn compatibility, no JS shipped |
| Components format | `.astro` for all sections | Zero JS output, simpler, idiomatic |
| Images location | `src/assets/images/` | Enables Astro image optimisation pipeline |
| Analytics | Manual `@vercel/analytics` script | Decoupled from adapter, works anywhere |
| shadcn | Initialised, no components added yet | Foundation for future UI additions |
| Skip link | Added in layout | Fills accessibility gap from original |
