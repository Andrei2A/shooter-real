# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Operation Abandoned** ("Операция Заброшка") — a hardcore browser-based 3D FPS built as a single `index.html` file. Full spec in `ТЗ.md` (Russian). Target hardware: Nvidia RTX 3000 8GB / 64GB RAM.

## Strict Rules

- **Output is ALWAYS a single `index.html` file** — all HTML, CSS, and JS inline, no external dependencies
- **No libraries/frameworks** — pure WebGL, Canvas API, Web Audio API, ES6+
- **Visual quality must be high** — particles, fog, neon glow, smooth 60+ FPS. Do not economize on GPU resources
- **Dark theme** with neon accents (green crosshair, red HP, cyan ammo)

## How to Test

Open `index.html` directly in Chrome. No build step, no server required.

Syntax-check JS without running: `node -e "const fs=require('fs');const html=fs.readFileSync('index.html','utf8');const m=html.match(/<script>([\s\S]*)<\/script>/);if(m)try{new Function(m[1]);console.log('OK')}catch(e){console.error(e.message)}"`

## Architecture (~2950 lines, single `<script>` block)

### Section Order (match this when adding code)
1. **CONFIG** (~L4) — constants: `CELL`, `WALL_H`, `MOVE_SPEED`, `BULLET_DMG`, `ENEMY_DMG`, etc.
2. **MAP** (~L31) — 22x17 grid: 0=empty, 1=wall, 2=crate, 3=enemy, 4=money, 5=car, 6=spawn
3. **GL INIT** (~L56) — WebGL context, canvas setup
4. **SHADERS** (~L111) — 4 shader programs compiled and linked:
   - `worldProg`: textured + lit + fog (walls/floor/crates)
   - `billProg`: billboard sprites (unused legacy, kept for particles)
   - `partProg`: point sprites with additive blending (particles)
   - `entityProg`: 3D colored boxes with model matrix `uM` (enemies, allies, weapons)
5. **MATH** (~L250) — `perspective()`, `viewMatrix()`, `mat4Mul()`, `mat4T/S/RX/RY/RZ` (column-major Float32Arrays)
6. **PROCEDURAL TEXTURES** (~L279) — canvas-generated, power-of-two dimensions (64/128)
7. **LEVEL GEOMETRY** (~L377) — VBOs built from MAP grid; `pushQuad()` = 6 verts per face
8. **COLLISION** (~L472) — `isWall()`, `collide()` (circle-vs-AABB), `lineOfSight()` (grid raymarching)
9. **SOUND** (~L516) — Web Audio procedural synthesis, `playSound()`, `playVoice()` (SpeechSynthesis)
10. **PARTICLES** (~L599) — `spawnParticles()` + `updateParticles()`: muzzle, blood, impact
11. **GAME STATE** (~L634) — variables, `initGame()`, `initJuggernaut()`, `spawnJugWave()`
12. **PLAYER UPDATE** (~L765) — `updatePlayer(dt)`: movement, collision, shooting, reload, F/G interaction, objectives
13. **SHOOTING** (~L1044) — `playerShoot()`: raycast with head/body hitboxes
14. **ENEMY AI** (~L1148) — `updateEnemies(dt)`: state machine `idle→combat→hurt→dead→blinded→captured`
15. **ALLY AI** (~L1298) — `updateAllies(dt)`: medics heal, defenders fight, dumb wanders
16. **RENDER** (~L1617) — `drawWorld()`, `drawEnemies()`, `drawParticles()`, `drawWeapon3D()`, `render()`
17. **MINIMAP** (~L2283) — 2D canvas minimap
18. **HUD** (~L2353) — `updateHUD()`: HP, ammo, objectives, prompts, scope
19. **INPUT** (~L2592) — keydown/keyup/mouse event listeners
20. **GAME CONTROL** (~L2616) — `game` object: `start()`, `restart()`, `toMenu()`, shop/trade functions
21. **MAIN LOOP** (~L2738) — `frame()`: updatePlayer→updateEnemies→updateAllies→render→HUD

### Entity Rendering Pattern (entityProg)
All 3D characters (enemies, allies, first-person weapon) use the same unit cube VBO with a `box(matrix, sx,sy,sz, r,g,b)` helper. Body parts are composed via matrix multiplication chains:
```
baseM = T(pos) * RY(-angle)
limb  = baseM * T(pivot) * RX(swing) * T(offset) * S(size)
```

### Enemy States
- `idle` — patrols, transitions to `combat` on line-of-sight
- `combat` — faces player, shoots, moves toward/strafes
- `hurt` — brief stun after taking damage, stores `_prevState` to return to correct state
- `dead` — ragdoll on ground with randomized limb poses (`_limbPose`), pushable body physics
- `blinded` — bag on head (G key), wanders randomly, shoots randomly (can hit other enemies)
- `captured` — handcuffed (G key), follows player as shield, allies ignore

### Game Modes
- `'normal'` — standard mission: kill enemies, collect money, evacuate
- `'juggernaut'` — endless waves, 500HP, minigun, unlocked after first win

## Key Technical Gotchas

- **mat4RY sign convention**: `mat4RY(a)` maps +Z to `(-sin(a), 0, cos(a))`. For entities using `atan2(dx,dz)`, always negate: `mat4RY(-angle)`
- **Vertex attrib state leaks** — call `resetAttribs()` (disables attribs 0-3) before switching shader programs
- **Every `getElementById()` target must exist in DOM** — missing elements crash the entire script
- **Scene brightness** — ambient factor in world vertex shader must stay ≥ 0.55
- **Procedural textures** — must be power-of-two for WebGL1 mipmap/repeat
- **`_prevState` for enemy state recovery** — when setting enemy state to `hurt`, save `_prevState` first so `blinded`/`captured` enemies return to correct state after hurt recovery
- **Dead body `_fallSide`/`_limbPose`** — generated once on death via `undefined` check, don't reset

## Controls

| Key | Action |
|-----|--------|
| WASD | Move |
| Mouse | Look (Pointer Lock) |
| LMB | Shoot |
| RMB | Scope/ADS |
| R | Reload |
| Space | Jump |
| Ctrl | Crouch toggle |
| Shift | Sprint |
| Q/E | Lean left/right |
| F | Interact (loot, heal/revive ally) |
| G | Capture (bag on enemy, then handcuffs) |
| H | Use medkit |

## Persistent State (survives between missions)
`playerCoins`, `playerMedkits`, `weaponBonus`, `armorBonus`, `lootCount`, `playerBags`, `playerCuffs`, `jugUnlocked`

Reset per mission in `initGame()`: `capturedEnemy`, `evacTimer`, `particles`, `helmetCracks`, `allies`, `enemies`
