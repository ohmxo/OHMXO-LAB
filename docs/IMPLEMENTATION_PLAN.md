# IMPLEMENTATION PLAN — Improving the Current Features

> Living companion to ^[`BRAINSTORM.md`](../BRAINSTORM.md) §7. This plan turns the
> feature-by-feature ideas there into ordered, independently shippable work. Each task
> leaves the app buildable (`npm run build`) and testable (`npm test`).
>
> **Priority logic:** correctness wins (§7.1) come first because they make the site
> cooler simply by working as designed. Then the highest visible-impact upgrades
> (§7.2 monitor, §7.3 steam, §7.6 BIOS) plus the desk-lamp toggle. Rebrand now has
> real tokens from the Ohmxo knowledge base (agency = **Ohmxo**, founder = **Jacob**,
> ecosystem = **Nexus** web / **Agentix** AI / Digital Marketing).

## Scope

- **In:** Fix the §7 bugs (⚠️) across camera, monitor, audio, film grain, renderer,
  hitboxes; the highest-value visible upgrades (§7.2 dimmer/glare, §7.3 steam layers/
  mouse, §7.6 BIOS easter eggs, §7.4 audio fixes + ducking) **and the desk-lamp toggle
  (new §7.9)**; Ohmxo/Jacob branding where text appears (BIOS boot, monitor iframe
  title, loading screen); per-file unit tests; Vitest already scaffolded.
- **Out:** Full domain/GA/OS-URL swap (hardcode deferred until the inner site is chosen);
  day/night, second room, dust motes, scroll storytelling (§4 backlog); audio-reactive
  analyser→uniforms (depends on the pool-key fix settling first); own inner OS in the
  iframe (separate subsystem).

## Action items

[ ] **1. Fix the camera motion bugs** — `src/Application/Camera/CameraKeyframes.ts`,
    `Camera.ts`. Replace `new Time()` in `IdleKeyframe` with `application.time`; switch
    the `* 0.05` desk parallax to `THREE.MathUtils.damp(…, delta)`; delete the dead
    `this.position.z = this.position.z;`; gate the global IDLE↔DESK `mousedown` so it
    never fires during free-cam, over hotspots, or on the iframe.

[ ] **2. Add camera tests** — `src/Application/Camera/__tests__/`. TDD the
    frame-rate-independent damping and the IDLE↔DESK gating logic as pure functions so
    they don't need the DOM. Run `npm test`.

[ ] **3. Make the monitor dimmer real** — `src/Application/World/MonitorScreen.ts`.
    Switch the `dimmingPlane` from black+`AdditiveBlending` to `NormalBlending` so the
    angle/distance math in `update()` actually darkens the reflection stack at glancing
    angles; if it stays invisible, replace the approach (per §7.2). Rebrand the iframe
    `title` to "Ohmxo" (from `HeffernanOS`).

[ ] **4. Add camera-tracking glare + power button** — same file. Modulate the
    reflection-layer opacity by the view dot-product (reflections "slide" as you orbit);
    add a bezel power button (click → sleep state: black + tiny LED, click → wake). Pure
    DOM + one fade, no shader changes.

[ ] **5. Add the desk-lamp toggle** — `NEW src/Application/World/DeskLamp.ts` +
    `World.ts`. Constraint: the whole scene is baked-lit (`BakedModel` → `MeshBasicMaterial`,
    zero dynamic lights), so the lamp's glow is baked into the texture as "on". Add a small
    emissive glow mesh + a soft light-spill gradient over the desk that fades in/out when the
    lamp is toggled, plus a desk hotspot (→ step 10) so the lamp is clickable (or a keyboard
    key). Start from the baked "on" state; "off" dims the glow plane rather than re-baking.
    Verify final look in `npm run dev`.

[ ] **6. Upgrade coffee steam** — `src/Application/World/CoffeeSteam.ts` +
    `src/Application/Shaders/coffee/*`. Add a second, faster outer steam layer (different
    UV frequency) and switch to additive blending; add a `uMouseNear` uniform so the
    steam scatters near the cursor.

[ ] **7. Fix the audio engine** — `src/Application/Audio/AudioManager.ts`,
    `AudioSources.ts`. Use a monotonic counter for `poolKey` (stop key collisions from
    the shrinking pool); floor the ambience filter at ~100 Hz and re-tune the 0.1 volume
    clamp; ramp `playAudio` gain over ~50 ms to kill clicks/pops; persist mute to
    `localStorage`. Add unit tests for the pool-key counter and the frequency floor.

[ ] **8. Add ambience ducking on monitor entry** — same files. `enterMonitor` dips +
    lowpasses the ambience; `leftMonitor` crossfades back (not a snap). Drive off the
    existing `enterMonitor`/`leftMonitor` camera events.

[ ] **9. Fix the film-grain + overlay** — `src/Application/Renderer.ts`,
    `src/Application/Shaders/screen/*`. Replace `u_time = Math.sin(time.current*0.01)`
    with a steadily increasing value; fix the invalid `zIndex = '1px'` → `'1'`; add a
    subtle vignette to the same fullscreen pass.

[ ] **10. Make the BIOS boot screen interactive + rebrand** — `src/Application/UI/components/LoadingScreen.tsx`.
    Wire progress to real load events (progress = `loaded/toLoad`); add ESC to
    fast-forward the RAM check and DEL to open a fake BIOS setup panel; replace the
    "Heffernan / Henry Inc." boot logo + "Henry Heffernan Portfolio Showcase" text with
    **Ohmxo / Jacob** branding (keep the BIOS aesthetic — it's the site's personality);
    optionally reuse the `_AUTO_` typing sound for a boot-log typewriter effect.

[ ] **11. Finish the hotspots (hitbox) system** — `src/Application/World/Hitboxes.ts`,
    `World.ts`, `Camera.ts`. Fix the class-name bug, remove console spam + the global
    `preventDefault` mousedown, and the `RENDER_WIREFRAME` opacity-1 override; enable
    `new Hitboxes()`. Wire hover → pointer/tooltip, click → camera focus + click sound,
    click empty space → return to desk. Add a **desk-lamp hotspot** toggling step 5's
    lamp. Coordinate with step 1's gating so hotspot clicks win over the IDLE↔DESK toggle.

[ ] **12. Cleanup + cross-cutting** — remove dead `idle` self-assignment fallout, pause
    the `Time` rAF loop on `document.hidden`, add dynamic resolution scaling when frame
    time > 40 ms, type the `postMessage` contract in `MonitorScreen.ts`.

[ ] **13. Full validation** — `npm test`, `npm run build`, then `npm run dev` and
    manually click through: idle cover, monitor entry/exit, hotspot focus + return,
    desk-lamp on/off, power-button sleep/wake, steam mouse reaction, BIOS DEL/ESC,
    Ohmxo boot branding, mute persistence.

## Open questions

- **Lamp model** — the decor comes from baked `.glb` models with no named lamp mesh in
  source. Can you point out the lamp's approximate position ($x, y, z$) in the scene, or
  should I probe the baked model/UVs at runtime to locate it before placing the glow?
- **Hotspot targets** — besides the desk lamp and coffee cup, which decor is clickable
  (plant? headphone?), and does each get a tooltip or a camera focus?
- **Inner-OS URL** — the monitor iframe is hardcoded to Henry's site. Do you want me to
  point it at your own inner site (build a 2D Ohmxo OS, or reuse an existing URL) as part
  of this plan, or keep it as a separate subsystem?
- **Scope lock** — should the §4 backlog features (day/night, second room) be sequenced
  after this plan, or held as separate follow-on plans?
