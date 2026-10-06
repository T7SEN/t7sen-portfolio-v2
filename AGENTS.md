# AGENTS.md — t7sen.com v2

Read this at the start of every session. Operational reference: `SKILL.md`. Product spec: `docs/SPEC.md`.

## Project

Portfolio for T7SEN, replacing t7sen.com v1. One persistent GPU simulation, choreographed by scroll, is the protagonist: a red field and a blue field collide, fuse into purple, and resolve into the T7 mark. The site serves two goals, full-time developer roles and freelance client work. Visual fidelity outranks load speed; frame pacing is non-negotiable because stutter is a visual defect.

## Setup

- Node 24 LTS and pnpm, run from the workspace root.
- `pnpm install`, then `pnpm dev` (content layer on http://localhost:3000).
- Worker secrets live in `apps/edge/.dev.vars` and public build variables in `apps/web/.env.local`. Neither is committed.

## Layout

| Path | Purpose |
|---|---|
| `apps/web` | Next.js 16 static export: routes, content layer, case studies, palette, contact form |
| `apps/edge` | Cloudflare Worker (Hono): serves the static build, handles `POST /api/contact` |
| `packages/engine` | Framework-agnostic three.js WebGPU engine: tiers, simulation, scroll bridge, readout |
| `packages/tokens` | Design tokens: single source for CSS variables and shader constants |
| `packages/contracts` | Zod schemas shared by `apps/web` and `apps/edge` |
| `packages/config` | Shared tsconfig and ESLint config |
| `tools/assets` | glTF and texture compression pipeline |
| `tools/capture` | Records the live simulation for the static tier |
| `docs/adr` | Architecture decision records |
| `docs/walkthroughs` | Annotated walkthroughs of non-obvious code |

## Conventions

- TypeScript strict everywhere. No `any`, no non-null assertions without a comment explaining why.
- `packages/engine` never imports React or Next.js.
- three.js is imported only from `three/webgpu` and `three/tsl`.
- Colors, eases, and durations come only from `packages/tokens`.
- One clock: `gsap.ticker` drives Lenis and the engine. No other `requestAnimationFrame` loops.
- UI copy is sentence case, first person, direct.

## Critical rules

1. Static export only. No server actions, route handlers, middleware, ISR, or image optimizer in `apps/web`.
2. Every engine change passes the forced-WebGL run (`pnpm test:webgl`).
3. Simulation kernels are pair-mode work: draft and explain, then stop for T7SEN's sign-off before merging.
4. No scope additions, refactors, or improvements beyond the task. Flag them; T7SEN decides when they happen.
5. Before using any API not documented in `SKILL.md`, verify it against the official docs for the pinned version.

## Testing

- `pnpm typecheck && pnpm lint && pnpm test` before every commit.
- `pnpm test:e2e` covers routes, the command palette, and the contact flow.
- `pnpm test:visual` compares scene keyframes in deterministic mode. Baseline updates are deliberate and called out in the summary.
- Engine changes: confirm a steady 60 fps on the reference GPU (RTX 4060 Ti class) using the readout.

## Finish

Run the pre-finish checklist in `SKILL.md` section 12. Record decisions in an ADR and non-obvious code in a walkthrough.
