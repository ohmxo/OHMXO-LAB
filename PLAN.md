# Improve the 3D Portfolio — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make this fork of Henry Heffernan's 2022 portfolio *ours* and clean: rebrand all metadata, enable a working clickable-decor hitbox system, remove dead/debug code, make asset loading resilient to failures, and add bloom post-processing for visual punch.

**Architecture:** Keep the existing Bruno-Simon-style `Application` singleton and 4-layer renderer. Work in small, independently shippable phases. Add Vitest as a lightweight test runner so the hitbox math and loader logic are TDD'd. Every phase leaves the app buildable and runnable.

**Tech Stack:** Three.js, TypeScript, Webpack, React 17, Express; Vitest (new, dev-only) for tests; `three/examples/jsm` EffectComposer + UnrealBloomPass.

**Build/run commands:**
- Dev: `npm run dev` (webpack-dev-server)
- Build: `npm run build`
- Serve: `npm start`
- Test: `npx vitest run`

---

## Phase 0 — Test infrastructure

The project has **no test runner**. Before TDD'ing hitbox/loader logic, add Vitest. It's the lightest fit for a webpack+TS project and needs no Babel config changes.

### Task 1: Add Vitest + a smoke test

**Files:**
- Modify: `package.json`
- Create: `vitest.config.ts`
- Create: `src/Application/Utils/__tests__/smoke.test.ts`

- [ ] **Step 1: Add Vitest as a dev dependency**

```bash
npm i -D vitest@^0.34.0
```

- [ ] **Step 2: Create `vitest.config.ts`**

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
    test: {
        environment: 'node',
        include: ['src/**/__tests__/**/*.test.ts'],
    },
});
```

- [ ] **Step 3: Create `src/Application/Utils/__tests__/smoke.test.ts`**

```typescript
import { describe, it, expect } from 'vitest';

describe('smoke', () => {
    it('1 + 1 equals 2', () => {
        expect(1 + 1).toBe(2);
    });
});
```

- [ ] **Step 4: Add a `test` script to `package.json`**

In `package.json`, add to `scripts`:
```json
"test": "vitest run",
"test:watch": "vitest"
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `npm test`
Expected: PASS — `1 + 1 equals 2`

- [ ] **Step 6: Commit**

```bash
git add package.json vitest.config.ts src/Application/Utils/__tests__/smoke.test.ts
git commit -m "chore: add vitest test runner"
```

---

## Phase 1 — Rebrand: make it ours

Every visible Henry reference is replaced with your brand. **You must supply your values** — the plan uses placeholder tokens (`YOUR_NAME`, `YOUR_TITLE`, `YOUR_DOMAIN`, `YOUR_GA_ID`). Pick them before starting and substitute everywhere.

### Task 2: Replace all metadata + remove Henry's analytics

**Files:**
- Modify: `src/index.html`
- Modify: `readme.md`
- Modify: `package.json`

- [ ] **Step 1: Rewrite `src/index.html` `<head>` metadata**

Replace the `<title>` and all meta tags (title, description, og:, twitter:) with your own. Keep the `<link rel="shortcut icon">` line but point it at your own favicon if you have one. Example:
```html
<title>YOUR_NAME - YOUR_TITLE</title>
<meta name="description" content="YOUR_NAME — a software engineer who builds interactive 3D web experiences.">
```

- [ ] **Step 2: Remove Henry's Google Analytics block**

In `src/index.html`, delete the two lines from `<script async src="https://www.googletagmanager.com/gtag/js?id=G-4FJBF6WF60"></script>` through the closing `</script>` that calls `gtag('config', 'G-4FJBF6WF60')`. If you have your own GA, replace `G-4FJBF6WF60` with your ID; otherwise remove the whole block.

- [ ] **Step 3: Rewrite `readme.md`**

Replace the whole file with:
```markdown
# YOUR_NAME — Portfolio

An interactive 3D portfolio built with Three.js. Explore a stylized desk scene and
click the computer to enter a live inner site.

## Setup

```bash
npm i
npm run dev        # webpack-dev-server
```

## Production

```bash
npm run build      # webpack production build
npm start          # serve with express
```

## Tests

```bash
npm test
```
```

- [ ] **Step 4: Fix `package.json` repo + license**

Replace:
```json
"repository": "#",
"license": "UNLICENSED",
```
with your real repo URL and license (the repo already contains a `LICENSE.md`):
```json
"repository": "https://github.com/YOUR_USERNAME/YOUR_REPO",
"license": "MIT"
```

- [ ] **Step 5: Commit**

```bash
git add src/index.html readme.md package.json
git commit -m "chore: rebrand metadata, remove upstream analytics"
```

---

## Phase 2 — Fix the hitbox system

This is the copy-paste bug + the disabled-but-scaffolded interactive decor system. We fix the bug, wire it into `World`, add TDD'd raycast hit detection, and make the coffee cup + plant clickable → they trigger a camera focus.

### Task 3: Fix the `Hitboxes` class bug

**Files:**
- Modify: `src/Application/World/Hitboxes.ts`

- [ ] **Step 1: Write a failing test for the class name/export**

Create `src/Application/World/__tests__/hitboxes.test.ts`:
```typescript
import { describe, it, expect } from 'vitest';
import Hitboxes from '../Hitboxes';

describe('Hitboxes', () => {
    it('exports a class named Hitboxes with a getHitbox method', () => {
        expect(Hitboxes).toBeTypeOf('function');
        const instance = new Hitboxes();
        expect(instance.getHitbox('coffee')).toBeDefined();
    });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx vitest run src/Application/World/__tests__/hitboxes.test.ts`
Expected: FAIL — `getHitbox` is not a function (or class name mismatch).

- [ ] **Step 3: Rewrite `Hitboxes.ts`**

Replace the entire file. The bug is that line 8 declares `export default class Decor` — it must be `Hitboxes`. We also strip the `console.log` debug spam and expose a clean API:
```typescript
import * as THREE from 'three';
import Application from '../Application';
import Camera from '../Camera/Camera';
import Mouse from '../Utils/Mouse';

export interface HitboxAction {
    name: string;
    action: () => void;
}

export default class Hitboxes {
    application: Application;
    scene: THREE.Scene;
    camera: Camera;
    mouse: Mouse;
    raycaster: THREE.Raycaster;
    hitboxes: { [name: string]: HitboxAction };

    constructor() {
        this.application = new Application();
        this.scene = this.application.scene;
        this.camera = this.application.camera;
        this.mouse = this.application.mouse;
        this.raycaster = new THREE.Raycaster();
        this.hitboxes = {};
    }

    getHitbox(name: string): HitboxAction | undefined {
        return this.hitboxes[name];
    }

    createHitbox(
        name: string,
        action: () => void,
        position: THREE.Vector3,
        size: THREE.Vector3
    ) {
        const hitboxMaterial = new THREE.MeshBasicMaterial({
            color: 0xff0000,
            side: THREE.DoubleSide,
            transparent: true,
            opacity: 0,
            depthWrite: false,
        });

        const hitbox = new THREE.Mesh(
            new THREE.BoxBufferGeometry(size.x, size.y, size.z),
            hitboxMaterial
        );
        hitbox.name = name;
        hitbox.position.copy(position);
        this.scene.add(hitbox);

        this.hitboxes[name] = { name, action };
        return hitbox;
    }

    checkIntersections(clientX: number, clientY: number): HitboxAction | null {
        this.raycaster.setFromCamera(
            new THREE.Vector2(clientX, clientY),
            this.camera.instance
        );
        const meshes = Object.keys(this.hitboxes).map(
            (key) => this.scene.getObjectByName(key) as THREE.Object3D
        );
        const intersects = this.raycaster.intersectObjects(meshes);
        if (intersects.length > 0) {
            const name = intersects[0].object.name;
            return this.hitboxes[name] || null;
        }
        return null;
    }
}
```

> Note: the mouse coordinates used by `setFromCamera` are normalized device coords in `[-1, 1]`. The `Mouse` util stores raw pixels; see Task 4 where we normalize before calling `checkIntersections`.

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run src/Application/World/__tests__/hitboxes.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/Application/World/Hitboxes.ts src/Application/World/__tests__/hitboxes.test.ts
git commit -m "fix: correct Hitboxes class name, strip debug logs, expose clean API"
```

### Task 4: Wire hitboxes into World and make decor clickable

**Files:**
- Modify: `src/Application/World/World.ts`
- Modify: `src/Application/Utils/Mouse.ts`
- Modify: `src/Application/Camera/Camera.ts`

- [ ] **Step 1: Add normalized mouse coordinates to `Mouse.ts`**

The raycast needs NDC (`-1..1`), but `Mouse` stores pixels. Add a derived `normalized` vector. Modify `Mouse.ts`:
```typescript
import EventEmitter from './EventEmitter';
import Application from '../Application';
import Sizes from './Sizes';

export default class Mouse extends EventEmitter {
    x: number;
    y: number;
    inComputer: boolean;
    application: Application;
    sizes: Sizes;
    ndc: { x: number; y: number };

    constructor() {
        super();
        this.x = 0;
        this.y = 0;
        this.inComputer = false;
        this.application = new Application();
        this.sizes = this.application.sizes;
        this.ndc = { x: 0, y: 0 };

        this.on('mousemove', (event: any) => {
            if (event.clientX && event.clientY) {
                this.x = event.clientX;
                this.y = event.clientY;
                this.ndc.x = (event.clientX / this.sizes.width) * 2 - 1;
                this.ndc.y = -(event.clientY / this.sizes.height) * 2 + 1;
            }
            this.inComputer = event.inComputer ? true : false;
        });
    }
}
```

- [ ] **Step 2: Add a `focusDecor` transition to `Camera.ts`**

Add a new camera key and a method. First add to the `CameraKey` enum:
```typescript
export enum CameraKey {
    IDLE = 'idle',
    MONITOR = 'monitor',
    LOADING = 'loading',
    DESK = 'desk',
    ORBIT_CONTROLS_START = 'orbitControlsStart',
    DECOR = 'decor',
}
```
Then add the keyframe to the `keyframes` map in the constructor and a transition helper:
```typescript
import { ..., DecorKeyframe } from './CameraKeyframes';
// inside constructor keyframes map:
decor: new DecorKeyframe(),
```
And a method:
```typescript
focusDecor(position: THREE.Vector3, focalPoint: THREE.Vector3) {
    this.transition(CameraKey.DECOR, 1200, BezierEasing(0.13, 0.99, 0, 1));
}
```

- [ ] **Step 3: Add a `DecorKeyframe` to `CameraKeyframes.ts`**

Append:
```typescript
export class DecorKeyframe extends CameraKeyframeInstance {
    position: THREE.Vector3;
    focalPoint: THREE.Vector3;
    constructor() {
        const pos = new THREE.Vector3(1500, 1200, 2400);
        const foc = new THREE.Vector3(1670, 200, 900); // the coffee cup
        super({ position: pos, focalPoint: foc });
        this.position = pos;
        this.focalPoint = foc;
    }
    update() {}
}
```
> These are placeholder coordinates — tune to your actual scene after wiring in.

- [ ] **Step 4: Wire hitboxes + click handling in `World.ts`**

Modify `World.ts`. Add `Hitboxes` to imports, instantiate in the `'ready'` handler, create a coffee hitbox, and add a click listener that calls `checkIntersections` using `mouse.ndc`:
```typescript
import Hitboxes from './Hitboxes';
// ...
export default class World {
    // ...
    hitboxes: Hitboxes;
    // in the 'ready' callback, after other setup:
    this.hitboxes = new Hitboxes();
    this.hitboxes.createHitbox(
        'coffee',
        () => this.application.camera.focusDecor(
            new THREE.Vector3(1500, 1200, 2400),
            new THREE.Vector3(1670, 200, 900)
        ),
        new THREE.Vector3(1670, 200, 900),
        new THREE.Vector3(300, 300, 300)
    );

    document.addEventListener('mousedown', (event: any) => {
        if (event.inComputer) return; // don't hijack computer clicks
        this.hitboxes.checkIntersections(this.application.mouse.ndc.x, this.application.mouse.ndc.y);
    });
}
```
> `checkIntersections` returns the action; call `hit.action()` on it. This plan wires the plumbing; finalize the handler in Task 5 after the camera focus returns correctly.

- [ ] **Step 5: Build to verify no compile errors**

Run: `npm run build`
Expected: build succeeds (webpack emits `dist/`).

- [ ] **Step 6: Commit**

```bash
git add src/Application/World/World.ts src/Application/Utils/Mouse.ts src/Application/Camera/Camera.ts src/Application/Camera/CameraKeyframes.ts
git commit -m "feat: wire clickable decor hitboxes into world, add decor camera focus"
```

### Task 5: Return-to-desk after decor focus

**Files:**
- Modify: `src/Application/World/World.ts`

- [ ] **Step 1: Make the click handler complete the action**

Update the `mousedown` handler so it actually calls the hitbox action and then returns to desk on a subsequent click:
```typescript
this.hitboxes.checkIntersections(this.application.mouse.ndc.x, this.application.mouse.ndc.y)
    ?.action?.();
```
To return to desk, add a listener: when `currentKeyframe === DECOR` and the user clicks anywhere not on a hitbox, transition back to `DESK`. Implement in `Camera.ts` `setMonitorListeners`-style, or simplest: in the same `mousedown` handler:
```typescript
const hit = this.hitboxes.checkIntersections(this.application.mouse.ndc.x, this.application.mouse.ndc.y);
if (hit) {
    hit.action();
} else if (this.application.camera.currentKeyframe === CameraKey.DECOR) {
    this.application.camera.transition(CameraKey.DESK, 1200, BezierEasing(0.13, 0.99, 0, 1));
}
```

- [ ] **Step 2: Test manually**

Run: `npm run dev`, open the page, wait for load, click the coffee cup → camera focuses on it; click empty space → camera returns to desk.

- [ ] **Step 3: Commit**

```bash
git add src/Application/World/World.ts
git commit -m "feat: return to desk after decor focus"
```

---

## Phase 3 — Clean dead code + debug spam

### Task 6: Remove dead files, unused code, and console.logs

**Files:**
- Delete: `src/Application/World/Cursor.ts` (unused)
- Modify: `src/Application/World/Decor.ts` (remove dead raycaster code)
- Modify: `src/Application/Application.ts` (uncomment/remove `window.Application`)

- [ ] **Step 1: Delete `Cursor.ts`**

```bash
git rm src/Application/World/Cursor.ts
```
Verify nothing imports it: `grep -rn "Cursor" src/`. Remove any remaining imports.

- [ ] **Step 2: Clean `Decor.ts`**

`Decor.ts` currently duplicates the baked-model pattern and had raycast stubs. If it's a pure baked decor, keep it and remove any commented-out/duplicate code. If `Hitboxes` (Task 3) now owns interaction, strip raycast code from `Decor.ts` so it only bakes + adds the model. Verify it still compiles with `npm run build`.

- [ ] **Step 3: Remove debug comments in `Application.ts`**

In `Application.ts`, either uncomment or delete the `// window.Application = this;` block and the `//@ts-ignore` above it. Decide: if you want global debug access, uncomment it; otherwise remove. Keep it removed for the final codebase, or gate it behind `this.debug.active`.

- [ ] **Step 4: Sweep remaining `console.log` / `console.error` in `src/`**

Run: `grep -rn "console\.\(log\|debug\)" src/`
Expected: no hits in production code (remove any; keep `console.error` only if used for real failure reporting).

- [ ] **Step 5: Build + commit**

Run: `npm run build`
Then:
```bash
git add -A
git commit -m "chore: remove dead cursor file and debug console logs"
```

---

## Phase 4 — Resilient resource loading

### Task 7: Add loader error handling so a failed asset doesn't hang the app

**Files:**
- Modify: `src/Application/Utils/Resources.ts`

- [ ] **Step 1: Write a failing test for error counting**

Create `src/Application/Utils/__tests__/resources.test.ts`:
```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';

// We test the pure "ready when loaded === toLoad" logic that Resources uses.
// Resources imports THREE + the Application singleton (which needs a DOM),
// so we test a small pure mirror instead of the class itself.
import { isLoadComplete } from '../resourcesComplete';

describe('resource load completion', () => {
    it('is not complete until loaded equals toLoad', () => {
        expect(isLoadComplete(3, 3)).toBe(true);
        expect(isLoadComplete(2, 3)).toBe(false);
    });
    it('accounts for failures without blocking completion', () => {
        expect(isLoadComplete(2, 3, 1)).toBe(true); // 2 loaded + 1 failed = 3 resolved
    });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npx vitest run src/Application/Utils/__tests__/resources.test.ts`
Expected: FAIL — `resourcesComplete` module not found.

- [ ] **Step 3: Create `src/Application/Utils/resourcesComplete.ts`**

```typescript
export function isLoadComplete(
    loaded: number,
    toLoad: number,
    failed: number = 0
): boolean {
    return loaded + failed >= toLoad;
}
```

- [ ] **Step 4: Wire errors into `Resources.ts`**

In `Resources.ts` `startLoading()`, add an `onError` callback to every loader call. Each loader signature:
- `gltfLoader.load(url, onLoad, onProgress, onError)`
- `textureLoader.load(url, onLoad, onProgress, onError)`
- `audioLoader.load(url, onLoad, onProgress, onError)`

Add a `failed` counter and an `onError` that increments it and calls a new `sourceFailed` method:
```typescript
sourceFailed(source: Resource, error: unknown) {
    this.failed++;
    console.error(`Failed to load ${source.name}:`, error);
    this.loading.trigger('loadedSource', [source.name, this.loaded, this.toLoad]);
    if (isLoadComplete(this.loaded, this.toLoad, this.failed)) {
        this.trigger('ready');
    }
}
```
Add `this.failed = 0` in the constructor, import `isLoadComplete`, and pass `onError: (e) => this.sourceFailed(source, e)` to each `load` call.

- [ ] **Step 5: Run tests + build**

Run: `npm test` and `npm run build`. Both must pass.

- [ ] **Step 6: Commit**

```bash
git add src/Application/Utils/Resources.ts src/Application/Utils/resourcesComplete.ts src/Application/Utils/__tests__/resources.test.ts
git commit -m "feat: add resource load failure handling, prevent hang on error"
```

---

## Phase 5 — Bloom post-processing

### Task 8: Add UnrealBloom so the monitor glows into the dark room

**Files:**
- Modify: `src/Application/Renderer.ts`
- Modify: `src/Application/Renderer.ts` (add composer)

- [ ] **Step 1: Import EffectComposer + passes**

In `Renderer.ts` add imports:
```typescript
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js';
```

- [ ] **Step 2: Create the composer in `setInstance()`**

After `this.instance` is set up, add:
```typescript
this.composer = new EffectComposer(this.instance);
this.composer.addPass(new RenderPass(this.scene, this.camera.instance));
const bloomPass = new UnrealBloomPass(
    new THREE.Vector2(this.sizes.width, this.sizes.height),
    1.5,   // strength
    0.4,   // radius
    0.85   // threshold
);
this.composer.addPass(bloomPass);
```

- [ ] **Step 3: Replace the main render call in `update()`**

Change the main scene render from:
```typescript
this.instance.render(this.scene, this.camera.instance);
```
to:
```typescript
this.composer.render();
```
Keep the CSS3D and overlay renders as-is.

- [ ] **Step 4: Resize the composer**

In `resize()`, add:
```typescript
this.composer.setSize(this.sizes.width, this.sizes.height);
```

- [ ] **Step 5: Add a `composer: EffectComposer` field**

Declare the field on the `Renderer` class: `composer: EffectComposer;` and assign it in `setInstance()`.

- [ ] **Step 6: Build + verify visually**

Run: `npm run dev`. The monitor and bright areas should bloom softly into the dark room. Tune `strength`/`threshold` in Step 2 if too strong/weak.

- [ ] **Step 7: Commit**

```bash
git add src/Application/Renderer.ts
git commit -m "feat: add UnrealBloom post-processing for monitor glow"
```

---

## Phase 6 — Security & metadata hardening

### Task 9: Serve a Content Security Policy + replace GA

**Files:**
- Modify: `server/index.ts`

- [ ] **Step 1: Read `server/index.ts`**

Open `server/index.ts` to see how the Express server sets headers. Add a middleware that sets a CSP allowing self + the iframe's inner site.

- [ ] **Step 2: Add a CSP middleware**

Add before routes:
```typescript
app.use((_req: any, res: any, next: any) => {
    res.setHeader(
        'Content-Security-Policy',
        "default-src 'self'; script-src 'self' 'unsafe-inline' https://www.googletagmanager.com; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self'; frame-src 'self' https://os.YOUR_DOMAIN.com; object-src 'none'; base-uri 'self';"
    );
    res.setHeader('X-Content-Type-Options', 'nosniff');
    res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
    next();
});
```
> Adjust `frame-src` to your inner site's origin. If you removed GA in Task 2, drop the `https://www.googletagmanager.com` entry and the GA inline script.

- [ ] **Step 3: Verify `frame-src` matches the iframe**

In `MonitorScreen.ts`, the iframe loads `https://os.henryheffernan.com/` (or your own). Ensure `frame-src` includes that exact origin or the iframe will be blocked.

- [ ] **Step 4: Build + commit**

Run: `npm run build`
Then:
```bash
git add server/index.ts
git commit -m "feat: serve CSP and security headers"
```

---

## Self-review

- **Spec coverage (BRAINSTORM §3, §5):** Confirmed issues §3 are all mapped — hitbox bug (Task 3), disabled hitboxes (Task 4), console.logs (Task 3, 6), Cursor unused (Task 6), zIndex (not in plan — add Task 6 note), metadata/GA (Task 2, 9), no CSP (Task 9), Resources error handling (Task 7). Creative features §4: hitboxes (#2) done; bloom (#9) done; audio-reactive, day/night, second room, scroll narrative are larger — **out of scope** for this plan (separate plans).
- **Placeholder scan:** The `YOUR_*` tokens are intentional user-supplied values, called out at the top of Phase 1. Coordinate values in `DecorKeyframe`/hitbox are placeholders explicitly flagged "tune to your scene." No TBD/TODO.
- **Type consistency:** `Hitboxes.getHitbox` and `checkIntersections` are defined in Task 3 and used in Task 4. `isLoadComplete` defined in Task 7 Step 3, used in Step 4. `focusDecor` defined in Task 4 Step 2, used in Step 4/5. `CameraKey.DECOR` added in Task 4 Step 2, referenced in Task 5. Consistent.

**Not covered (future plans):** audio-reactive shaders, day/night, second room, scroll storytelling, own inner site (that's a whole separate subsystem — recommended as its own plan after this one lands).
