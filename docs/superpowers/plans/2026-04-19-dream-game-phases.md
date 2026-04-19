# Dream Game – Phases & Levels Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a green typewriter intro, a level-select screen, Medium/Hard difficulties for Phase 1, and a full Hotline Miami-aesthetic Phase 2 — all inside the single `index.html` file.

**Architecture:** Everything lives in one HTML file. Screens (story, level-select, game, overlay) are layered `position:fixed` divs shown/hidden by JS. Phase 1 uses the existing Three.js isometric scene. Phase 2 swaps in a second Three.js scene (top-down camera, neon palette, different board) by hiding the Phase 1 canvas and showing a Phase 2 canvas. Difficulty is applied by reading a `DIFFICULTY_CONFIGS` map at game-start and overwriting the live constants before spawning units.

**Tech Stack:** Vanilla JS ES modules, Three.js r0.162 (importmap CDN), CSS animations for typewriter blink.

---

## File Structure

- **Modify only:** `C:\Users\lenovo\dreamgame\index.html`
  - `<style>` block — typewriter CSS, level-select CSS, phase-2 HUD/overlay overrides
  - `#story-screen` div — replace static text with empty spans filled by typewriter JS
  - New `#level-select` div — Easy / Medium / Hard / Phase 2 buttons
  - `<script type="module">` — all JS changes below

- **Plan file:** `C:\Users\lenovo\dreamgame\docs\superpowers\plans\2026-04-19-dream-game-phases.md` (this file)

---

## Task 1: Green Typewriter Intro

**Files:**
- Modify: `C:\Users\lenovo\dreamgame\index.html` — `<style>` block + `#story-screen` HTML + `startFromStory` JS

### What it does
Story text types out character by character in bright green (`#00ff41` — Matrix green). A blinking `_` cursor follows each character. When all text is typed, the **Start** button fades in.

- [ ] **Step 1: Add typewriter CSS**

Inside the `<style>` block, after `.story-content button:hover { ... }`, add:

```css
/* ── Typewriter ─────────────────────────────────────────────── */
.tw-line { color: #00ff41; display: block; min-height: 1.9em; }
.tw-cursor {
  display: inline-block; width: 2px; height: 1em;
  background: #00ff41; margin-left: 2px; vertical-align: text-bottom;
  animation: blink 0.7s step-end infinite;
}
@keyframes blink { 0%,100%{opacity:1} 50%{opacity:0} }
#story-btn { opacity: 0; transition: opacity 0.6s; pointer-events: none; }
#story-btn.ready { opacity: 1; pointer-events: auto; }
```

- [ ] **Step 2: Replace static story HTML with typewriter spans**

Replace the entire `<div class="story-content">` block with:

```html
<div class="story-content">
  <div class="subtitle">A true story</div>
  <h1>The Dream</h1>

  <p id="tw-p1"><span class="tw-line"></span></p>
  <p id="tw-p2"><span class="tw-line"></span></p>
  <p id="tw-p3"><span class="tw-line"></span></p>
  <p id="tw-p4"><span class="tw-line"></span></p>
  <p id="tw-p5"><span class="tw-line"></span></p>
  <p id="tw-p6" style="font-size:12px; color:#666;"></p>

  <div class="phase-label">Now Playing</div>
  <div class="phase-title" id="story-phase-title">Phase 1</div>

  <button id="story-btn" onclick="window.startFromStory()">Start</button>
</div>
```

- [ ] **Step 3: Add typewriter JS — place it just before `window.startFromStory`**

```js
// ── Typewriter intro ───────────────────────────────────────────
(function runTypewriter() {
  const lines = [
    { id: 'tw-p1', text: 'One night I had a dream about a game.' },
    { id: 'tw-p2', text: 'Blue vs Red. Two armies firing at each other across a glowing board. Every bullet that got through the enemy lines added a new soldier to your side. The side that took over the whole board would win.' },
    { id: 'tw-p3', text: 'I woke up thinking it was a real game I had played before. It was not.' },
    { id: 'tw-p4', text: 'So I went to Claude. No coding skills, no experience at all. I just described what I saw in the dream. We talked, I gave feedback, and bit by bit it became real.' },
    { id: 'tw-p5', text: 'This is Phase 1. The game exactly as I dreamed it.' },
    { id: 'tw-p6', text: 'Phase 2 is coming. Something bigger.' },
  ];

  const CHAR_DELAY = 28;   // ms per character
  const LINE_PAUSE = 320;  // ms pause between lines

  let lineIdx = 0;
  let charIdx = 0;
  let cursor = null;

  function nextChar() {
    if (lineIdx >= lines.length) {
      // All done — remove cursor, show button
      if (cursor) cursor.remove();
      document.getElementById('story-btn').classList.add('ready');
      return;
    }

    const { id, text } = lines[lineIdx];
    const span = document.querySelector('#' + id + ' .tw-line') || document.getElementById(id);

    if (charIdx === 0 && cursor) cursor.remove();

    if (charIdx < text.length) {
      span.textContent = text.slice(0, charIdx + 1);
      // Re-attach cursor after span
      cursor = document.createElement('span');
      cursor.className = 'tw-cursor';
      span.after(cursor);
      charIdx++;
      setTimeout(nextChar, CHAR_DELAY);
    } else {
      lineIdx++;
      charIdx = 0;
      setTimeout(nextChar, LINE_PAUSE);
    }
  }

  // Start after a short initial pause
  setTimeout(nextChar, 600);
})();
```

- [ ] **Step 4: Verify in browser**

Reload the page. You should see green text typing out character by character in each paragraph, with a blinking cursor following. After the last line finishes, the **Start** button fades in. Check browser console — no errors.

- [ ] **Step 5: Commit**

```bash
git add C:/Users/lenovo/dreamgame/index.html
git commit -m "feat: green typewriter intro with blinking cursor"
```

---

## Task 2: Level Select Screen

**Files:**
- Modify: `C:\Users\lenovo\dreamgame\index.html` — add `#level-select` div + CSS + JS routing

### What it does
After the story **Start** button is clicked, hide the story screen and show a level-select screen with four cards: **Easy**, **Medium**, **Hard** (all Phase 1), and **Phase 2** (Hotline Miami, locked until Phase 1 Hard is beaten — stored in `localStorage`). Clicking a card starts that mode.

- [ ] **Step 1: Add level-select CSS** (inside `<style>`, after typewriter CSS)

```css
/* ── Level Select ───────────────────────────────────────────── */
#level-select {
  position: fixed; inset: 0; z-index: 99; display: none;
  align-items: center; justify-content: center; flex-direction: column; gap: 32px;
  background: #0a0a0f; font-family: monospace; color: #fff;
}
#level-select.show { display: flex; }
#level-select h2 {
  font-size: 13px; letter-spacing: 4px; text-transform: uppercase; color: #555;
}
.level-cards { display: flex; gap: 16px; flex-wrap: wrap; justify-content: center; }
.level-card {
  width: 140px; padding: 24px 16px; text-align: center;
  border: 1px solid #333; border-radius: 6px; cursor: pointer;
  transition: border-color 0.2s, background 0.2s;
  user-select: none;
}
.level-card:hover:not(.locked) { border-color: #00ff41; background: rgba(0,255,65,0.06); }
.level-card.locked { opacity: 0.35; cursor: default; }
.level-card .card-phase {
  font-size: 10px; letter-spacing: 3px; text-transform: uppercase;
  color: #555; margin-bottom: 8px;
}
.level-card .card-name { font-size: 20px; color: #fff; margin-bottom: 6px; }
.level-card .card-sub  { font-size: 11px; color: #555; }
.level-card.easy   { border-color: #2255cc; }
.level-card.medium { border-color: #cc8822; }
.level-card.hard   { border-color: #cc2222; }
.level-card.phase2 { border-color: #ff2d78; }
.level-card.easy:hover   { border-color: #4499ff; background: rgba(68,153,255,0.06); }
.level-card.medium:hover:not(.locked) { border-color: #ffaa44; background: rgba(255,170,68,0.06); }
.level-card.hard:hover:not(.locked)   { border-color: #ff4444; background: rgba(255,68,68,0.06); }
.level-card.phase2:hover:not(.locked) { border-color: #ff2d78; background: rgba(255,45,120,0.06); }
```

- [ ] **Step 2: Add `#level-select` HTML** (after `#story-screen` div, before `#hud` div)

```html
<!-- Level select -->
<div id="level-select">
  <h2>Select Level</h2>
  <div class="level-cards">
    <div class="level-card easy" onclick="window.selectLevel('easy')">
      <div class="card-phase">Phase 1</div>
      <div class="card-name">Easy</div>
      <div class="card-sub">The Dream</div>
    </div>
    <div class="level-card medium" id="card-medium" onclick="window.selectLevel('medium')">
      <div class="card-phase">Phase 1</div>
      <div class="card-name">Medium</div>
      <div class="card-sub">Faster</div>
    </div>
    <div class="level-card hard" id="card-hard" onclick="window.selectLevel('hard')">
      <div class="card-phase">Phase 1</div>
      <div class="card-name">Hard</div>
      <div class="card-sub">No mercy</div>
    </div>
    <div class="level-card phase2" id="card-phase2" onclick="window.selectLevel('phase2')">
      <div class="card-phase">Phase 2</div>
      <div class="card-name">Miami</div>
      <div class="card-sub" id="card-phase2-sub">Beat Hard first</div>
    </div>
  </div>
</div>
```

- [ ] **Step 3: Update `startFromStory` to show level-select instead of starting game directly**

Replace:
```js
window.startFromStory = function() {
  document.getElementById('story-screen').style.display = 'none';
  gamePaused = false;
};
```

With:
```js
window.startFromStory = function() {
  document.getElementById('story-screen').style.display = 'none';
  showLevelSelect();
};

function showLevelSelect() {
  // Refresh lock state from localStorage
  const unlocked = JSON.parse(localStorage.getItem('dg_unlocked') || '["easy"]');
  ['medium','hard','phase2'].forEach(lvl => {
    const card = document.getElementById('card-' + lvl);
    if (card) card.classList.toggle('locked', !unlocked.includes(lvl));
  });
  const p2sub = document.getElementById('card-phase2-sub');
  if (p2sub) p2sub.textContent = unlocked.includes('phase2') ? 'Hotline Miami' : 'Beat Hard first';
  document.getElementById('level-select').classList.add('show');
}

window.selectLevel = function(level) {
  const unlocked = JSON.parse(localStorage.getItem('dg_unlocked') || '["easy"]');
  if (!unlocked.includes(level)) return; // locked
  currentLevel = level;
  document.getElementById('level-select').classList.remove('show');
  if (level === 'phase2') {
    startPhase2();
  } else {
    applyDifficulty(level);
    gamePaused = false;
  }
};
```

- [ ] **Step 4: Add `currentLevel` state variable** (near `let gameOver`)

```js
let currentLevel = 'easy';
```

- [ ] **Step 5: Verify level select renders**

Reload page. Type-out completes, click Start. Level-select screen appears with four cards. Easy is clickable; Medium/Hard/Phase2 show as locked (dim). Easy card click dismisses screen and starts game. Console: no errors.

- [ ] **Step 6: Commit**

```bash
git add C:/Users/lenovo/dreamgame/index.html
git commit -m "feat: level select screen with lock state from localStorage"
```

---

## Task 3: Difficulty Configs + Medium / Hard for Phase 1

**Files:**
- Modify: `C:\Users\lenovo\dreamgame\index.html` — constants block + `applyDifficulty` + AI update + win/restart

### What it does
Define a config per difficulty that overrides bullet speed, spawn interval, AI speed, and starting unit count. Medium and Hard also use more aggressive AI with a shorter ramp-up time.

- [ ] **Step 1: Add `DIFFICULTY_CONFIGS` map** (replace the existing constants block `BULLET_SPEED` through `RED_BASE_SPEED`)

Keep all constant declarations at the top but make them `let` so `applyDifficulty` can overwrite them:

```js
// ── Difficulty config ──────────────────────────────────────────────────────
const DIFFICULTY_CONFIGS = {
  easy: {
    bulletSpeed:    6.0,
    spawnInterval:  0.22,
    aiSpeedStart:   3.5,
    aiSpeedEnd:     6.0,
    aiRampSeconds:  90,
    startUnits:     3,
    winScore:       100,
  },
  medium: {
    bulletSpeed:    9.0,
    spawnInterval:  0.16,
    aiSpeedStart:   6.0,
    aiSpeedEnd:     11.0,
    aiRampSeconds:  60,
    startUnits:     3,
    winScore:       100,
  },
  hard: {
    bulletSpeed:    13.0,
    spawnInterval:  0.11,
    aiSpeedStart:   10.0,
    aiSpeedEnd:     18.0,
    aiRampSeconds:  40,
    startUnits:     3,
    winScore:       100,
  },
};

let BULLET_SPEED    = 6.0;
let SPAWN_INTERVAL  = 0.22;
let RED_BASE_SPEED  = 3.5;   // AI cursor start speed (units/s)
let RED_END_SPEED   = 6.0;   // AI cursor end speed
let AI_RAMP_SECS    = 90;
let WIN_SCORE       = 100;
```

- [ ] **Step 2: Add `applyDifficulty` function** (after the `DIFFICULTY_CONFIGS` block)

```js
function applyDifficulty(level) {
  const cfg = DIFFICULTY_CONFIGS[level] || DIFFICULTY_CONFIGS.easy;
  BULLET_SPEED   = cfg.bulletSpeed;
  SPAWN_INTERVAL = cfg.spawnInterval;
  RED_BASE_SPEED = cfg.aiSpeedStart;
  RED_END_SPEED  = cfg.aiSpeedEnd;
  AI_RAMP_SECS   = cfg.aiRampSeconds;
  WIN_SCORE      = cfg.winScore;
  // Update HUD /100 labels
  document.querySelectorAll('#hud span').forEach(s => {
    if (s.textContent === '/100') s.textContent = '/' + WIN_SCORE;
  });
  // Update HUD level badge if present
  const badge = document.getElementById('hud-level');
  if (badge) badge.textContent = level.toUpperCase();
}
```

- [ ] **Step 3: Update `updateRedUnits` to use `RED_BASE_SPEED`, `RED_END_SPEED`, `AI_RAMP_SECS`**

Find the AI speed lines (currently hardcoded `3.5 + difficulty * 2.5`):

```js
// OLD:
const difficulty = Math.min(elapsed / 90, 1);
const cursorSpeed = (3.5 + difficulty * 2.5) * dt;

// NEW:
const difficulty = Math.min(elapsed / AI_RAMP_SECS, 1);
const cursorSpeed = (RED_BASE_SPEED + difficulty * (RED_END_SPEED - RED_BASE_SPEED)) * dt;
```

- [ ] **Step 4: Add a level badge to the HUD** so the player always knows which difficulty they're on.

Inside `#hud` div, add after the second bar-wrap:

```html
<span id="hud-level" style="font-size:11px;letter-spacing:2px;color:#888;text-transform:uppercase;">EASY</span>
```

- [ ] **Step 5: Update `restartGame` to re-apply current difficulty**

Inside `window.restartGame`, after resetting scores, add:

```js
applyDifficulty(currentLevel);
```

- [ ] **Step 6: Update `triggerGameOver` to unlock next level on win**

```js
function triggerGameOver(message) {
  gameOver = true;
  document.getElementById('overlay-title').textContent = message;

  const won = message.includes('WIN');
  const nextMap = { easy: 'medium', medium: 'hard', hard: 'phase2' };
  const next = nextMap[currentLevel];

  if (won && next) {
    const unlocked = JSON.parse(localStorage.getItem('dg_unlocked') || '["easy"]');
    if (!unlocked.includes(next)) {
      unlocked.push(next);
      localStorage.setItem('dg_unlocked', JSON.stringify(unlocked));
    }
  }

  const nextDiv = document.getElementById('overlay-next');
  const nextTitle = document.getElementById('overlay-next-title');
  if (won && next) {
    nextDiv.style.display = 'block';
    const labels = { medium: 'Phase 1 / Medium', hard: 'Phase 1 / Hard', phase2: 'Phase 2 / Miami' };
    nextTitle.textContent = labels[next];
  } else {
    nextDiv.style.display = 'none';
  }

  document.getElementById('overlay').classList.add('show');
}
```

- [ ] **Step 7: Update win overlay HTML to have correct IDs**

Replace the `#overlay-next` block with:

```html
<div id="overlay-next" style="display:none">
  <div class="next-label">Unlocked</div>
  <div class="next-title" id="overlay-next-title">Phase 1 / Medium</div>
  <button style="margin-top:8px;background:#222;border:1px solid #555;"
          onclick="document.getElementById('overlay').classList.remove('show');showLevelSelect()">
    Choose Level
  </button>
</div>
```

- [ ] **Step 8: Verify in browser**

1. Start Easy — confirm game plays at current speed
2. Open console: `window.selectLevel('medium')` — verify bullets are noticeably faster and AI reacts quicker
3. Open console: `window.selectLevel('hard')` — bullets very fast, AI aggressive

- [ ] **Step 9: Commit**

```bash
git add C:/Users/lenovo/dreamgame/index.html
git commit -m "feat: difficulty configs easy/medium/hard with AI scaling and level unlock"
```

---

## Task 4: Phase 2 — Hotline Miami Aesthetic + Scene

**Files:**
- Modify: `C:\Users\lenovo\dreamgame\index.html` — Phase 2 CSS + canvas + Three.js scene + game loop

### What it does
Phase 2 swaps to a **top-down** camera, **neon Miami palette** (hot pink `#ff2d78`, cyan `#00fff0`, dark purple `#0d0015` background, neon grid floor), and **faster/more chaotic** gameplay. Units look like neon silhouettes. Bullets leave a short neon trail. Otherwise the core mechanics (blue vs red, cursor swarm, scatter physics) remain identical — just turned up and reskinned.

Gameplay differences from Phase 1:
- Bullets travel diagonally (small random X velocity so they aren't perfectly straight)
- AI tracks blue bullets AND player cursor position simultaneously (split attention)
- Board is narrower: 24×24 (more claustrophobic)
- WIN_SCORE = 150
- Bullet speed: 14; spawn interval: 0.10

- [ ] **Step 1: Add Phase 2 CSS** (inside `<style>`, after level-select CSS)

```css
/* ── Phase 2 / Miami ───────────────────────────────────────── */
#p2-canvas-wrap {
  position: fixed; inset: 0; z-index: 1; display: none;
}
#p2-canvas-wrap.show { display: block; }
#p2-hud {
  position: fixed; top: 16px; left: 50%; transform: translateX(-50%);
  display: none; gap: 24px; align-items: center;
  font-family: monospace; color: #ff2d78; font-size: 13px;
  background: rgba(13,0,21,0.8); padding: 8px 16px; border-radius: 0;
  border: 1px solid #ff2d78; pointer-events: none; letter-spacing: 2px;
}
#p2-hud.show { display: flex; }
.p2-bar { width: 120px; height: 8px; background: #200030; }
.p2-fill { height: 100%; width: 0%; }
.p2-fill.blue { background: #00fff0; }
.p2-fill.pink { background: #ff2d78; }
```

- [ ] **Step 2: Add Phase 2 canvas wrapper + HUD** (after `#level-select` div, before `#hud` div)

```html
<!-- Phase 2 canvas (hidden until activated) -->
<div id="p2-canvas-wrap"></div>

<!-- Phase 2 HUD -->
<div id="p2-hud">
  <div class="bar-wrap">
    <span>YOU</span>
    <div class="p2-bar"><div id="p2-blue-fill" class="p2-fill blue"></div></div>
    <span id="p2-blue-score">0</span><span>/150</span>
  </div>
  <div class="bar-wrap">
    <span>THEM</span>
    <div class="p2-bar"><div id="p2-red-fill" class="p2-fill pink"></div></div>
    <span id="p2-red-score">0</span><span>/150</span>
  </div>
  <span style="font-size:10px;color:#ff2d78;letter-spacing:3px;">MIAMI</span>
</div>
```

- [ ] **Step 3: Add `startPhase2` function** — create a fully self-contained Phase 2 scene.

Add this entire function block right before the closing `</script>` tag:

```js
// ── Phase 2 — Hotline Miami ────────────────────────────────────────────────
function startPhase2() {
  // Show Phase 2 canvas, hide Phase 1 canvas
  renderer.domElement.style.display = 'none';
  document.getElementById('hud').style.display = 'none';
  document.getElementById('p2-hud').classList.add('show');

  const wrap = document.getElementById('p2-canvas-wrap');
  wrap.classList.add('show');

  // ── P2 Renderer ──
  const p2renderer = new THREE.WebGLRenderer({ antialias: true });
  p2renderer.setSize(window.innerWidth, window.innerHeight);
  p2renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  wrap.appendChild(p2renderer.domElement);

  // ── P2 Scene ──
  const p2scene = new THREE.Scene();
  p2scene.background = new THREE.Color(0x0d0015);

  // Top-down orthographic camera
  const p2aspect = window.innerWidth / window.innerHeight;
  const p2d = 16;
  const p2camera = new THREE.OrthographicCamera(
    -p2d * p2aspect, p2d * p2aspect, p2d, -p2d, 0.1, 200
  );
  p2camera.position.set(0, 30, 0);
  p2camera.lookAt(0, 0, 0);

  // ── P2 Board ──
  const P2_W = 24, P2_D = 24, P2_HALF = 12;

  // Dark floor
  const floorGeo = new THREE.PlaneGeometry(P2_W, P2_D);
  const floorMat = new THREE.MeshBasicMaterial({ color: 0x0d0015 });
  const floor = new THREE.Mesh(floorGeo, floorMat);
  floor.rotation.x = -Math.PI / 2;
  p2scene.add(floor);

  // Neon grid lines
  const gridMat = new THREE.LineBasicMaterial({ color: 0x330044 });
  for (let i = -P2_W / 2; i <= P2_W / 2; i += 2) {
    const g = new THREE.BufferGeometry().setFromPoints([
      new THREE.Vector3(i, 0.01, -P2_D / 2),
      new THREE.Vector3(i, 0.01,  P2_D / 2),
    ]);
    p2scene.add(new THREE.Line(g, gridMat));
  }
  for (let z = -P2_D / 2; z <= P2_D / 2; z += 2) {
    const g = new THREE.BufferGeometry().setFromPoints([
      new THREE.Vector3(-P2_W / 2, 0.01, z),
      new THREE.Vector3( P2_W / 2, 0.01, z),
    ]);
    p2scene.add(new THREE.Line(g, gridMat));
  }

  // Center divider (hot pink)
  const divMat = new THREE.LineBasicMaterial({ color: 0xff2d78 });
  const divG = new THREE.BufferGeometry().setFromPoints([
    new THREE.Vector3(-P2_W / 2, 0.02, 0),
    new THREE.Vector3( P2_W / 2, 0.02, 0),
  ]);
  p2scene.add(new THREE.Line(divG, divMat));

  // Zone tints
  const makeZone = (color, z) => {
    const m = new THREE.Mesh(
      new THREE.PlaneGeometry(P2_W, P2_HALF),
      new THREE.MeshBasicMaterial({ color, transparent: true, opacity: 0.04, depthWrite: false })
    );
    m.rotation.x = -Math.PI / 2;
    m.position.set(0, 0.01, z);
    p2scene.add(m);
  };
  makeZone(0x00fff0, P2_HALF / 2);   // blue/cyan zone
  makeZone(0xff2d78, -P2_HALF / 2);  // pink/red zone

  // Ambient light only (flat-lit top-down look)
  p2scene.add(new THREE.AmbientLight(0xffffff, 1.0));

  // ── P2 Unit factory ──
  function createP2Unit(color) {
    const g = new THREE.Group();
    // Top-down flat diamond silhouette
    const mat = new THREE.MeshBasicMaterial({ color });
    const body = new THREE.Mesh(new THREE.BoxGeometry(0.7, 0.1, 0.7), mat);
    const head = new THREE.Mesh(new THREE.BoxGeometry(0.3, 0.1, 0.3), new THREE.MeshBasicMaterial({ color: 0xffffff }));
    head.position.set(0, 0.05, -0.22);
    g.add(body);
    g.add(head);
    return g;
  }

  const P2_BLUE = 0x00fff0;
  const P2_RED  = 0xff2d78;

  const p2Blue = [], p2Red = [];

  function p2SpawnUnit(side) {
    const mesh = createP2Unit(side === 'blue' ? P2_BLUE : P2_RED);
    const x = (Math.random() - 0.5) * (P2_W - 1.5);
    const z = side === 'blue'
      ? P2_HALF * 0.1 + Math.random() * (P2_HALF - 0.4)
      : -P2_HALF * 0.1 - Math.random() * (P2_HALF - 0.4);
    mesh.position.set(x, 0.05, z);
    p2scene.add(mesh);
    const u = { mesh, vx: 0, vz: 0, spawnScale: 0 };
    if (side === 'blue') p2Blue.push(u); else p2Red.push(u);
  }

  for (let i = 0; i < 3; i++) { p2SpawnUnit('blue'); p2SpawnUnit('red'); }

  // ── P2 Bullets ──
  const P2_BULLET_GEO  = new THREE.BoxGeometry(0.18, 0.1, 0.18);
  const P2_BULLET_BLUE = new THREE.MeshBasicMaterial({ color: 0x00fff0 });
  const P2_BULLET_RED  = new THREE.MeshBasicMaterial({ color: 0xff2d78 });
  const P2_BULLET_SPEED = 14;
  const P2_WIN = 150;

  const p2BlueBullets = [], p2RedBullets = [];
  let p2BlueSpawn = 0, p2RedSpawn = 0;
  let p2BlueScore = 0, p2RedScore = 0;
  let p2Elapsed = 0;
  let p2Over = false;
  const P2_SPAWN_INT = 0.10;

  // P2 cursor (mouse)
  const p2cursor = { x: 0, z: P2_HALF / 2 };
  let p2RedCursorX = 0;

  const p2Raycaster = new THREE.Raycaster();
  const p2Plane = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0);
  window.addEventListener('mousemove', e => {
    if (!wrap.classList.contains('show')) return;
    const ndc = new THREE.Vector2(
      (e.clientX / window.innerWidth) * 2 - 1,
      (e.clientY / window.innerHeight) * -2 + 1
    );
    p2Raycaster.setFromCamera(ndc, p2camera);
    const hit = new THREE.Vector3();
    if (p2Raycaster.ray.intersectPlane(p2Plane, hit)) {
      p2cursor.x = Math.max(-P2_W / 2 + 0.5, Math.min(P2_W / 2 - 0.5, hit.x));
      p2cursor.z = Math.max(0.2, Math.min(P2_HALF - 0.2, hit.z));
    }
  });

  function p2SpawnBullet(side) {
    const x = (Math.random() - 0.5) * (P2_W - 0.5);
    const vx = (Math.random() - 0.5) * 3.0; // diagonal drift
    const vz = (side === 'blue' ? -1 : 1) * P2_BULLET_SPEED;
    const mat = side === 'blue' ? P2_BULLET_BLUE : P2_BULLET_RED;
    const mesh = new THREE.Mesh(P2_BULLET_GEO, mat);
    const z = side === 'blue' ? P2_HALF : -P2_HALF;
    mesh.position.set(x, 0.15, z);
    // Point light glow
    const light = new THREE.PointLight(side === 'blue' ? 0x00fff0 : 0xff2d78, 1.2, 2.5);
    mesh.add(light);
    p2scene.add(mesh);
    const b = { mesh, vx, vz, alive: true,
      destroy() { p2scene.remove(this.mesh); this.alive = false; }
    };
    if (side === 'blue') p2BlueBullets.push(b); else p2RedBullets.push(b);
  }

  function p2ScatterUnits(units, ix, iz, force) {
    for (const u of units) {
      const dx = u.mesh.position.x - ix;
      const dz = u.mesh.position.z - iz;
      const d = Math.sqrt(dx*dx + dz*dz) || 0.001;
      if (d < 3.0) {
        const s = force * (1 - d / 3.0);
        u.vx += (dx/d)*s; u.vz += (dz/d)*s;
      }
    }
  }

  function p2UpdateUnits(units, tgX, tgZ, halfMin, halfMax, dt) {
    const SK = 14, RD = 1.4, RK = 3.0, DMP = 3.0, MV = 60;
    for (const u of units) {
      u.vx += (tgX - u.mesh.position.x) * SK * dt;
      u.vz += (tgZ - u.mesh.position.z) * SK * dt;
      for (const o of units) {
        if (o === u) continue;
        const ex = u.mesh.position.x - o.mesh.position.x;
        const ez = u.mesh.position.z - o.mesh.position.z;
        const d = Math.sqrt(ex*ex+ez*ez)||0.001;
        if (d < RD) { const f = RK/(d*d)*dt; u.vx+=(ex/d)*f; u.vz+=(ez/d)*f; }
      }
      u.vx *= Math.max(0, 1 - DMP*dt);
      u.vz *= Math.max(0, 1 - DMP*dt);
      u.vx = Math.max(-MV, Math.min(MV, u.vx));
      u.vz = Math.max(-MV, Math.min(MV, u.vz));
      u.mesh.position.x += u.vx * dt;
      u.mesh.position.z += u.vz * dt;
      // Wall bounces
      if (u.mesh.position.x < -P2_W/2+0.5) { u.mesh.position.x=-P2_W/2+0.5; u.vx=45+Math.random()*20; u.vz+=(Math.random()-0.5)*10; }
      else if (u.mesh.position.x > P2_W/2-0.5) { u.mesh.position.x=P2_W/2-0.5; u.vx=-(45+Math.random()*20); u.vz+=(Math.random()-0.5)*10; }
      if (u.mesh.position.z < halfMin+0.3) { u.mesh.position.z=halfMin+0.3; u.vz=30+Math.random()*15; u.vx+=(Math.random()-0.5)*10; }
      else if (u.mesh.position.z > halfMax-0.3) { u.mesh.position.z=halfMax-0.3; u.vz=-(30+Math.random()*15); u.vx+=(Math.random()-0.5)*10; }
    }
  }

  function p2UpdateHUD() {
    document.getElementById('p2-blue-fill').style.width = Math.min(p2BlueScore, P2_WIN) + '%';
    document.getElementById('p2-red-fill').style.width  = Math.min(p2RedScore,  P2_WIN) + '%';
    document.getElementById('p2-blue-score').textContent = p2BlueScore;
    document.getElementById('p2-red-score').textContent  = p2RedScore;
  }

  function p2TriggerGameOver(msg) {
    p2Over = true;
    document.getElementById('overlay-title').textContent = msg;
    document.getElementById('overlay-next').style.display = 'none';
    document.getElementById('overlay').classList.add('show');
  }

  // Resize handler for p2
  window.addEventListener('resize', () => {
    const a = window.innerWidth / window.innerHeight;
    p2camera.left = -p2d * a; p2camera.right = p2d * a;
    p2camera.updateProjectionMatrix();
    p2renderer.setSize(window.innerWidth, window.innerHeight);
    p2renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  });

  let p2Last = performance.now();

  function p2Loop() {
    requestAnimationFrame(p2Loop);
    if (p2Over) { p2renderer.render(p2scene, p2camera); return; }

    const now = performance.now();
    const dt = Math.min((now - p2Last) / 1000, 0.05);
    p2Last = now;
    p2Elapsed += dt;

    // Spawn bullets
    p2BlueSpawn += dt; p2RedSpawn += dt;
    if (p2BlueSpawn >= P2_SPAWN_INT) { p2SpawnBullet('blue'); p2BlueSpawn = 0; }
    if (p2RedSpawn  >= P2_SPAWN_INT) { p2SpawnBullet('red');  p2RedSpawn  = 0; }

    // Move bullets (diagonal)
    for (const b of [...p2BlueBullets, ...p2RedBullets]) {
      if (!b.alive) continue;
      b.mesh.position.x += b.vx * dt;
      b.mesh.position.z += b.vz * dt;
      // Clamp X drift within board
      if (b.mesh.position.x < -P2_W/2) b.mesh.position.x = -P2_W/2;
      if (b.mesh.position.x >  P2_W/2) b.mesh.position.x =  P2_W/2;
    }

    // Remove out-of-bounds bullets
    for (const b of p2BlueBullets) { if (b.alive && b.mesh.position.z < -P2_HALF-1) b.destroy(); }
    for (const b of p2RedBullets)  { if (b.alive && b.mesh.position.z >  P2_HALF+1) b.destroy(); }
    for (let i = p2BlueBullets.length-1; i >= 0; i--) if (!p2BlueBullets[i].alive) p2BlueBullets.splice(i,1);
    for (let i = p2RedBullets.length-1;  i >= 0; i--) if (!p2RedBullets[i].alive)  p2RedBullets.splice(i,1);

    // AI red cursor — weighted by bullet proximity + slight attraction to player cursor
    if (p2BlueBullets.length > 0) {
      let wx = 0, wt = 0;
      for (const b of p2BlueBullets) {
        if (!b.alive) continue;
        const prox = (-b.mesh.position.z + P2_HALF) / (2 * P2_HALF);
        const w = prox * prox + 0.1;
        wx += b.mesh.position.x * w; wt += w;
      }
      // Also pull slightly toward player cursor to apply pressure
      wx += p2cursor.x * 0.4; wt += 0.4;
      const tgt = wx / wt;
      const diff = Math.min(p2Elapsed / 60, 1);
      const spd = (9 + diff * 8) * dt;
      const dx = tgt - p2RedCursorX;
      p2RedCursorX += Math.sign(dx) * Math.min(Math.abs(dx), spd);
      p2RedCursorX = Math.max(-P2_W/2+0.5, Math.min(P2_W/2-0.5, p2RedCursorX));
    }

    // Update units
    p2UpdateUnits(p2Blue, p2cursor.x, p2cursor.z, 0, P2_HALF, dt);
    p2UpdateUnits(p2Red, p2RedCursorX, -P2_HALF/2, -P2_HALF, 0, dt);

    // Collisions: red bullets vs blue units
    for (const b of p2RedBullets) {
      if (!b.alive) continue;
      for (const u of p2Blue) {
        const dx = b.mesh.position.x - u.mesh.position.x;
        const dz = b.mesh.position.z - u.mesh.position.z;
        if (Math.sqrt(dx*dx+dz*dz) < 0.6) {
          p2ScatterUnits(p2Blue, b.mesh.position.x, b.mesh.position.z, 80);
          b.destroy(); break;
        }
      }
      if (b.alive && b.mesh.position.z > P2_HALF) {
        b.destroy(); p2SpawnUnit('red'); p2RedScore++;
      }
    }

    // Collisions: blue bullets vs red units
    for (const b of p2BlueBullets) {
      if (!b.alive) continue;
      for (const u of p2Red) {
        const dx = b.mesh.position.x - u.mesh.position.x;
        const dz = b.mesh.position.z - u.mesh.position.z;
        if (Math.sqrt(dx*dx+dz*dz) < 0.6) {
          p2ScatterUnits(p2Red, b.mesh.position.x, b.mesh.position.z, 80);
          b.destroy(); break;
        }
      }
      if (b.alive && b.mesh.position.z < -P2_HALF) {
        b.destroy(); p2SpawnUnit('blue'); p2BlueScore++;
      }
    }

    // Win check
    if (p2BlueScore >= P2_WIN) p2TriggerGameOver('YOU WIN 🎉');
    if (p2RedScore  >= P2_WIN) p2TriggerGameOver('YOU LOSE 💀');

    p2UpdateHUD();

    // Spawn scale animation
    for (const u of [...p2Blue, ...p2Red]) {
      if (u.spawnScale < 1) {
        u.spawnScale = Math.min(1, u.spawnScale + dt * 4);
        u.mesh.scale.setScalar(u.spawnScale);
      }
    }

    p2renderer.render(p2scene, p2camera);
  }
  p2Loop();
}
```

- [ ] **Step 4: Verify Phase 2 launches**

Open browser console and run: `localStorage.setItem('dg_unlocked', JSON.stringify(['easy','medium','hard','phase2']))`. Reload. Click Start. Click the **Miami** card. You should see a dark neon board with top-down view, pink divider line, grid, and cyan/pink units. Bullets travel at an angle. Console: no errors.

- [ ] **Step 5: Commit**

```bash
git add C:/Users/lenovo/dreamgame/index.html
git commit -m "feat: Phase 2 Hotline Miami top-down neon scene"
```

---

## Self-Review

### Spec coverage
| Requirement | Task |
|---|---|
| Green blinking typewriter on story screen | Task 1 |
| Start button fades in after typing | Task 1 |
| Level select screen (Easy/Medium/Hard/Phase2) | Task 2 |
| Lock/unlock via localStorage | Task 2, 3 |
| Medium difficulty config | Task 3 |
| Hard difficulty config | Task 3 |
| AI scales with difficulty | Task 3 |
| Win unlocks next level | Task 3 |
| Phase 2 top-down camera | Task 4 |
| Phase 2 neon Hotline Miami palette | Task 4 |
| Phase 2 diagonal bullets | Task 4 |
| Phase 2 separate HUD | Task 4 |

### Placeholder scan
No TBDs found. All code blocks are complete and self-contained.

### Type consistency
- `p2SpawnUnit` used in Phase 2 loop — defined in the same Task 4 block ✓
- `AI_RAMP_SECS` introduced in Task 3 Step 1 and consumed in Step 3 ✓
- `showLevelSelect` defined in Task 2 Step 3, called in Task 3 Step 6 overlay button ✓
- `currentLevel` declared in Task 2 Step 4, used in Task 3 Steps 5 and 6 ✓
