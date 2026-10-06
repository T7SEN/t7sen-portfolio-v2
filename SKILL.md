---
name: t7sen-v2
description: Complete working reference for the t7sen.com v2 portfolio repo, a Next.js 16 static-export content layer over a persistent three.js WebGPU particle simulation (TSL compute; WebGPU, WebGL 2 and static tiers), GSAP ScrollTrigger and Lenis on one clock, design tokens shared by CSS and shaders, and one Cloudflare Worker serving static assets and the contact form. Use this skill for ANY task in this repository, including engine or shader work, simulation kernels, scroll choreography, tiers and fallbacks, content pages and case studies, the command palette and easter eggs, tokens, tests (unit, e2e, visual regression, forced WebGL), the asset and capture pipelines, deployment, and docs, even when the task looks small or purely cosmetic.
---

# t7sen.com v2

Product spec: `docs/SPEC.md`. Lean rules: `AGENTS.md`. This file is the operational reference. Keep it accurate: a wrong path or URL here propagates into every future session.

## 1. Identity and priorities

- Owner: T7SEN. The site is faceless; the logo, the motion, the work, and the copy carry the personality.
- Goals: full-time developer roles and freelance client work. Two calls to action: "Start a project" and "Hire me full-time".
- Signature: one persistent GPU simulation, choreographed by scroll. A red field (Reversal, repulsion) and a blue field (Lapse, attraction) collide, fuse into purple (Hollow), and resolve into the T7 mark.
- Priority order: visual fidelity, then frame pacing, then everything else. Load time and Lighthouse scores are not targets.

## 2. Stack manifest

Exact versions are pinned in Phase 0 after checking official release notes, then written into the Pinned column. Runtime dependencies never use `latest` or caret ranges.

| Layer | Choice | Floor | Pinned |
|---|---|---|---|
| Runtime | Node LTS, pnpm workspaces | Node 24 | — |
| Content | Next.js App Router, `output: 'export'` | 16.x | — |
| UI | React | 19.x | — |
| Language | TypeScript, `strict: true` | current stable | — |
| Styling | Tailwind CSS, CSS-first `@theme` | 4.x | — |
| Engine | three.js `WebGPURenderer` and TSL | r182 | — |
| Motion | GSAP with ScrollTrigger | 3.13 | — |
| Smooth scroll | Lenis | current stable | — |
| Edge | Cloudflare Workers static assets, Hono, Wrangler | Wrangler 4.x | — |
| Forms | Zod, Cloudflare Turnstile, Discord webhook | Zod 4.x | — |
| Media | Cloudflare R2, public bucket on a custom domain | — | — |
| Tests | Vitest, Playwright | current stable | — |
| Assets | glTF-Transform with Meshopt, KTX2 textures | current stable | — |

## 3. Repository structure

```
/
├── AGENTS.md                 lean rules, read every session
├── CLAUDE.md                 imports AGENTS.md
├── SKILL.md                  this file
├── wrangler.jsonc            single Worker: static assets + /api/*
├── docs/
│   ├── SPEC.md               product spec and phase plan
│   ├── PROJECT_INSTRUCTIONS.md
│   ├── SESSION_PROMPTS.md
│   ├── adr/                  NNNN-kebab-title.md
│   └── walkthroughs/         annotated explanations of non-obvious code
├── apps/
│   ├── web/                  Next.js 16 static export → apps/web/out
│   │   ├── app/              routes; layout.tsx mounts EngineHost once
│   │   ├── src/engine-host/  React boundary around packages/engine
│   │   ├── src/motion/       transitions, reveals, cursor
│   │   ├── src/palette/      command palette
│   │   ├── src/eggs/         easter eggs
│   │   ├── src/contact/      contact form
│   │   └── public/brand/     T7SEN-Dark.svg, T7SEN-Light.svg, T7SEN-Purple.svg
│   └── edge/                 Hono Worker: POST /api/contact
├── packages/
│   ├── engine/src/
│   │   ├── core/             createEngine, renderer bootstrap, frame, dispose
│   │   ├── tiers/            probe, benchmark, dynamic resolution, tier kernels
│   │   ├── sim/              TSL kernels, buffers, forces, scene state
│   │   ├── mark/             T7 polygon and its signed distance function
│   │   ├── output/           P3, HDR, tone mapping, post-processing
│   │   ├── readout/          frame time, GPU time, particles, tier
│   │   └── deterministic/    seeded RNG, fixed timestep, keyframe scrub
│   ├── tokens/               OKLCH source → CSS variables + shader constants
│   ├── contracts/            shared Zod schemas
│   └── config/               tsconfig, ESLint
└── tools/
    ├── assets/               glTF and texture compression
    └── capture/              Playwright capture for the static tier
```

## 4. Architecture

- **Two layers.** The content layer is static HTML that works with no GPU at all. The engine is progressive enhancement on top.
- **Persistent canvas.** `EngineHost` mounts once in `apps/web/app/layout.tsx`, full-bleed and fixed behind the content. Route changes never remount it.
- **One clock.** `gsap.ticker` calls `lenis.raf(time * 1000)` and `engine.frame(time)`. `gsap.ticker.lagSmoothing(0)`. Lenis scroll events call `ScrollTrigger.update`.
- **Scroll bridge.** ScrollTrigger timelines tween a plain `SceneState` object. The engine reads `SceneState` each frame and maps it to uniforms. Components never tween three.js objects directly.
- **Scenes are states.** There is one simulation. Sections change its parameters; nothing is torn down between sections.
- **Tiers.** A: WebGPU full. B: WebGPU reduced. C: WebGL 2 with its own portable kernel set. D: static captures. Selection runs a capability probe (adapter, features, limits), then a short first-visit benchmark; the result is cached in `localStorage` under `t7:tier` and can be overridden from the palette.
- **Runtime scaling.** Dynamic resolution holds the frame-time budget. The engine drops a tier only after the resolution floor is reached. Resolution and particle count are sacrificed before smoothness.
- **HDR overlay.** Tiers A and B only, when the canvas accepts `toneMapping: { mode: 'extended' }`. Custom output path.
- **Failure handling.** On `device.lost`, re-initialize once; on a second loss, drop a tier. A black screen is never an acceptable failure mode.

## 5. Subsystems

### 5.1 Engine core — `packages/engine/src/core`
`createEngine(canvas, options)` returns `{ frame, setState, setTier, dispose, readout }`. Initialization is an idempotent singleton guarded by an `AbortController`, so a React development double-mount never creates two GPU devices.

### 5.2 Simulation — `packages/engine/src/sim`
TSL compute kernels over storage buffers: position, velocity, field identity (reversal, lapse, hollow), and target. Forces: curl noise, reversal repulsion, lapse attraction, fusion (blends field identity toward hollow), mark attraction via the SDF, model-surface targets, and the cursor. Pair-mode: drafts stop for T7SEN's sign-off.

### 5.3 Mark — `packages/engine/src/mark`
The T7 mark is one 14-vertex polygon (section 6.4). Its signed distance is computed analytically per particle: minimum distance to the edges, sign from the winding number. No baked texture. The same vertex constant renders the SVG mark in `apps/web`.

### 5.4 Tiers — `packages/engine/src/tiers`
Probe, benchmark, dynamic resolution, and one kernel set per tier. Tier C kernels use only features WebGL 2 supports: no atomics, no indirect draws.

### 5.5 Output — `packages/engine/src/output`
Canvas `colorSpace: 'display-p3'` where supported. HDR extended mode is capability-gated with custom tone mapping. Bloom and tone mapping run as TSL post-processing.

### 5.6 Readout — `packages/engine/src/readout`
Frame time, GPU time (via the `timestamp-query` feature when available), particle count, active tier, and device pixel ratio. Toggled from the palette and rendered by `apps/web` in the mono typeface.

### 5.7 Content layer — `apps/web`
Routes: `/`, `/work/coup-online`, `/work/t7sen-url`, `/work/v1`, plus a designed 404. Long text sits on Ink reading planes over the canvas. Static metadata and Open Graph images per route.

### 5.8 Motion system — `apps/web/src/motion`
Page transitions, reveal patterns, and the custom cursor, all driven by duration and ease tokens. The cursor is also a force in the simulation.

### 5.9 Tokens — `packages/tokens`
OKLCH source of truth. The build emits `tokens.css` (Tailwind v4 `@theme`) and `tokens.ts` (linear-sRGB constants for shaders).

### 5.10 Command palette — `apps/web/src/palette`
Built from scratch. ARIA combobox and listbox pattern, fuzzy search, and an action registry covering navigation, readout, reduced motion, tier override, copy email, and easter egg hints.

### 5.11 Easter eggs — `apps/web/src/eggs`
Each egg exercises the engine rather than CSS. Hints are discoverable through the palette.

### 5.12 Contact pipeline — `apps/web/src/contact` → `apps/edge`
Form fields: intent (client project or full-time role), name, email, message, plus optional budget and timeline for client projects. A shared Zod schema from `packages/contracts` validates on both sides. The Worker verifies Turnstile server-side, checks a honeypot, and posts an embed to the Discord webhook.

### 5.13 Asset pipeline — `tools/assets`
glTF compressed with Meshopt; textures to KTX2. Any file over the static-asset cap goes to R2.

### 5.14 Capture pipeline — `tools/capture`
Playwright runs deterministic mode, records keyframe stills and loop videos for Tier D, and uploads them to R2.

## 6. Design tokens

### 6.1 Palette

| Token | Hex | OKLCH | Role |
|---|---|---|---|
| `void` | `#000000` | `oklch(0% 0 0)` | Page background, canvas |
| `ink` | `#111114` | `oklch(17.9% 0.006 285.8)` | Reading planes |
| `graphite` | `#1d1d23` | `oklch(23.3% 0.011 285.5)` | Raised elements: palette, cards |
| `smoke` | `#313138` | `oklch(31.6% 0.012 285.7)` | Borders, dividers |
| `ash` | `#a2a2ac` | `oklch(71.5% 0.014 286.0)` | Secondary text |
| `bone` | `#f0f0f4` | `oklch(95.6% 0.005 286.3)` | Primary text |
| `hollow` | `#a052ff` | `oklch(62% 0.243 299.9)` | Main accent: fills, focus, key UI |
| `hollow-light` | `#b688ff` | `oklch(72% 0.172 299.9)` | Accent text and links |
| `reversal` | `#ef4444` | `oklch(63.7% 0.208 25.3)` | Simulation and logo gradient only |
| `lapse` | `#3b82f6` | `oklch(62.3% 0.188 259.8)` | Simulation and logo gradient only |

P3 enhancement: `hollow` becomes `oklch(62% 0.263 300)` and `hollow-light` becomes `oklch(72% 0.188 300)` under `@media (color-gamut: p3)`.

Shader constants (linear sRGB): hollow `0.3515, 0.0844, 1.0000`; reversal `0.8632, 0.0578, 0.0578`; lapse `0.0437, 0.2232, 0.9216`. HDR emissive values are multiples of these, applied only in the HDR overlay.

### 6.2 Background layering
- `void` is set on `<html>` inline in the document head, with `<meta name="theme-color" content="#000000">` and `color-scheme: dark`, so the page never flashes white.
- The canvas clears to `void`.
- Long text sits on `ink` planes at 92% opacity over the canvas. There is no backdrop blur over the live canvas.
- `graphite` is for raised elements; `smoke` is for hairlines.
- The site is dark-only. There is no light mode.

### 6.3 Contrast rules (WCAG 2.x, measured)
- `bone` on `ink` 16.6:1. `ash` on `ink` 7.4:1. `hollow-light` on `ink` 7.1:1. `hollow` on `ink` 4.6:1.
- Labels on `hollow` fills are `void` (5.2:1). White on `hollow` is 3.6:1 and is banned.
- `reversal` and `lapse` never appear in UI chrome.

### 6.4 The mark
viewBox `0 0 464.22 442.36`. Vertices, in order:
`(464.22,0) (464.22,7.43) (411.18,73.13) (186.22,351.77) (113.09,442.36) (113.09,325.94) (186.22,235.35) (317.19,73.13) (186.22,73.13) (186.22,164.38) (113.09,254.97) (113.09,73.13) (0,73.13) (0,0)`.
The purple logo's gradient runs `reversal` → `hollow` → `lapse`.

### 6.5 Motion tokens (initial values, tuned in Phase 5)
Durations: `fast` 160 ms, `base` 320 ms, `slow` 640 ms, `cinematic` 1200 ms. Eases: `reveal` expo.out, `transit` power3.inOut, `collide` a CustomEase defined in Phase 5.

### 6.6 Typography
Chosen in Phase 0 from three pairings T7SEN approves: a display face, a text face, and a mono face for the readout. The fonts are self-hosted, and their licenses must allow web embedding.

## 7. Conventions

- **Code.** TypeScript strict. Files in kebab-case. Named exports only. No magic numbers: constants live in `tokens` or a named `constants.ts`.
- **Engine.** No React or DOM framework imports. Kernels are named `kernelX`, fields `fieldX`. Uniforms come only from `SceneState`.
- **CSS.** Token variables only. Tailwind v4 `@theme` comes from `packages/tokens`. Animate `transform` and `opacity` only.
- **Copy.** Sentence case, first person, direct, and short. Banned words: seamless, leverage, unlock, empower, simply, just. No placeholder copy ships.
- **Commits.** Conventional Commits, one concern per commit.
- **ADRs.** `docs/adr/NNNN-kebab-title.md` with Context, Decision, Consequences, and Alternatives.
- **Walkthroughs.** `docs/walkthroughs/<subsystem>.md`: the code path, why it is shaped that way, and what would break it.
- **Homage.** The red, blue, and purple concept is an homage expressed in mechanics and internal token names only. No series names, characters, or symbols appear on the public site.

## 8. Environment variables

| Name | Where | Purpose |
|---|---|---|
| `NEXT_PUBLIC_SITE_URL` | `apps/web` build | Canonical URL |
| `NEXT_PUBLIC_ASSET_BASE_URL` | `apps/web` build | R2 public domain for large media |
| `NEXT_PUBLIC_TURNSTILE_SITE_KEY` | `apps/web` build | Turnstile widget |
| `TURNSTILE_SECRET_KEY` | `apps/edge` secret | Server-side Turnstile verification |
| `DISCORD_WEBHOOK_URL` | `apps/edge` secret | Contact notifications |
| `ALLOWED_ORIGIN` | `apps/edge` var | Origin check on `/api/contact` |

## 9. Commands

| Command | Does |
|---|---|
| `pnpm dev` | Content layer dev server |
| `pnpm build` | tokens → contracts → engine → static export |
| `pnpm typecheck` / `pnpm lint` / `pnpm test` | Static checks and unit tests |
| `pnpm test:e2e` | Playwright: routes, palette, contact |
| `pnpm test:visual` | Scene keyframe screenshots in deterministic mode |
| `pnpm test:webgl` | Full suite with the renderer forced to WebGL 2 |
| `pnpm assets` | Compress models and textures |
| `pnpm capture` | Record Tier D captures and upload them to R2 |
| `pnpm deploy:preview` / `pnpm deploy` | Wrangler deploy |

## 10. Landmines

1. **Two copies of three.** Mixing `three` with `three/webgpu` loads two module instances. Import only `three/webgpu` and `three/tsl`.
2. **Compute is not portable.** Atomics and indirect draws exist only on WebGPU and cannot be emulated in WebGL 2. Tier C has its own kernels, verified with `forceWebGL`.
3. **StrictMode double-mount.** React development mode runs effects twice. A non-idempotent engine initialization creates two GPU devices.
4. **Canvas inside a page.** Mounting the engine in a page component destroys the GPU device on navigation. It lives in the root layout only.
5. **Two clocks.** Any second `requestAnimationFrame` loop drifts against scroll. Everything runs on `gsap.ticker`.
6. **Blur over the canvas.** `backdrop-filter` over the live canvas forces full-screen compositing every frame and breaks frame pacing.
7. **Static export limits.** Server actions, route handlers, middleware, ISR, and the image optimizer do not exist in this build.
8. **Free-tier file caps.** Workers static assets allow 20,000 files per deployment and 25 MiB per file on the free plan. Larger media goes to R2.
9. **Device lost.** An unhandled `device.lost` leaves a black canvas. Recover or drop a tier.
10. **HDR is custom.** `toneMapping: 'extended'` is not Baseline, and three.js HDR output tone mapping was incomplete as of October 2026. Capability-gate it and verify current status before building on it.
11. **Nondeterministic baselines.** GPU output varies by device. Visual tests require deterministic mode (seeded RNG, fixed timestep, keyframe scrub). WebGPU baselines are generated on the reference machine; CI runs Tier C under software rendering with a tolerance threshold.
12. **Red and blue in chrome.** `reversal` and `lapse` are simulation colors. Using them in UI breaks the single-accent system.

## 11. Known tech debt

None yet. Record every entry with a label: **Intentional decision** (a tradeoff made on purpose, with its ADR) or **Genuine bug** (behavior that is wrong). Improvements are deferred until T7SEN schedules them.

## 12. Pre-finish checklist

- [ ] `pnpm typecheck`, `pnpm lint`, and `pnpm test` pass.
- [ ] `pnpm build` succeeds, and no server-only Next.js feature was introduced.
- [ ] Engine change: `pnpm test:webgl` passes.
- [ ] Engine change: the readout shows a steady 60 fps on the reference GPU.
- [ ] Visual baselines changed only deliberately, and the summary names them.
- [ ] Reduced motion and keyboard paths still work.
- [ ] No `reversal` or `lapse` in UI; no white labels on `hollow`.
- [ ] No unpinned dependency was added.
- [ ] An ADR covers each decision; a walkthrough covers non-obvious code.
- [ ] This file is updated if structure, commands, conventions, or landmines changed, with every path verified.
