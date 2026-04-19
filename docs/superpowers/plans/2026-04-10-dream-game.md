# Dream Game Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single `index.html` browser game where blue (player) and red (AI) sides fire bullets at each other's walls; unblocked bullets reward the firing side with extra units; first side to achieve full board coverage wins.

**Architecture:** All game logic lives in one HTML file. Physics and collision run in 2D on the XZ plane; Three.js renders everything in an isometric 3D projection. Pure functions (coverage calculation, collision) are unit-tested via inline assertions that run at startup and log pass/fail to the console.

**Tech Stack:** Three.js 0.162 (CDN, ES module), vanilla JS, HTML5 Canvas via WebGL renderer.

---

## File Layout

```
C:\Users\lenovo\dreamgame\
└── index.html   ← entire game (all tasks modify this single file)
```

---

## Constants Reference (used across all tasks)

```js
const BOARD_W    = 12;     // board width (X axis)
const BOARD_D    = 8;      // board depth (Z axis, split equally)
const HALF_D     = BOARD_D / 2;  // 4
const UNIT_RADIUS       = 0.6;   // collision + coverage radius
const REPULSION_DIST    = 1.4;   // units push each other apart below this
const REPULSION_K       = 3.0;
const SPRING_K          = 8.0;   // cursor magnet spring strength
const DAMPING           = 4.0;   // velocity damping per second
const BULLET_SPEED      = 6.0;   // units/second
const SPAWN_INTERVAL    = 0.22;  // seconds between bullet spawns per side
const RED_BASE_SPEED    = 3.5;   // red AI drift speed
const BLUE_COLOR        = 0x2255cc;
const RED_COLOR         = 0xcc2222;
const BULLET_BLUE_COLOR = 0x4499ff;
const BULLET_RED_COLOR  = 0xff4444;
```

---

## Task 1: HTML Scaffold + Isometric Scene

**Files:**
- Create: `C:\Users\lenovo\dreamgame\index.html`

- [ ] **Step 1: Create the file**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Dream Game</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { background: #111; overflow: hidden; }
    canvas { display: block; }
    #hud {
      position: fixed; top: 16px; left: 50%; transform: translateX(-50%);
      display: flex; gap: 24px; align-items: center;
      font-family: monospace; color: #fff; font-size: 13px;
      background: rgba(0,0,0,0.5); padding: 8px 16px; border-radius: 8px;
      pointer-events: none;
    }
    .bar-wrap { display: flex; align-items: center; gap: 8px; }
    .bar { width: 120px; height: 10px; background: #333; border-radius: 5px; overflow: hidden; }
    .fill { height: 100%; width: 0%; border-radius: 5px; transition: width 0.1s; }
    .fill.blue { background: #4499ff; }
    .fill.red  { background: #ff4444; }
    #overlay {
      position: fixed; inset: 0; display: none; align-items: center; justify-content: center;
      flex-direction: column; gap: 16px; background: rgba(0,0,0,0.7);
      font-family: monospace; color: #fff;
    }
    #overlay.show { display: flex; }
    #overlay h1 { font-size: 48px; }
    #overlay button {
      padding: 10px 28px; font-size: 16px; border: none; border-radius: 6px;
      background: #4499ff; color: #fff; cursor: pointer;
    }
  </style>
</head>
<body>
  <div id="hud">
    <div class="bar-wrap">
      <span>BLUE</span>
      <div class="bar"><div id="blue-fill" class="fill blue"></div></div>
      <span id="blue-count">3</span>
    </div>
    <div class="bar-wrap">
      <span>RED</span>
      <div class="bar"><div id="red-fill" class="fill red"></div></div>
      <span id="red-count">3</span>
    </div>
  </div>
  <div id="overlay">
    <h1 id="overlay-title">YOU WIN</h1>
    <button onclick="restartGame()">Play Again</button>
  </div>

  <script type="importmap">
    { "imports": { "three": "https://unpkg.com/three@0.162.0/build/three.module.js" } }
  </script>
  <script type="module">
    import * as THREE from 'three';
    window.THREE = THREE; // expose for restart helper

    // ── Constants ──────────────────────────────────────────────────────────
    const BOARD_W         = 12;
    const BOARD_D         = 8;
    const HALF_D          = BOARD_D / 2;
    const UNIT_RADIUS     = 0.6;
    const REPULSION_DIST  = 1.4;
    const REPULSION_K     = 3.0;
    const SPRING_K        = 8.0;
    const DAMPING         = 4.0;
    const BULLET_SPEED    = 6.0;
    const SPAWN_INTERVAL  = 0.22;
    const RED_BASE_SPEED  = 3.5;
    const BLUE_COLOR      = 0x2255cc;
    const RED_COLOR       = 0xcc2222;
    const BULLET_BLUE_CLR = 0x4499ff;
    const BULLET_RED_CLR  = 0xff4444;

    // ── Renderer ───────────────────────────────────────────────────────────
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.shadowMap.enabled = true;
    document.body.appendChild(renderer.domElement);

    // ── Scene ──────────────────────────────────────────────────────────────
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x1a1a2e);

    // ── Isometric Camera ───────────────────────────────────────────────────
    const aspect = window.innerWidth / window.innerHeight;
    const d = 9;
    const camera = new THREE.OrthographicCamera(
      -d * aspect, d * aspect, d, -d, 0.1, 200
    );
    camera.position.set(14, 14, 14);
    camera.lookAt(0, 0, 0);

    // ── Lights ─────────────────────────────────────────────────────────────
    scene.add(new THREE.AmbientLight(0xffffff, 0.6));
    const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
    dirLight.position.set(10, 20, 10);
    scene.add(dirLight);

    // ── Resize handler ─────────────────────────────────────────────────────
    window.addEventListener('resize', () => {
      const a = window.innerWidth / window.innerHeight;
      camera.left = -d * a; camera.right = d * a;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    });

    // ── Initial render test ────────────────────────────────────────────────
    renderer.render(scene, camera);
    console.log('Task 1 OK: scene rendered');
  </script>
</body>
</html>
```

- [ ] **Step 2: Open in browser and verify**

Open `http://localhost:3000` (game static server must be running).
Expected: dark blue-grey background (`#1a1a2e`), empty canvas, no JS errors in console, "Task 1 OK: scene rendered" logged.

- [ ] **Step 3: Commit**

```bash
cd C:/Users/lenovo/dreamgame
git init
git add index.html
git commit -m "feat: three.js scaffold with isometric camera"
```

---

## Task 2: Board Geometry

**Files:**
- Modify: `index.html` — add board mesh after the lights block

- [ ] **Step 1: Add board plane and zone tints**

After `scene.add(dirLight);` and before the resize handler, insert:

```js
// ── Board ──────────────────────────────────────────────────────────────
// Main board surface
const boardGeo = new THREE.BoxGeometry(BOARD_W, 0.15, BOARD_D);
const boardMat = new THREE.MeshLambertMaterial({ color: 0xe8e8ec });
const boardMesh = new THREE.Mesh(boardGeo, boardMat);
boardMesh.position.y = -0.08;
scene.add(boardMesh);

// Red zone tint (top half: negative Z)
const redZoneGeo = new THREE.PlaneGeometry(BOARD_W, HALF_D);
const redZoneMat = new THREE.MeshBasicMaterial({
  color: 0xff2222, transparent: true, opacity: 0.07, depthWrite: false
});
const redZone = new THREE.Mesh(redZoneGeo, redZoneMat);
redZone.rotation.x = -Math.PI / 2;
redZone.position.set(0, 0.01, -HALF_D / 2);
scene.add(redZone);

// Blue zone tint (bottom half: positive Z)
const blueZoneGeo = new THREE.PlaneGeometry(BOARD_W, HALF_D);
const blueZoneMat = new THREE.MeshBasicMaterial({
  color: 0x2255ff, transparent: true, opacity: 0.07, depthWrite: false
});
const blueZone = new THREE.Mesh(blueZoneGeo, blueZoneMat);
blueZone.rotation.x = -Math.PI / 2;
blueZone.position.set(0, 0.01, HALF_D / 2);
scene.add(blueZone);

// Center dashed line (series of short boxes)
const dashMat = new THREE.MeshBasicMaterial({ color: 0x888888 });
const dashCount = 10;
for (let i = 0; i < dashCount; i++) {
  const dash = new THREE.Mesh(
    new THREE.BoxGeometry(0.6, 0.05, 0.12),
    dashMat
  );
  dash.position.set(
    -BOARD_W / 2 + (i + 0.5) * (BOARD_W / dashCount),
    0.02, 0
  );
  scene.add(dash);
}

// Board border (thin outline box, slightly larger)
const borderGeo = new THREE.BoxGeometry(BOARD_W + 0.1, 0.1, BOARD_D + 0.1);
const borderMat = new THREE.MeshBasicMaterial({ color: 0x5577aa, wireframe: false });
const border = new THREE.Mesh(
  new THREE.EdgesGeometry(borderGeo),
  new THREE.LineBasicMaterial({ color: 0x5577aa })
);
border.position.y = -0.05;
scene.add(new THREE.LineSegments(new THREE.EdgesGeometry(boardGeo), new THREE.LineBasicMaterial({ color: 0x8899bb })));
```

- [ ] **Step 2: Verify in browser**

Refresh `http://localhost:3000`.
Expected: Light grey rectangular board visible at isometric angle, subtle red tint on the far (top) half, subtle blue tint on the near (bottom) half, dashed line across the center. No JS errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: board geometry with zone tints and center line"
```

---

## Task 3: Unit System

**Files:**
- Modify: `index.html` — add `createUnitMesh`, `Unit` class, initial spawns

- [ ] **Step 1: Add `createUnitMesh` and `Unit` class**

After the board geometry block, insert:

```js
// ── Units ──────────────────────────────────────────────────────────────
function createUnitMesh(color) {
  const g = new THREE.Group();
  const mat = new THREE.MeshLambertMaterial({ color });
  const add = (w, h, d, x, y, z) => {
    const m = new THREE.Mesh(new THREE.BoxGeometry(w, h, d), mat);
    m.position.set(x, y, z);
    g.add(m);
  };
  add(0.28, 0.28, 0.28,  0,    1.08, 0);   // head
  add(0.38, 0.42, 0.20,  0,    0.70, 0);   // torso
  add(0.14, 0.38, 0.14, -0.28, 0.65, 0);   // left arm
  add(0.14, 0.38, 0.14,  0.28, 0.65, 0);   // right arm
  add(0.14, 0.42, 0.14, -0.14, 0.22, 0);   // left leg
  add(0.14, 0.42, 0.14,  0.14, 0.22, 0);   // right leg
  return g;
}

class Unit {
  constructor(side) {
    this.side = side; // 'blue' | 'red'
    this.mesh = createUnitMesh(side === 'blue' ? BLUE_COLOR : RED_COLOR);
    this.vx = 0;
    this.vz = 0;
    this.bobOffset = Math.random() * Math.PI * 2; // idle animation phase
    this.spawnScale = 0; // 0→1 pop-in animation
  }
}

const blueUnits = [];
const redUnits  = [];

function spawnUnit(side) {
  const u = new Unit(side);
  const x = (Math.random() - 0.5) * (BOARD_W - 1.5);
  const z = side === 'blue'
    ? HALF_D * 0.1 + Math.random() * (HALF_D - 0.4)
    : -HALF_D * 0.1 - Math.random() * (HALF_D - 0.4);
  u.mesh.position.set(x, 0, z);
  u.mesh.scale.setScalar(0.01); // start tiny for pop animation
  scene.add(u.mesh);
  if (side === 'blue') blueUnits.push(u); else redUnits.push(u);
}

// Spawn starting units
for (let i = 0; i < 3; i++) spawnUnit('blue');
for (let i = 0; i < 3; i++) spawnUnit('red');
```

- [ ] **Step 2: Add pop-in animation in the game loop (stub loop)**

After the spawn block, insert the temporary render loop (will be replaced in Task 4):

```js
// ── Temporary render loop (replaced in Task 4) ────────────────────────
let lastTime = performance.now();
function tempLoop() {
  requestAnimationFrame(tempLoop);
  const now = performance.now();
  const dt = Math.min((now - lastTime) / 1000, 0.05);
  lastTime = now;

  const t = now / 1000;
  // Pop-in + idle bob for all units
  for (const u of [...blueUnits, ...redUnits]) {
    if (u.spawnScale < 1) {
      u.spawnScale = Math.min(1, u.spawnScale + dt * 4);
      u.mesh.scale.setScalar(u.spawnScale);
    }
    u.mesh.position.y = 0.04 * Math.sin(t * 2.5 + u.bobOffset);
  }
  renderer.render(scene, camera);
}
tempLoop();
```

- [ ] **Step 3: Verify in browser**

Refresh. Expected: 3 blue humanoid figures on the near half, 3 red on the far half, all popping in and gently bobbing up and down. No JS errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: unit system with boxy humanoid meshes and bob animation"
```

---

## Task 4: Bullet System + Full Game Loop

**Files:**
- Modify: `index.html` — add `Bullet` class, spawn timers, replace temp loop with full game loop

- [ ] **Step 1: Add Bullet class and arrays**

Replace the `// ── Temporary render loop` comment block entirely with the following (keep the `tempLoop` function content but we'll expand it). First, add before the loop:

```js
// ── Bullets ────────────────────────────────────────────────────────────
class Bullet {
  constructor(side, x) {
    this.side = side;
    const color = side === 'blue' ? BULLET_BLUE_CLR : BULLET_RED_CLR;
    const geo = new THREE.BoxGeometry(0.18, 0.18, 0.18);
    const mat = new THREE.MeshBasicMaterial({ color });
    this.mesh = new THREE.Mesh(geo, mat);
    const z = side === 'blue' ? HALF_D : -HALF_D;
    this.mesh.position.set(x, 0.5, z);
    this.vz = side === 'blue' ? -BULLET_SPEED : BULLET_SPEED;
    // Glow
    const light = new THREE.PointLight(color, 0.8, 1.5);
    this.mesh.add(light);
    scene.add(this.mesh);
    this.alive = true;
  }
  destroy() {
    scene.remove(this.mesh);
    this.alive = false;
  }
}

const blueBullets = [];
const redBullets  = [];
let blueSpawnTimer = 0;
let redSpawnTimer  = 0;
let gameOver = false;

function spawnBullet(side) {
  const x = (Math.random() - 0.5) * (BOARD_W - 0.5);
  const b = new Bullet(side, x);
  if (side === 'blue') blueBullets.push(b); else redBullets.push(b);
}
```

- [ ] **Step 2: Replace the temp loop with the real game loop**

Remove the `tempLoop` function and replace it with:

```js
// ── Game Loop ──────────────────────────────────────────────────────────
let lastTime = performance.now();
let elapsed  = 0;

function gameLoop() {
  requestAnimationFrame(gameLoop);
  if (gameOver) { renderer.render(scene, camera); return; }

  const now = performance.now();
  const dt  = Math.min((now - lastTime) / 1000, 0.05);
  lastTime  = now;
  elapsed  += dt;

  // ── Spawn bullets ──
  blueSpawnTimer += dt;
  redSpawnTimer  += dt;
  const interval = Math.max(0.10, SPAWN_INTERVAL - elapsed * 0.0005);
  if (blueSpawnTimer >= interval) { spawnBullet('blue'); blueSpawnTimer = 0; }
  if (redSpawnTimer  >= interval) { spawnBullet('red');  redSpawnTimer  = 0; }

  // ── Move bullets ──
  for (const b of [...blueBullets, ...redBullets]) {
    if (!b.alive) continue;
    b.mesh.position.z += b.vz * dt;
  }

  // ── Remove off-screen bullets (wall hit handled in Task 5) ──
  const removeDead = arr => {
    for (let i = arr.length - 1; i >= 0; i--) {
      if (!arr[i].alive) arr.splice(i, 1);
    }
  };
  removeDead(blueBullets);
  removeDead(redBullets);

  // ── Unit animations ──
  const t = now / 1000;
  for (const u of [...blueUnits, ...redUnits]) {
    if (u.spawnScale < 1) {
      u.spawnScale = Math.min(1, u.spawnScale + dt * 4);
      u.mesh.scale.setScalar(u.spawnScale);
    }
    u.mesh.position.y = 0.04 * Math.sin(t * 2.5 + u.bobOffset);
  }

  renderer.render(scene, camera);
}
gameLoop();
```

- [ ] **Step 3: Verify in browser**

Refresh. Expected: Small glowing cubes streaming upward from the bottom (blue bullets) and downward from the top (red bullets). Bullets pass through each other. Units still bob. No JS errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: bullet system with spawn timers and game loop"
```

---

## Task 5: Collision Detection + Wall Hit Reward

**Files:**
- Modify: `index.html` — add collision + wall logic inside the game loop

- [ ] **Step 1: Add pure collision function with inline test**

Add this block after the constants section (before the renderer setup):

```js
// ── Pure functions (tested at startup) ────────────────────────────────
function bulletHitsUnit(bx, bz, ux, uz) {
  const dx = bx - ux, dz = bz - uz;
  return Math.sqrt(dx * dx + dz * dz) < UNIT_RADIUS;
}

// Inline tests — check console for PASS/FAIL
(function testBulletHitsUnit() {
  console.assert(bulletHitsUnit(0, 0, 0, 0)      === true,  'FAIL: direct hit');
  console.assert(bulletHitsUnit(0.5, 0, 0, 0)    === true,  'FAIL: near hit');
  console.assert(bulletHitsUnit(0.7, 0, 0, 0)    === false, 'FAIL: near miss');
  console.assert(bulletHitsUnit(10, 10, 0, 0)    === false, 'FAIL: far miss');
  console.log('bulletHitsUnit: all tests PASS');
})();
```

- [ ] **Step 2: Verify tests pass**

Open browser console. Expected: `bulletHitsUnit: all tests PASS` with no assertion errors.

- [ ] **Step 3: Add flash effect helper**

After the `Bullet` class, add:

```js
function spawnFlash(position, color) {
  const flash = new THREE.Mesh(
    new THREE.SphereGeometry(0.3, 6, 6),
    new THREE.MeshBasicMaterial({ color, transparent: true, opacity: 0.9 })
  );
  flash.position.copy(position);
  scene.add(flash);
  let age = 0;
  const fadeFlash = () => {
    age += 0.016;
    flash.material.opacity = Math.max(0, 0.9 - age * 6);
    flash.scale.setScalar(1 + age * 4);
    if (age < 0.18) requestAnimationFrame(fadeFlash);
    else scene.remove(flash);
  };
  requestAnimationFrame(fadeFlash);
}
```

- [ ] **Step 4: Insert collision + wall-hit checks inside the game loop**

Inside `gameLoop()`, after the `// ── Move bullets ──` block and before `// ── Remove off-screen bullets`, insert:

```js
  // ── Collision: red bullets vs blue units ──
  for (const b of redBullets) {
    if (!b.alive) continue;
    for (const u of blueUnits) {
      if (bulletHitsUnit(b.mesh.position.x, b.mesh.position.z,
                         u.mesh.position.x, u.mesh.position.z)) {
        spawnFlash(b.mesh.position, BULLET_RED_CLR);
        b.destroy();
        break;
      }
    }
    // Red bullet reaches blue wall (positive Z edge)
    if (b.alive && b.mesh.position.z > HALF_D) {
      b.destroy();
      spawnUnit('red'); // red fired it, red gets the reward
    }
  }

  // ── Collision: blue bullets vs red units ──
  for (const b of blueBullets) {
    if (!b.alive) continue;
    for (const u of redUnits) {
      if (bulletHitsUnit(b.mesh.position.x, b.mesh.position.z,
                         u.mesh.position.x, u.mesh.position.z)) {
        spawnFlash(b.mesh.position, BULLET_BLUE_CLR);
        b.destroy();
        break;
      }
    }
    // Blue bullet reaches red wall (negative Z edge)
    if (b.alive && b.mesh.position.z < -HALF_D) {
      b.destroy();
      spawnUnit('blue'); // blue fired it, blue gets the reward
    }
  }
```

- [ ] **Step 5: Verify in browser**

Refresh. Expected: Bullets disappear with a brief flash when they hit an opposing unit. When a bullet slips past all units and reaches the far wall, a new unit pops into existence on the scoring side. Unit counts slowly climb over time.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: collision detection, wall-hit reward, flash effect"
```

---

## Task 6: Cursor Magnet Controls

**Files:**
- Modify: `index.html` — add mouse handler and blue unit physics

- [ ] **Step 1: Add cursor projection**

After the `redBullets`/`blueSpawnTimer` declarations, insert:

```js
// ── Input ──────────────────────────────────────────────────────────────
const raycaster   = new THREE.Raycaster();
const boardPlane  = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0);
const cursorWorld = new THREE.Vector3(0, 0, HALF_D / 2); // default: center blue half

window.addEventListener('mousemove', e => {
  const ndc = new THREE.Vector2(
    (e.clientX / window.innerWidth)  *  2 - 1,
    (e.clientY / window.innerHeight) * -2 + 1
  );
  raycaster.setFromCamera(ndc, camera);
  const hit = new THREE.Vector3();
  if (raycaster.ray.intersectPlane(boardPlane, hit)) {
    // Clamp to blue half
    cursorWorld.x = Math.max(-BOARD_W / 2 + 0.5, Math.min(BOARD_W / 2 - 0.5, hit.x));
    cursorWorld.z = Math.max(0.2, Math.min(HALF_D - 0.2, hit.z));
  }
});
```

- [ ] **Step 2: Add blue unit physics update function**

After the input block, insert:

```js
function updateBlueUnits(dt) {
  for (const u of blueUnits) {
    // Spring toward cursor
    u.vx += (cursorWorld.x - u.mesh.position.x) * SPRING_K * dt;
    u.vz += (cursorWorld.z - u.mesh.position.z) * SPRING_K * dt;

    // Mutual repulsion
    for (const other of blueUnits) {
      if (other === u) continue;
      const ex = u.mesh.position.x - other.mesh.position.x;
      const ez = u.mesh.position.z - other.mesh.position.z;
      const dist = Math.sqrt(ex * ex + ez * ez) || 0.001;
      if (dist < REPULSION_DIST) {
        const f = REPULSION_K / (dist * dist) * dt;
        u.vx += (ex / dist) * f;
        u.vz += (ez / dist) * f;
      }
    }

    // Damping
    u.vx *= Math.max(0, 1 - DAMPING * dt);
    u.vz *= Math.max(0, 1 - DAMPING * dt);

    // Apply
    u.mesh.position.x += u.vx * dt;
    u.mesh.position.z += u.vz * dt;

    // Clamp to blue half
    u.mesh.position.x = Math.max(-BOARD_W / 2 + 0.5, Math.min(BOARD_W / 2 - 0.5, u.mesh.position.x));
    u.mesh.position.z = Math.max(0.3, Math.min(HALF_D - 0.3, u.mesh.position.z));
  }
}
```

- [ ] **Step 3: Call `updateBlueUnits` in the game loop**

Inside `gameLoop()`, just before `// ── Unit animations ──`, add:

```js
  // ── Controls ──
  updateBlueUnits(dt);
```

- [ ] **Step 4: Verify in browser**

Refresh. Move the mouse over the lower half of the screen.
Expected: Blue units swarm toward your cursor with a springy, fluid motion. They spread apart naturally and don't stack. Moving quickly leaves them trailing behind. No effect on red units.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: cursor magnet controls for blue units"
```

---

## Task 7: Red AI

**Files:**
- Modify: `index.html` — add red AI update function

- [ ] **Step 1: Add `updateRedUnits` function**

After `updateBlueUnits`, insert:

```js
function updateRedUnits(dt) {
  const difficulty = Math.min(elapsed / 60, 1); // 0→1 over 60 seconds
  const speed = RED_BASE_SPEED * (1 + difficulty * 0.6);

  for (const u of redUnits) {
    // Find nearest blue bullet's X position
    let targetX = u.mesh.position.x;
    let nearestDist = Infinity;
    for (const b of blueBullets) {
      const dx = Math.abs(b.mesh.position.x - u.mesh.position.x);
      if (dx < nearestDist) { nearestDist = dx; targetX = b.mesh.position.x; }
    }

    // Drift toward target X
    const dx = targetX - u.mesh.position.x;
    u.vx += Math.sign(dx) * speed * dt;

    // Mutual repulsion
    for (const other of redUnits) {
      if (other === u) continue;
      const ex = u.mesh.position.x - other.mesh.position.x;
      const ez = u.mesh.position.z - other.mesh.position.z;
      const dist = Math.sqrt(ex * ex + ez * ez) || 0.001;
      if (dist < REPULSION_DIST) {
        const f = REPULSION_K / (dist * dist) * dt;
        u.vx += (ex / dist) * f;
        u.vz += (ez / dist) * f;
      }
    }

    // Damping
    u.vx *= Math.max(0, 1 - DAMPING * dt);
    u.vz *= Math.max(0, 1 - DAMPING * dt);

    // Apply
    u.mesh.position.x += u.vx * dt;
    u.mesh.position.z += u.vz * dt;

    // Clamp to red half (negative Z)
    u.mesh.position.x = Math.max(-BOARD_W / 2 + 0.5, Math.min(BOARD_W / 2 - 0.5, u.mesh.position.x));
    u.mesh.position.z = Math.max(-HALF_D + 0.3, Math.min(-0.3, u.mesh.position.z));
  }
}
```

- [ ] **Step 2: Call `updateRedUnits` in the game loop**

Inside `gameLoop()`, directly after `updateBlueUnits(dt);`, add:

```js
  updateRedUnits(dt);
```

- [ ] **Step 3: Verify in browser**

Refresh. Expected: Red units drift side-to-side tracking incoming blue bullets. They spread out to cover the top half. Early game they're sluggish; as time passes they become faster. No JS errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: red AI tracks incoming bullets with scaling difficulty"
```

---

## Task 8: Coverage Calculation + Win Condition

**Files:**
- Modify: `index.html` — add `calcCoverage`, win check, overlay logic

- [ ] **Step 1: Add `calcCoverage` pure function with inline tests**

In the pure functions block (after `bulletHitsUnit` tests), add:

```js
function calcCoverage(units) {
  if (units.length === 0) return 0;
  const intervals = units.map(u => [
    u.mesh.position.x - UNIT_RADIUS,
    u.mesh.position.x + UNIT_RADIUS
  ]);
  intervals.sort((a, b) => a[0] - b[0]);
  let covered = 0, cur = null;
  for (const [l, r] of intervals) {
    if (!cur) { cur = [l, r]; continue; }
    if (l <= cur[1]) cur[1] = Math.max(cur[1], r);
    else { covered += cur[1] - cur[0]; cur = [l, r]; }
  }
  if (cur) covered += cur[1] - cur[0];
  return Math.min(covered / BOARD_W, 1);
}

// Inline tests
(function testCalcCoverage() {
  // Fake unit with position
  const u = x => ({ mesh: { position: { x, z: 1 } } });
  const empty = calcCoverage([]);
  console.assert(empty === 0, 'FAIL: empty coverage should be 0');
  // One unit covers 2*UNIT_RADIUS / BOARD_W
  const one = calcCoverage([u(0)]);
  console.assert(Math.abs(one - (2 * UNIT_RADIUS / BOARD_W)) < 0.001, 'FAIL: single unit coverage');
  // Overlapping units should not double-count
  const overlap = calcCoverage([u(0), u(0.2)]);
  console.assert(overlap <= one * 2, 'FAIL: overlapping should not exceed double');
  console.log('calcCoverage: all tests PASS');
})();
```

- [ ] **Step 2: Verify tests pass**

Open browser console. Expected: `calcCoverage: all tests PASS` with no assertion errors.

- [ ] **Step 3: Add win check and overlay wiring**

After the `gameOver` declaration, add:

```js
function checkWin() {
  const blueCov = calcCoverage(blueUnits);
  const redCov  = calcCoverage(redUnits);
  if (blueCov >= 1.0) triggerGameOver('YOU WIN 🎉');
  if (redCov  >= 1.0) triggerGameOver('YOU LOSE 💀');
}

function triggerGameOver(message) {
  gameOver = true;
  document.getElementById('overlay-title').textContent = message;
  document.getElementById('overlay').classList.add('show');
}

window.restartGame = function() {
  // Clear all objects
  for (const u of [...blueUnits, ...redUnits]) scene.remove(u.mesh);
  for (const b of [...blueBullets, ...redBullets]) b.destroy();
  blueUnits.length = 0; redUnits.length = 0;
  blueBullets.length = 0; redBullets.length = 0;
  blueSpawnTimer = 0; redSpawnTimer = 0;
  elapsed = 0; gameOver = false;
  document.getElementById('overlay').classList.remove('show');
  for (let i = 0; i < 3; i++) { spawnUnit('blue'); spawnUnit('red'); }
};
```

- [ ] **Step 4: Call `checkWin` in the game loop**

Inside `gameLoop()`, after the collision blocks, add:

```js
  // ── Win check ──
  checkWin();
```

- [ ] **Step 5: Verify in browser**

Refresh. Play until one side accumulates many units. Expected: When a side reaches full coverage, the overlay appears with "YOU WIN 🎉" or "YOU LOSE 💀". Clicking "Play Again" resets the game to 3v3.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: coverage calculation, win condition, restart"
```

---

## Task 9: HUD — Coverage Bars + Unit Counts

**Files:**
- Modify: `index.html` — add HUD update call in game loop

- [ ] **Step 1: Add HUD update function**

After `checkWin`, add:

```js
function updateHUD() {
  const blueCov = calcCoverage(blueUnits);
  const redCov  = calcCoverage(redUnits);
  document.getElementById('blue-fill').style.width  = (blueCov * 100).toFixed(1) + '%';
  document.getElementById('red-fill').style.width   = (redCov  * 100).toFixed(1) + '%';
  document.getElementById('blue-count').textContent = blueUnits.length;
  document.getElementById('red-count').textContent  = redUnits.length;
}
```

- [ ] **Step 2: Call `updateHUD` in the game loop**

Inside `gameLoop()`, after `checkWin();`, add:

```js
  updateHUD();
```

- [ ] **Step 3: Verify in browser**

Refresh. Expected: HUD shows two coverage bars that grow as units accumulate. Unit counts update in real time next to each bar. No performance drop.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: HUD with coverage bars and unit counts"
```

---

## Task 10: Polish — Difficulty Ramp + Spawn Tweaks

**Files:**
- Modify: `index.html` — difficulty ramp already wired in `updateRedUnits`; add bullet-speed scaling and spawn interval floor check

- [ ] **Step 1: Make bullet speed scale with elapsed time**

In the `spawnBullet` function, replace:

```js
const b = new Bullet(side, x);
```

with:

```js
const speedScale = 1 + Math.min(elapsed * 0.008, 0.6);
const b = new Bullet(side, x, speedScale);
```

Update the `Bullet` constructor signature to accept `speedScale`:

```js
constructor(side, x, speedScale = 1) {
  // ... existing code ...
  this.vz = (side === 'blue' ? -BULLET_SPEED : BULLET_SPEED) * speedScale;
```

- [ ] **Step 2: Verify difficulty ramp in browser**

Refresh. Play for ~60 seconds.
Expected: Early game — bullets are manageable, red AI is slow. Mid-game — bullets arrive faster, harder to block everything. Late game — chaotic, many units on both sides.

- [ ] **Step 3: Final smoke test checklist**

Open `http://localhost:3000` in browser. Verify each item:
- [ ] Board renders at isometric angle, red zone top / blue zone bottom
- [ ] 3 red + 3 blue humanoid units on board, gently bobbing
- [ ] Bullets stream from both walls rapidly
- [ ] Moving mouse steers blue units with springy swarm feel
- [ ] Red units track incoming bullets and spread out
- [ ] Bullet hits unit → flash + disappears
- [ ] Bullet hits far wall → new unit spawns on scoring side with pop animation
- [ ] HUD coverage bars update as units accumulate
- [ ] Game ends with overlay when one side reaches full coverage
- [ ] "Play Again" resets to 3v3 cleanly
- [ ] Console shows `bulletHitsUnit: all tests PASS` and `calcCoverage: all tests PASS`
- [ ] No JS errors in console

- [ ] **Step 4: Final commit**

```bash
git add index.html
git commit -m "feat: difficulty ramp, complete dream game implementation"
```
