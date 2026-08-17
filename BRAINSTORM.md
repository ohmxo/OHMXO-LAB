# BRAINSTORM — Portfolio Website Improvements

> Working ideas, analysis, and feature backlog for the 3D Three.js portfolio.
> This is a living document. Move decided items into an implementation plan.

---

## 1. What this project is

A **Three.js interactive 3D portfolio** built on the Bruno Simon *threejs-journey*
pattern. A stylized desk/room is the scene; the centerpiece is a **real, live
computer monitor** that renders an iframe (Henry Heffernan's inner OS site)
via a `CSS3DRenderer`, composited with 4 stacked render layers.

**Important:** this repo is a **fork of Henry Heffernan's 2022 portfolio**.
Branding, metadata, the iframe URL, and Google Analytics ID are all his. To
make it *ours*, rebranding is the highest-priority first move.

## 2. How the scene works (architecture map)

| Layer | Container | Renderer | Draws |
|-------|-----------|----------|-------|
| CSS3D | `#css` | `CSS3DRenderer` | Live iframe (computer screen) |
| WebGL main | `#webgl` | `WebGLRenderer` | 3D room, desk, decor, baked models |
| Overlay | `#overlay` | WebGL (soft-light) | Fullscreen film-grain shader |
| React UI | `#ui`, `#ui-interactive` | React DOM | Loading, intro text, toggles |

**Flow:** `script.ts` → `Application` singleton → `Resources` loads 18+ assets →
`'ready'` → `World` builds scene → `LoadingScreen` → camera keyframe sweep → audio starts.

**Camera:** 5 keyframes (`idle`, `desk`, `monitor`, `loading`, `orbitControlsStart`)
tweened with `@tweenjs/tween.js`. `DeskKeyframe` = mouse parallax, `IdleKeyframe` =
slow sin drift, `MonitorKeyframe` = aspect-responsive zoom.

**Monitor realism:** CSS3D iframe + stacked GL layers (smudge, innerShadow, 2
VideoTextures) + enclosing planes + a dynamic `dimmingPlane` that reacts to camera
angle/distance.

**Shaders:** coffee steam (Perlin displacement, hand-written) + fullscreen golden-ratio
hash noise film grain.

**Audio:** positional + global; keystroke/mouse-clack sounds gated by `event.inComputer`;
office ambience maps camera-distance → lowpass filter + volume.

**Performance secret:** `BakedModel.ts` applies one baked lightmap to all meshes with
`MeshBasicMaterial` → zero real-time lighting cost.

## 3. Confirmed issues (fix these)

- [x] **`Hitboxes.ts:8` declares `class Decor`** (copy-paste bug) — should be `class Hitboxes`. Breaks import of `Hitboxes`.
- [x] Hitbox system scaffolded but **disabled** — `new Hitboxes()` commented out in `World.ts:38`.
- [x] Heavy `console.log` debug spam in `Hitboxes.ts` `createRaycaster()`.
- [x] `Cursor.ts` unused. `window.Application = this` commented out.
- [x] Duplicate logic between `Decor.ts` and `Hitboxes.ts`.
- [ ] `Resources` has **no error handling** — one failed asset hangs the loader forever.
- [ ] `Renderer.ts:59` sets `zIndex = '1px'` (invalid value → coerced to `1`).
- [ ] All meta/OG/Twitter tags + GA ID are Henry's. `package.json` repo/license not set.
- [ ] No CSP header on the served page; GA inline script needs `'unsafe-inline'`/nonce.
- [ ] Hover-based `inComputer` model breaks on touch/mobile; no `touchstart` handling.
- [ ] No `prefers-reduced-motion` support.

## 4. Creative features (backlog)

1. **Audio-reactive scene** — feed WebAudio analyser into shader uniforms (steam, grain, lamp glow).
2. **Interactive decor hotspots** — finish the hitbox system: click coffee/lamp/plant → tooltip or mini camera focus. *(highest wow-per-effort, scaffolding exists)*
3. **Day/night cycle** — crossfade two baked textures + flickering lamp glow; tie to the on-screen clock.
4. **Connected second room** — door click triggers a 3D pan; reuses keyframe + baked pipeline.
5. **Coffee steam reactive to mouse proximity** — add `uMouseNear` uniform.
6. **CRT boot sequence** — green scanlines + "boot" text before iframe appears; uses `startup.mp3`.
7. **Ambient life** — dust motes / occasional bird for idle orbit.
8. **Scroll-based camera storytelling** — `gsap` + `ScrollTrigger` (gsap already a dep) to narrate resume sections.
9. **Bloom / vignette post-processing** — `EffectComposer` + `UnrealBloomPass` so the bright monitor glows into the dark room.
10. **Own inner site in the iframe** — build a 2D OS-like inner site; point `iframe.src` at it (`?dev` → `localhost:3000`).

## 5. Improvement themes (non-feature)

- **Untangle the `Application` singleton** — DI over global `new Application()` for testability.
- **Dead code + debug cleanup** — remove comments, unused files, log spam.
- **Types** — remove `@ts-ignore`/`any`; type the `postMessage` contract.
- **Resources resilience** — loader error/timeout + retry.
- **Security** — CSP, iframe `frame-ancestors`, replace GA ID, correct `package.json`.
- **Perf** — webpack bundle budget, lazy-load heavy deps, tree-shake three.
- **A11y** — `prefers-reduced-motion`, `aria-live` overlays, keyboard path for `enterMonitor`.
- **Mobile** — touch handling for the hover-driven monitor interaction.

---

## 6. My recommended first move

**Phase order (each independently shippable):**

1. **Rebrand** — make it ours (metadata, README, package.json, remove Henry's GA).
2. **Enable + finish the hitbox interaction system** — the biggest wow that's already 80% scaffolded.
3. **Clean dead code + console.logs** — de-risk before adding features.
4. **Resources error handling** — stop the hang-on-failure footgun.
5. **Bloom post-processing** — transform the mood cheaply.
6. **Own inner site in the iframe** — separate plan (whole other subsystem).

See `docs/superpowers/plans/2026-08-17-improve-portfolio.md` for the executable plan.
