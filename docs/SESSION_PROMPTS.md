# Session prompts — t7sen.com v2

Use the Long Form to open a phase and the Short Form for small tasks inside a phase. Append the matching phase addendum to either one.

## Long Form

```
You are working on t7sen.com v2. Before anything else, read AGENTS.md, SKILL.md, and docs/SPEC.md in full. They are canonical.

Phase: <N — name>. Task: <one-sentence goal>.

Start by giving me:
1. Your understanding of the task in three sentences or fewer.
2. Every file you expect to create or modify.
3. The SKILL.md landmines this task touches, by number.
4. Any API or version-specific behavior you need that SKILL.md does not document, with the official source you will check.
5. Assumptions and open questions.

Wait for my approval before writing code.

While implementing:
- Work in small, reviewable steps and show each diff with a one-line rationale.
- Stay inside the task. List anything else you notice under "Flagged, not fixed".
- Simulation kernels are pair-mode: draft, explain the math and the data flow, then stop for my sign-off.

Before you finish:
- Run the SKILL.md pre-finish checklist and report each item as pass, fail, or not applicable.
- Write ADRs for decisions and walkthroughs for non-obvious code.
- Update SKILL.md if structure, commands, conventions, or landmines changed, and confirm every path you wrote exists.
- End with a summary: what changed, what was verified, what is flagged, and what the next step is.
```

## Short Form

```
t7sen.com v2. Read AGENTS.md first; consult SKILL.md as needed.
Task: <task>. Scope: <files or subsystem>.
Stay in scope, verify anything SKILL.md doesn't document, run the pre-finish checklist, and report what you flagged but didn't fix.
```

## Phase addenda

### Phase 0 — Foundations
```
Objective: scaffold the monorepo exactly as SKILL.md section 3 describes.
- Research the current stable version of every manifest entry from official release notes; pin exact versions in package.json and fill the Pinned column, citing sources.
- Set up the tokens package build (OKLCH source → tokens.css + tokens.ts) using the approved palette, unchanged.
- Build the static shell: Void set inline in the head, theme-color, color-scheme dark, no white flash on a throttled load.
- Configure wrangler.jsonc for one Worker with static assets from apps/web/out and /api/* routed to the script.
- CI: typecheck, lint, unit tests, build, Playwright smoke test.
- Typography: propose three pairings (display, text, mono) with license notes, rendered against the palette with the mark. Stop for my choice.
Stop and ask if: any manifest floor is no longer valid, or the free-plan limits in SKILL.md have changed.
```

### Phase 1 — Engine core
```
Objective: the engine's skeleton, proven with a placeholder particle field.
- createEngine with an idempotent singleton init (landmine 3), EngineHost in the root layout only (landmine 4).
- Tier probe, first-visit benchmark, localStorage cache under t7:tier, palette-ready override API.
- One clock on gsap.ticker with Lenis and ScrollTrigger (landmine 5); SceneState scroll bridge.
- Deterministic mode: seeded RNG, fixed timestep, keyframe scrub.
- Readout, including GPU time via timestamp-query when available.
- device.lost handling (landmine 9).
Proof: the field runs on Tiers A and C, survives navigation and a StrictMode double-mount, and responds to scroll.
```

### Phase 2 — Simulation, WebGPU (pair mode)
```
Objective: the signature sequence, from hero orbit through collision and fusion to the resolved mark.
- Before each kernel: explain the math, buffer layout, and per-frame cost, then wait for my go.
- Kernels: curl noise, reversal repulsion, lapse attraction, fusion, mark SDF attraction, cursor force.
- Mark SDF: analytical signed distance to the 14-vertex polygon in SKILL.md section 6.4; sign by winding number; no baked texture.
- Set particle budgets by benchmark on the reference GPU; document them in an ADR.
- Write a walkthrough per kernel.
Stop after each kernel for my sign-off. Do not batch kernels.
```

### Phase 3 — Fallback tiers
```
Objective: Tier C and Tier D deliver deliberate images, not degraded copies.
- Tier C: portable kernels with no atomics or indirect draws (landmine 2) and their own art direction; propose that direction before building.
- Dynamic resolution: hold the frame budget; drop resolution and particles before smoothness; drop a tier only at the resolution floor.
- Capture pipeline: deterministic stills and loops per keyframe, uploaded to R2; respect the file caps (landmine 8).
- Visual baselines per tier (landmine 11).
```

### Phase 4 — Content layer
```
Objective: every route complete and usable with no GPU.
- Sections per docs/SPEC.md section 4; Ink reading planes over the canvas at 92% opacity, no backdrop blur (landmine 6).
- Case studies with the structure in SPEC section 8.2. Ask me for every fact you do not have; never invent figures.
- Draft copy in the voice from SPEC section 8.1 and submit it for my review before it ships.
- Contact: shared Zod schema in packages/contracts, Turnstile verified in the Worker, honeypot, Discord webhook embed, both intents.
- Metadata and Open Graph images per route.
```

### Phase 5 — Choreography and models
```
Objective: the experience map plays end to end at full fidelity.
- Scroll timelines per section driving SceneState only.
- Model targets: run models through tools/assets and sample surfaces into target buffers.
- Page transitions that keep the canvas alive; the preloader reporting real progress; the custom cursor as a simulation force.
- Tune the motion tokens and define the collide CustomEase; record the final values in SKILL.md section 6.5.
- Reduced-motion path per SPEC section 7.
```

### Phase 6 — Command palette and easter eggs
```
Objective: both built from scratch.
- Palette: ARIA combobox and listbox, fuzzy search, action registry per SKILL.md 5.10; verify with keyboard and a screen reader.
- Easter eggs: propose at least five concepts that each drive the engine; I pick at least three. No series names, characters, or symbols.
```

### Phase 7 — HDR and P3
```
Objective: the collision core outshines SDR white on HDR displays without affecting anything else.
- Verify current browser and three.js HDR support before starting (landmine 10); report what changed since SKILL.md was written.
- P3 accent tokens under @media (color-gamut: p3); canvas colorSpace display-p3 where supported.
- HDR overlay: capability-gated, custom tone mapping, documented in an ADR and a walkthrough.
```

### Phase 8 — Hardening and launch
```
Objective: launch t7sen.com v2 on Cloudflare.
- Re-verify the browser matrix in SPEC section 5.1 and update it.
- Frame pacing on the reference GPU and an integrated-GPU laptop; reduced-motion and keyboard audits.
- DNS cutover plan with a rollback step; execute only after my approval.
- Archive the v1 repository.
- Full pre-finish checklist.
```

## Notes on use

- Fill in the open inputs in docs/SPEC.md section 14 as each phase needs them; agents must ask rather than invent.
- One phase per session where possible. Long sessions compact; the repo docs are the memory.
- After each phase, update SKILL.md before starting the next one, so the next session starts accurate.
- If an agent proposes something outside the spec, treat it as a flagged idea, not a decision.
