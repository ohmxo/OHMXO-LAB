# IMPLEMENTATION PLAN — Improving the Current Features

> Living companion to ^[`BRAINSTORM.md`](../BRAINSTORM.md) §7. This plan turns the
> feature-by-feature ideas there into ordered, independently shippable work. Each task
> leaves the app buildable (`npm run build`) and testable (`npm test`).
>
> **Priority logic:** correctness wins (§7.1) come first because they make the site
> cooler simply by working as designed. Then the BIOS easter eggs (§7.6).
> (Rebrand to Ohmxo is already complete.)

## Scope

- **In:** Fix the §7 bugs (⚠️) across camera, monitor, audio, renderer; the chosen
  visible upgrade (§7.6 BIOS easter eggs) + §7.4 audio fixes/ducking; per-file unit
  tests; Vitest already scaffolded.
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

[ ] **4. Fix the audio engine** — `src/Application/Audio/AudioManager.ts`,
    `AudioSources.ts`. Use a monotonic counter for `poolKey` (stop key collisions from
    the shrinking pool); floor the ambience filter at ~100 Hz and re-tune the 0.1 volume
    clamp; ramp `playAudio` gain over ~50 ms to kill clicks/pops; persist mute to
    `localStorage`. Add unit tests for the pool-key counter and the frequency floor.

[ ] **5. Add ambience ducking on monitor entry** — same files. `enterMonitor` dips +
    lowpasses the ambience; `leftMonitor` crossfades back (not a snap). Drive off the
    existing `enterMonitor`/`leftMonitor` camera events.

[ ] **6. Make the BIOS boot screen interactive** — `src/Application/UI/components/LoadingScreen.tsx`.
    Wire progress to real load events (progress = `loaded/toLoad`); add ESC to
    fast-forward the RAM check and DEL to open a fake BIOS setup panel; optionally reuse
    the `_AUTO_` typing sound for a boot-log typewriter effect. (Ohmxo branding already done.)

[ ] **7. Cleanup + cross-cutting** — remove dead `idle` self-assignment fallout; fix the
    invalid `zIndex = '1px'` → `'1'` in `Renderer.ts`; pause the `Time` rAF loop on
    `document.hidden`; add dynamic resolution scaling when frame time > 40 ms; type the
    `postMessage` contract in `MonitorScreen.ts`.

[ ] **8. Full validation** — `npm test`, `npm run build`, then `npm run dev` and
    manually click through: idle cover, monitor entry/exit, BIOS DEL/ESC, Ohmxo boot
    branding, mute persistence.

## Open questions

- **Scope lock** — should the §4 backlog features (day/night, second room) be sequenced
  after this plan, or held as separate follow-on plans?
