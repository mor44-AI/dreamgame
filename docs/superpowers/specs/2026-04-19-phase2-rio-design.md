# Phase 2 — Rio Design Spec

## Overview

Replace the current Hotline Miami top-down scene with **Rio**: a 2D canvas top-down maze shooter. The player navigates a 20-room maze under fog of war, avoids or bypasses 10 patrolling enemies, and wins by reaching the ✦ exit.

---

## Integration

- Triggered from the level-select screen when the player clicks the **Rio** card (was "Miami").
- `startPhase2()` hides the Phase 1 canvas and HUD, creates a new `<canvas>` element for the maze, and starts the Rio game loop.
- The existing `#overlay` div is reused for game-over and win screens.
  - Death → overlay shows **"GAME OVER"** + Restart button → calls `startPhase2()` again (full reset).
  - Exit reached → overlay shows **"YOU WON"** + Return to Menu button → shows level-select.
- Everything stays inside the single `index.html` file.
- Level-select card text and HUD label updated from "Miami" / "MIAMI" to "Rio" / "RIO".

---

## Map

**Layout:** v7 — 20 rooms across 4 columns, canvas 510×444px, scaled to fill the browser window via CSS (`width:100%; height:100%` on the canvas wrapper).

**Column positions:**
| Column | x   | width | right |
|--------|-----|-------|-------|
| Col 1  | 8   | 108   | 116   |
| Col 2  | 140 | 90    | 230   |
| Col 3  | 254 | 105   | 359   |
| Col 4  | 383 | 119   | 502   |

**Rooms (id, x, y, w, h):**
```
R0  START   8,8     108,96
R1          8,130   108,76
R2          8,206   108,84
R3          8,316   108,64
R4  ✕       8,380   108,56
R5          140,8    90,60
R6          140,68   90,76
R7          140,170  90,84
R8          140,254  90,72
R9  ✕       140,352  90,84
R10         254,8    105,88
R11         254,122  105,68
R12         254,190  105,88
R13         254,304  105,68
R14 ✕       254,372  105,64
R15         383,8    119,100
R16         383,108  119,72
R17         383,206  119,76
R18         383,282  119,74
R19 ✦ EXIT  383,382  119,54
```

**Corridors (x, y, w, h — V corridors have h>w, H corridors have w>h):**
```
V R0→R1    30,104   22,26
V R2→R3    30,290   22,26
V R6→R7    162,144  22,26
V R8→R9    162,326  22,26
V R10→R11  276,96   22,26
V R12→R13  276,278  22,26
V R16→R17  404,180  22,26
V R18→R19  404,356  22,26
H R0→R5    116,22   24,20
H R6→R10   230,72   24,20
H R8→R12   230,256  24,20
H R10→R15  359,22   24,20
H R13→R18  359,316  24,20
```

**Shared walls (y, x, w, room_a, room_b — doorway cut through touching wall):**
```
R1↔R2   y=206  x=30  w=22
R3↔R4   y=380  x=30  w=22
R5↔R6   y=68   x=162 w=22
R7↔R8   y=254  x=162 w=22
R11↔R12 y=190  x=276 w=22
R13↔R14 y=372  x=276 w=22
R15↔R16 y=108  x=404 w=22
R17↔R18 y=282  x=404 w=22
```

**Graph:**
- Main path to exit: R0 → R5 → R6 → R10 → R15 → R16 → R17 → R18 → R19
- Dead-end branch 1: R0 ↔ R1 ↔ R2 ↔ R3 ↔ R4 ✕
- Dead-end branch 2: R6 ↔ R7 ↔ R8 ↔ R9 ✕
- Dead-end branch 3: R10 ↔ R11 ↔ R12 ↔ R13 ↔ R14 ✕

---

## Player

- **Appearance:** cyan circle, radius 6px; short direction line (length 11px) showing facing angle.
- **Start position:** center of R0.
- **Movement:** WASD at 120 px/s (canvas coordinates). Diagonal movement normalized to 120 px/s.
- **Aiming:** facing angle = `atan2(mouse.y - player.y, mouse.x - player.x)` in canvas space. Movement direction is independent of facing.
- **Shooting:** left-click fires one bullet in the facing direction. Bullets travel at 400 px/s. Unlimited ammo.
- **Wall collision:** per-axis sliding — test X and Y movement separately; cancel only the axis that causes penetration.
- **Death:** one bullet hit → game over.

---

## Bullets

- **Player bullet:** cyan (`#00ccff`), 4px radius circle.
- **Enemy bullet:** red (`#ff3c3c`), 4px radius circle.
- Bullets are destroyed on hitting a wall or the opposite-side target.
- Max range: 600px (destroyed if they travel that far without hitting anything).

---

## Enemies

**Count:** 10, distributed across rooms on both the main path and dead ends. Starting positions:

| Enemy | Room | rx  | ry  | patrol axis | facing |
|-------|------|-----|-----|-------------|--------|
| E0    | R0   | 0.7 | 0.5 | H           | right  |
| E1    | R5   | 0.5 | 0.5 | H           | left   |
| E2    | R6   | 0.5 | 0.5 | V           | down   |
| E3    | R7   | 0.5 | 0.5 | H           | right  |
| E4    | R10  | 0.5 | 0.5 | H           | left   |
| E5    | R12  | 0.5 | 0.5 | V           | up     |
| E6    | R13  | 0.5 | 0.5 | H           | right  |
| E7    | R15  | 0.5 | 0.5 | H           | left   |
| E8    | R17  | 0.5 | 0.5 | V           | down   |
| E9    | R18  | 0.5 | 0.5 | H           | right  |

**Patrol:** each enemy moves back and forth along one axis within their room at 40 px/s. On hitting a wall they reverse.

**Field of view:** 30° cone, range 150px. Line-of-sight check: sample 5 points along the cone centre ray; if any point falls outside the current room (i.e. inside a wall), sight is blocked.

**Detection:** player is in FOV cone AND line-of-sight clear → enemy detects player.
- On detection: enemy stops patrolling, faces player, fires one bullet per second toward player's current position.
- If player out of sight for 3 continuous seconds: enemy resumes patrol from current position.

**Death:** one player bullet hit → enemy removed from scene, no respawn.

---

## Fog of War

- A `visitedRooms` Set tracks which room IDs the player has entered.
- A room is added to `visitedRooms` when the player's centre point is inside that room's rect.
- A corridor is visible if either connecting room is visited.
- Rendering: only visited rooms and their connecting corridors are drawn. Unvisited areas are simply not drawn (dark background shows through).
- Enemies in unvisited rooms are not drawn and do not shoot.

---

## HUD & Mini-map

**Top bar** (fixed, above canvas): `RIO` label left, `find the exit` label right. Monospace font, dim colour. No health indicator.

**Mini-map** (top-right corner of canvas, 20px padding):
- 20 small squares (6×6px, 2px gap) in a 5×4 grid.
- Unvisited: `#111`
- Visited: `#1a1a5a`
- Current room: `#00ccff`
- Exit room (once visited): `#003311`
- Player dot: 3×3px cyan square overlaid on current room square.

---

## Win / Lose

**Win:** player's centre enters R19 (✦ EXIT) → game loop stops → overlay shows **"YOU WON 🎉"** with a "Choose Level" button (returns to level-select).

**Lose:** player hit by any enemy bullet → game loop stops → overlay shows **"GAME OVER 💀"** with a "Try Again" button (calls `startPhase2()`) and a "Choose Level" button.

---

## Colour Palette

| Element          | Colour      |
|------------------|-------------|
| Background       | `#07070f`   |
| Room floor       | `#10102a`   |
| Dead-end floor   | `#180a0a`   |
| Exit floor       | `#071a09`   |
| Room walls       | `#222248`   |
| Dead-end walls   | `#3a1010`   |
| Exit walls       | `#006633`   |
| Corridor floor   | `#0d0d20`   |
| Corridor walls   | `#181830`   |
| Player           | `#00ccff`   |
| Player bullet    | `#00ccff`   |
| Enemy            | `#ff3c3c`   |
| Enemy bullet     | `#ff3c3c`   |
| Enemy FOV fill   | `rgba(255,50,50,0.14)` |
| Exit label       | `#00cc55`   |
| Dead-end label   | `#773333`   |
