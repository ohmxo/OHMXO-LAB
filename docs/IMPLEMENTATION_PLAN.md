# IMPLEMENTATION PLAN — Improving the Current Features

> Living companion to ^[`BRAINSTORM.md`](../BRAINSTORM.md) §7. This plan turns the
> feature-by-feature ideas there into ordered, independently shippable work. Each task
> leaves the app buildable (`npm run build`) and testable (`npm test`).
>
> **Priority logic:** correctness wins (§7.1) come first because they make the site
> cooler simply by working as designed. Then the chosen visible upgrades (§7.2 dimmer,
> §7.6 BIOS) plus the desk-lamp toggle. (Rebrand to Ohmxo is already complete.)

## Scope

- **In:** Fix the §7 bugs (⚠️) across camera, monitor, audio, film grain, renderer;
  the chosen visible upgrades (§7.2 dimmer, §7.6 BIOS easter eggs, §7.4 audio fixes +
  ducking) **and the desk-lamp toggle (new §7.9)**; per-file unit tests; Vitest already
  scaffolded.
- **Out:** Full domain/GA/OS-URL swap (hardcode deferred until the inner site is chosen);
  day/night, second room, dust motes, scroll storytelling (§4 backlog); audio-reactive
  analyser→uniforms (depends on the pool-key fix settling first); own inner OS in the
  iframe (separate subsystem).

## Action items

[ ] **1. Fix the camera motion bugs** — `src/Application/Camera/CameraKeyframes.ts`,
    `Camera.ts`. Replace `new Time()` in `IdleKeyframe` with `application.time`; switch
    the `* 0.05` desk parallax to `THREE.MathUtils.damp(…, delta)`; delete the dead
    `this.position.z = this.position.z;`; gate the global IDLE↔DESK `mousedown` so it
    never fires during free-cam or on the iframe.

[ ] **2. Add camera tests** — `src/Application/Camera/__tests__/`. TDD the
    frame-rate-independent damping and the IDLE↔DESK gating logic as pure functions so
    they don't need the DOM. Run `npm test`.

[ ] **3. Make the monitor dimmer real** — `src/Application/World/MonitorScreen.ts`.
    Switch the `dimmingPlane` from black+`AdditiveBlending` to `NormalBlending` so the
    angle/distance math in `update()` actually darkens the reflection stack at glancing
    angles; if it stays invisible, replace the approach (per §7.2).

[ ] **4. Add the desk-lamp toggle** — `NEW src/Application/World/DeskLamp.ts` +
    `World.ts`. Constraint: the whole scene is baked-lit (`BakedModel` → `MeshBasicMaterial`,
    zero dynamic lights), so the lamp's glow is baked into the texture as "on". Add a small
    emissive glow mesh + a soft light-spill gradient over the desk that fades in/out when the
    lamp is toggled, controlled via a keyboard key. Start from the baked "on" state; "off"
    dims the glow plane rather than re-baking. Verify final look in `npm run dev`.

[ ] **5. Fix the audio engine** — `src/Application/Audio/AudioManager.ts`,
    `AudioSources.ts`. Use a monotonic counter for `poolKey` (stop key collisions from
    the shrinking pool); floor the ambience filter at ~100 Hz and re-tune the 0.1 volume
    clamp; ramp `playAudio` gain over ~50 ms to kill clicks/pops; persist mute to
    `localStorage`. Add unit tests for the pool-key counter and the frequency floor.

[ ] **6. Add ambience ducking on monitor entry** — same files. `enterMonitor` dips +
    lowpasses the ambience; `leftMonitor` crossfades back (not a snap). Drive off the
    existing `enterMonitor`/`leftMonitor` camera events.

[ ] **7. Fix the film-grain + overlay** — `src/Application/Renderer.ts`,
    `src/Application/Shaders/screen/*`. Replace `u_time = Math.sin(time.current*0.01)`
    with a steadily increasing value; fix the invalid `zIndex = '1px'` → `'1'`; add a
    subtle vignette to the same fullscreen pass.

[ ] **8. Make the BIOS boot screen interactive** — `src/Application/UI/components/LoadingScreen.tsx`.
    Wire progress to real load events (progress = `loaded/toLoad`); add ESC to
    fast-forward the RAM check and DEL to open a fake BIOS setup panel; optionally reuse
    the `_AUTO_` typing sound for a boot-log typewriter effect. (Ohmxo branding already done.)

[ ] **9. Cleanup + cross-cutting** — remove dead `idle` self-assignment fallout, pause
    the `Time` rAF loop on `document.hidden`, add dynamic resolution scaling when frame
    time > 40 ms, type the `postMessage` contract in `MonitorScreen.ts`.

[ ] **10. Full validation** — `npm test`, `npm run build`, then `npm run dev` and
    manually click through: idle cover, monitor entry/exit, desk-lamp on/off, BIOS DEL/ESC,
    Ohmxo boot branding, mute persistence.

## Open questions

- **Lamp model** — the decor comes from baked `.glb` models with no named lamp mesh in
  source. Can you point out the lamp's approximate position ($x, y, z$) in the scene, or
  should I probe the baked model/UVs at runtime to locate it before placing the glow?
- **Scope lock** — should the §4 backlog features (day/night, second room) be sequenced
  after this plan, or held as separate follow-on plans?
