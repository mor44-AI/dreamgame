# Dream Game — Design Spec
**Date:** 2026-04-10
**Status:** Approved

---

## Overview

A browser-based 2.5D isometric arcade game where two sides — blue (player) and red (AI opponent) — face each other across a vertical board. Constant streams of bullets fly from each wall toward the opposite side. Players block bullets; unblocked bullets reward the firing side with an extra unit. The first side to pack their half so densely that no bullet can reach their wall wins.

---

## Board

- Rectangular field rendered in 2.5D isometric perspective using Three.js with a fixed orthographic camera (45° yaw, ~35° pitch). Camera never moves.
- Divided horizontally: **red zone (top half)**, **blue zone (bottom half)**.
- A dashed center line marks the boundary — neither side's units can cross it.
- Top and bottom edges are solid walls (scoring walls).
- Visual style matches the dream image: light grey/white rounded rectangle board on a neutral background, subtle drop shadow.

---

## Bullets

- Small glowing cubes (~0.2 units).
- **Red bullets** spawn at random X positions along the top wall and travel straight **down**.
- **Blue bullets** spawn at random X positions along the bottom wall and travel straight **up**.
- Spawn rate: constant and rapid (~3–5 bullets/second per side), slightly increasing over time.
- Bullets from opposite sides **pass through each other** — no cross-collision.
- **Hit a unit from the opposing side** → bullet is destroyed (flash effect at impact point).
- **Reach the far wall without hitting any unit** → the side that fired that bullet gains **+1 unit**, spawned at a random position within that side's half.

---

## Units (Players)

- Boxy 3D humanoid meshes: box head, box torso, box arms, box legs — matching the stick-figure style of the dream image.
- Blue tint for player side; red tint for opponent side.
- Subtle idle bobbing animation.
- Each unit has a **circular collision radius** (~0.6 units). Any bullet entering this radius is destroyed.
- Units cannot leave their own half of the board.
- Units repel each other with a soft spring force to prevent stacking.
- New units spawn at a random position within their half.
- Starting count: **3 blue, 3 red**.

---

## Controls — Cursor Magnet

- Mouse position is projected onto the board plane (blue half only).
- Each blue unit accelerates toward the cursor with **springy drag** (not instant — feels fluid and alive).
- Units also apply **mutual repulsion** so they spread out naturally rather than piling up.
- As blue's unit count grows, the crowd becomes harder to fully direct — intentional chaos that makes the late game feel overwhelming.
- Cursor stays constrained to the blue half visually (no effect if hovered over red half).

---

## AI Opponent (Red)

- Each red unit independently tracks the **nearest incoming blue bullet's X position** and drifts toward it.
- Movement has a **capped speed and reaction delay** so the AI is beatable at the start.
- AI speed and reaction time improve slightly as the game progresses (escalating pressure).
- Red units also repel each other to distribute coverage naturally across the top half.

---

## Win / Lose Condition

- **Coverage** is calculated per side as: `(sum of all unit collision areas) / board_width`.
- When coverage reaches **100%** (every column of the board is covered by at least one unit's hitbox), that side's wall becomes impenetrable.
- The game ends when one side reaches full coverage:
  - **Blue wins** if blue reaches 100% first.
  - **Red wins** if red reaches 100% first.
- A **coverage HUD bar** for each side is displayed at the top of the screen so the player can track progress.
- Player/unit count is shown per side.

---

## Visuals & Effects

| Element | Style |
|---|---|
| Camera | Fixed orthographic isometric (Three.js OrthographicCamera) |
| Board | Light grey rounded rect, subtle border + shadow |
| Units | Boxy humanoid meshes, tinted blue/red, idle bob |
| Bullets | Small glowing cubes with PointLight (color matches side) |
| Hit effect | Quick particle burst / flash at collision point |
| New unit spawn | Brief scale-up "pop" animation |
| HUD | Two coverage bars + unit counts, minimal, top of screen |

---

## Technical Delivery

- **Single `index.html` file** — no build step, no backend.
- Three.js loaded from CDN (`https://unpkg.com/three`).
- Open directly in any modern browser (or served via `npx serve`).
- Estimated ~400–600 lines of HTML + JS.
- Game loop via `requestAnimationFrame`.
- All collision detection in 2D (X/Z plane), projected into 3D for rendering.

---

## File Layout

```
C:\Users\lenovo\dreamgame\
├── index.html          ← the entire game
├── .claude\
│   └── launch.json     ← dev server configs
└── docs\
    └── superpowers\
        └── specs\
            └── 2026-04-10-dream-game-design.md  ← this file
```
