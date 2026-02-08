# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Operation Abandoned** ("Операция Заброшка") — a hardcore browser-based 3D FPS built as a single `index.html` file. Target hardware: Nvidia RTX 3000 8GB / 64GB RAM. Full spec in `ТЗ.md` (Russian).
* Используй модульный подход к разработке если сложные игры
* Используй подход разработка через тестирование
* Придерживайся правила не более 500 строк в файле если это возможно.
* Придерживайся прави

## Strict Rules

- **Output is ALWAYS a single `index.html` file** — all HTML, CSS, and JS inline, no external dependencies
- **No libraries/frameworks** — pure WebGL, Canvas API, Web Audio API, ES6+
- **Visual quality must be high** — particles, fog, neon glow, smooth 60+ FPS animations. Do not economize on GPU resources
- **Dark theme** with neon accents (green crosshair, red HP, cyan ammo)

## Architecture (single file, ~1300 lines)

The game is structured as sequential sections in one `<script>` block:

1. **CONFIG** — game constants (speeds, damage values, timers)
2. **MAP** — 22x17 grid level definition (0=empty, 1=wall, 2=crate, 3=enemy, 4=money, 5=car, 6=spawn)
3. **WebGL Setup** — context, shader compilation with error reporting to `document.title`
4. **3 Shader Programs** — world (textured+lit+fog), billboard (enemies as camera-facing sprites), particles (point sprites with additive blending)
5. **Math** — `perspective()`, `viewMatrix()`, `mat4Mul()` using column-major Float32Arrays
6. **Procedural Textures** — canvas-generated textures uploaded to WebGL (wall, floor, ceiling, crate, car, enemy, dead enemy)
7. **Level Geometry** — builds VBOs from MAP grid; `pushQuad()` emits 6 vertices (2 triangles) per face
8. **Collision** — circle-vs-AABB for player/enemies against walls; `lineOfSight()` via grid raymarching
9. **Sound** — Web Audio API procedural synthesis (no audio files)
10. **Particles** — muzzle flash, blood, impact sparks
11. **Game State** — states: `menu`, `playing`, `dead`, `win`
12. **Player Update** — FPS movement (WASD+mouse via Pointer Lock), jumping, shooting, reloading, bleeding
13. **Shooting** — raycast from camera; ray-sphere vs enemies, grid-step vs walls
14. **Enemy AI** — state machine: `idle` → `combat` → `hurt` → `dead`; line-of-sight triggers combat
15. **Render Pipeline** — `resetAttribs()` between shader programs to prevent attribute state leaking
16. **HUD** — 2D canvas weapon overlay, HTML minimap, health/ammo/objectives as DOM elements
17. **Main Loop** — `requestAnimationFrame` with dt capping at 0.05s; try-catch for error visibility

## Key Technical Gotchas

- **Vertex attribute state leaks between shader programs** — always call `resetAttribs()` (disables attribs 0-3) before switching programs, or WebGL will read out-of-bounds from wrong VBOs
- **Face winding order** — wall face quads currently have inconsistent winding; face culling is disabled globally as a workaround
- **Every HTML element referenced by `getElementById()` must exist in the DOM** — missing elements crash the script and prevent all subsequent event listener registration
- **Scene brightness** — ambient light factor in world vertex shader (`vL=0.55+0.45*...`) must stay above ~0.5 or the scene becomes invisible on most monitors
- **Procedural textures must be power-of-two dimensions** for WebGL1 mipmap/repeat compatibility (64, 128, etc.)

## Game Controls

WASD=move, Mouse=look, LMB=shoot, R=reload, Space=jump. Pointer Lock activates on canvas click.

## How to Test

Open `index.html` directly in Chrome. No build step, no server required.
