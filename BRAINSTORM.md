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

## 7. Improve what already exists (feature-by-feature deep dive)

> The inverse of §4: instead of adding features, how do we make each **current** feature
> dramatically better? Every item below is grounded in the actual source — `file:line`
> refs included so nothing is hand-wavy. ⚠️ = real bug found in the current code.

### 7.1 Camera & motion — `Camera.ts`, `CameraKeyframes.ts`, `Mouse.ts`

**Bugs (fix first):**
- ⚠️ `IdleKeyframe` constructs its own `new Time()` (`CameraKeyframes.ts:122`) instead of using `application.time`. Every `Time` spawns its own `requestAnimationFrame` loop (`Time.ts:19-38`) and re-registers the `loadingScreenDone` reset — so the app runs a duplicate rAF loop whose clock drifts from the real one.
- ⚠️ Desk parallax is frame-rate dependent: `targetFoc.x += (target - current) * 0.05` per **frame** (`CameraKeyframes.ts:94-104`). 144 Hz moves ~2.4× faster than 60 Hz. Replace with `THREE.MathUtils.damp(current, target, lambda, delta)`.
- ⚠️ `IdleKeyframe.update` ends with `this.position.z = this.position.z;` — dead self-assignment (`CameraKeyframes.ts:132`).
- ⚠️ Global `mousedown` toggles IDLE↔DESK on *every* click (`Camera.ts:69-85`). Once hotspots + monitor clicks exist this will fight them — gate it (ignore clicks on hotspots / iframe / UI).

**Upgrades:**
- **FOV kick** on transitions: push to ~42° during monitor entry, settle back to 35. Cheap, very cinematic.
- **Look-ahead focal point**: during tweens, offset the focal point slightly in the direction of travel.
- **Keyboard presets**: `1` idle · `2` desk · `3` monitor · `F` free-cam — ~20 lines reusing `transition()`.
- Disable the IDLE↔DESK toggle while free-cam is active (it currently still fires).

### 7.2 Monitor realism — `MonitorScreen.ts`

**Bugs:**
- ⚠️ The perspective `dimmingPlane` is a **no-op**: black `MeshBasicMaterial` + `THREE.AdditiveBlending` (`MonitorScreen.ts:455-461`) adds `black × alpha` to the buffer — i.e. nothing. The angle/distance math in `update()` (`:497-525`) has zero visual effect. Fix: `NormalBlending` (darkens the GL reflection stack at glancing angles) or delete it.
- ⚠️ `getVideoTextures` polls `setTimeout(…, 100)` forever when the element is missing (`:318-329`), and `VideoTexture`s of unloaded videos render black — no `loadedmetadata`/`error` handling.
- ⚠️ Enclosing planes are flat `0x48493f` rectangles (`:377-453`) — at glancing angles they read as flat squares floating behind the screen.
- Rebrand: `iframe.title = 'HeffernanOS'` + iframe URL (`:187, 207`).

**Upgrades:**
- **Real angle-based dimming** of the smudge/reflection stack once the blending bug is fixed (machinery already exists in `update()`).
- **Camera-tracking glare**: modulate the reflection layer's opacity with the view-dot product so reflections "slide" as you orbit.
- **Screen power button**: click the bezel → fade to sleep (black + tiny LED), click again to wake. All DOM + one fade.
- **Fake GI spill**: soft emissive gradient plane under the monitor so screen light spills onto the desk (the only cheap dynamic light, since the scene is baked-lit).
- Combine with bloom (§4.9) so the screen actually glows.

### 7.3 Coffee steam — `CoffeeSteam.ts` + `Shaders/coffee/*`

- **`uMouseNear` uniform** — steam curls/scatters when the cursor is close (this is an upgrade to *existing* steam, not a new feature).
- **Two-layer steam**: dense slow inner core + thin fast outer wisp (different UV frequencies) — one extra draw call, reads as real volume.
- Switch to **additive blending** — translucent white over the baked scene looks brighter/more ethereal.
- **Color ramp uniform** to tint steam with lamp glow (pairs with day/night later).
- **Audio pulse**: tie `uTimeFrequency` to the ambience analyser (pairs with §7.4).

### 7.4 Audio — `AudioManager.ts`, `AudioSources.ts`

**Bugs:**
- ⚠️ `poolKey = sourceName + '_' + Object.keys(this.audioPool).length` (`AudioManager.ts:71`) — the pool *shrinks* as sounds end (`onended` deletes), so a new sound can steal the key of one still playing and get clobbered. Use a monotonic counter.
- ⚠️ `AmbienceAudio.update` clamps volume to max `0.1` (`AudioSources.ts:120`) and sets filter freq `freq - 3000` (`:122`) which can go ≤ 0 → a degenerate 0 Hz biquad. Floor at ~100 Hz, re-tune the clamp.
- ⚠️ `playAudio` plays at full volume instantly → click/pop. Ramp gain over ~50 ms.
- Mute isn't persisted (`AudioManager.ts:43-45`) — save to `localStorage`.

**Upgrades:**
- **Analyser → shader uniforms** (steam pulse, grain intensity, lamp glow) — the §4.1 audio-reactive scene as an upgrade to the existing audio + shader stack.
- **Ducking**: `enterMonitor` → ambience dips + lowpass; `leftMonitor` → restore. Crossfade, not snap.
- **Cheap room reverb**: generate a 1 s noise-burst impulse and run the *computer* sounds (keystrokes/clicks) through a `ConvolverNode` — instant "in the room" quality.
- **Organic keystrokes**: randomize detune ±30 cents per keydown (the `randDetuneScale` plumbing already exists).

### 7.5 Film grain & screen overlay — `Renderer.ts`, `Shaders/screen/*`

- ⚠️ `u_time` is `Math.sin(time.current * 0.01)` (`Renderer.ts:118`) — the grain seed oscillates at non-uniform speed. Pass a steadily increasing value so the grain evolves at constant speed.
- ⚠️ `zIndex = '1px'` (`Renderer.ts:59`) — invalid, coerces to `1`. Use `'1'`.
- Upgrades: **vignette** (~10% corner darkening) in the same fullscreen pass; grain intensity modulated by scene darkness or audio; optional **scanlines masked to the monitor quad** for CRT texture.

### 7.6 Loading screen / BIOS boot — `LoadingScreen.tsx`

The BIOS theme *is* the site's personality — improve the existing one, don't replace it:
- Make the footer real: **ESC fast-forwards the RAM check**, **DEL opens a fake BIOS setup screen** (clock/date + your branding in the `HHBIOS (C)` style) — pure React.
- Wire the % to actual `Resources` load events (`loadedSource` already flows; the `useEffect` on `loaded` drives a `counter` that can diverge — make progress = `loaded/toLoad`).
- Reuse the existing `ccType`/`_AUTO_` typing sound (`AudioSources.ts:49-55`) for a boot-log typewriter effect.
- Touch: "tap the screen" affordance instead of hover-only monitor entry (§3 mobile item).

### 7.7 Hitboxes → hotspots (scaffolded feature, finished properly)

- Fix: class name (`Hitboxes.ts:8`), console spam + global `preventDefault` mousedown (`:32-55`), and the `RENDER_WIREFRAME` opacity-1 override (`:6, 75-81`) — as written it would render **red opaque boxes** in the scene.
- Upgrade: hover → `cursor: pointer` + tooltip (HTML overlay projected from the hitbox center); click → camera focus + click sound; click empty space → return to desk. Coordinate with the IDLE↔DESK toggle (§7.1) so hotspot clicks win.
- A11y: Tab-focusable hotspots with Enter trigger + `aria-label`s.

### 7.8 Performance & feel (cross-cutting)

- Frame-rate-independent damping everywhere (the `* 0.05` lerps in `DeskKeyframe`, etc.).
- `Time` keeps a rAF loop alive when the tab is hidden (`Time.ts:36-38`) — pause on `document.hidden`.
- Pixel ratio capped at 2 (`Renderer.ts:54`) — add dynamic resolution scaling (drop to 1.5 when frame time > 40 ms).
- Lazy-load the iframe until first interaction (defer the OS site until "START").
- Type the `postMessage` contract (`MonitorScreen.ts:150-181` is all `// @ts-ignore` + untyped `event.data.type`) — the most untyped surface in the app.

### 7.9 Wow idea — toggleable desk lamp

> **Branding note for the whole repo:** it's **Ohmxo** (Jacob's digital-architecture
> agency — Nexus web / Agentix AI / Digital Marketing). Replace "Heffernan / Henry Inc."
> and "Henry Heffernan Portfolio Showcase" in `LoadingScreen.tsx`, `readme.md`, and the
> monitor iframe `title = 'HeffernanOS'` with Ohmxo/Jacob text.

- **Constraint:** the scene is baked-lit (`BakedModel.ts` gives every mesh one
  `MeshBasicMaterial.map` — zero dynamic lights). So the lamp's "on" glow is already fired
  into the baked texture. A real `PointLight` would wash out the baked look and cost perf.
- **Approach:** instead of a light, add a small **emissive glow mesh** + a soft
  **light-spill gradient plane** over the desk, faded in/out when toggled. Start "on";
  "off" dims the glow plane rather than re-baking. Registers as a **hotspot** (§7.7) so the
  lamp is clickable, plus a keyboard key.
- **Extra juice:** tie the glow flicker or intensity to the film-grain/audio (§4.1) later.
- **Open:** the lamp lives in `static/models/Decor/decor.glb` — no named mesh in source,
  so pin down its position before placing the glow.

**How to pick:** every ⚠️ above is a *correctness* win (the site gets cooler simply by working as designed). The biggest *visible* upgrades are §7.2 (dimmer fix + glare + power button), §7.3 (steam layers + mouse), §7.9 (desk lamp), and §7.6 (BIOS easter eggs + Ohmxo rebrand). Start there.

See `docs/superpowers/plans/2026-08-17-improve-portfolio.md` for the executable plan.
