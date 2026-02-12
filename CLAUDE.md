# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Operation Abandoned** ("Операция Заброшка") — a hardcore browser-based 3D FPS built as a single `index.html` file (~6300 lines). Full spec in `ТЗ.md` (Russian). Target hardware: Nvidia RTX 3000 8GB / 64GB RAM.

## Strict Rules

- **Output is ALWAYS a single `index.html` file** — all HTML, CSS, and JS inline, no external dependencies, no modules
- **No libraries/frameworks** — pure WebGL, Canvas API, Web Audio API, ES6+
- **Visual quality must be high** — particles, fog, neon glow, smooth 60+ FPS. Do not economize on GPU resources
- **Dark theme** with neon accents (green crosshair, red HP, cyan ammo)

## How to Test

Open `index.html` directly in Chrome. No build step, no server required.

Syntax-check JS without running:
```
node -e "const fs=require('fs');const html=fs.readFileSync('index.html','utf8');const m=html.match(/<script>([\s\S]*)<\/script>/);if(m)try{new Function(m[1]);console.log('OK')}catch(e){console.error(e.message)}"
```

## Architecture (single `<script>` block)

### Section Order (match this when adding code)
1. **CONFIG** (~L302) — constants: `CELL`, `WALL_H`, `MOVE_SPEED`, `BULLET_DMG`, `ENEMY_DMG`, etc.
2. **MAP** (~L330) — 22x17 grid: 0=empty, 1=wall, 2=crate, 3=enemy, 4=money, 5=car, 6=spawn. `MAP_ORIGINAL` stores immutable copy for restart restoration.
3. **GL INIT** (~L355) — WebGL context, canvas setup, `resize()`, early `var _gfxRes` for resolution scaling, `_dom` cached DOM element references
4. **SHADERS** (~L452) — 4 shader programs compiled and linked:
   - `worldProg`: textured + lit + fog via `uFog` uniform (walls/floor/crates)
   - `billProg`: billboard sprites (unused legacy)
   - `partProg`: point sprites with additive blending (particles)
   - `entityProg`: 3D colored boxes with model matrix `uM`, `uFog`, `uAO` uniforms (enemies, allies, weapons)
5. **SHARED ENTITY HELPERS** (~L596) — `setupEntityDraw(vpMat)` and global `box(m,sx,sy,sz,r,g,b)` — shared by all entity rendering functions
6. **MATH** (~L614) — `perspective()`, `viewMatrix()`, `mat4Mul()`, `mat4T/S/RX/RY/RZ` (column-major Float32Arrays)
7. **PROCEDURAL TEXTURES** (~L643) — canvas-generated, power-of-two dimensions (64/128)
8. **LEVEL GEOMETRY** (~L741) — VBOs built from MAP grid; `pushQuad()` = 6 verts per face; `cubeVBO` unit cube
9. **COLLISION & RAGDOLL** (~L842) — `isWall()`, `collide()`, `lineOfSight()` (grid raymarching), 14-particle Verlet ragdoll with 21 constraints, `boneMat()`, `createRagdoll()`
10. **SOUND** (~L1023) — Web Audio procedural synthesis `playSound()`, SpeechSynthesis `playVoice()` with 6s cooldown
11. **PARTICLES** (~L1130) — `spawnParticles()`, `spawnDirectionalBlood()`, giblets, grenades, `updateParticles()`
12. **GAME STATE** (~L1362) — variables, `WEAPONS[]` array, graphics settings (`gfx*` vars, `effectiveFogDist`), `_aliveEnemies` per-frame cache, init functions:
    - `findOpenCells(cx,cz,minDist)` — shared helper for spawning entities on empty MAP cells
    - `initGame()` (~L1616), `initJuggernaut()`, `initMathMode()`, `initZombieMode()`, `initBossMode()`, `initBattleRoyale()`, `initWaveDefense()`, `initInfection()`
    - `spawnBarrels()`, `spawnDoors()`, `addXP()`
13. **PLAYER UPDATE** (~L2327) — `updatePlayer(dt)`: movement, collision, shooting, reload, F/G interaction, doors, objectives
14. **SHOOTING** (~L2725) — `playerShoot()`, `singleRaycast()` with head/body hitboxes, boss damage resistance, glitch weapon conversion, `explodeBarrel()`
15. **ENEMY AI** (~L3115) — `updateEnemies(dt)`: state machine `idle→combat→hurt→dead→blinded→captured`, boss charge AI, glitched enemy ally AI
16. **ALLY AI** (~L3334) — `updateAllies(dt)`: medics heal, defenders fight, dumb wanders; `allyShoot()`
17. **RENDER** (~L3697):
    - `drawWorld()` — walls, floor, ceiling, crates, cars
    - `drawEnemies()` — enemy/zombie/ally/boss bodies (~20 boxes each), zombie/boss-specific appearance
    - `drawParticles()`, `resetAttribs()`
    - `drawWeapon3D()` — 5 weapon models with local `boxW()` for gold skin support
    - `drawMinigun()`, `drawMenuCharacter()`
    - `drawGiblets()`, `drawBloodPools()`, `drawBloodDecals()`
    - `drawPlayerSkeleton()` — 3rd person skeleton player model (cheat mode)
    - `render()` (~L4851) — orchestrates all draw calls, inline rendering of grenades/barrels/doors/bombs/mines/cameras
18. **MINIMAP** (~L5035) — 2D canvas minimap with enemies, allies, barrels, doors, boss
19. **HUD** (~L5244) — `updateHUD()`: HP, ammo, objectives, scope overlay, bodycam overlay, mission prompts
20. **INPUT** (~L5775) — keydown/keyup/mouse/contextmenu event listeners
21. **GAME CONTROL** (~L5810) — `game` object: `start()`, `startJuggernaut()`, `startMath()`, `startZombie()`, `restart()`, `toMenu()`; shop/trade functions; cheat code system (`activateCheat()`)
22. **SCREEN EFFECTS** (~L6100) — `drawRain()`, `drawFlashlight()`
23. **MAIN LOOP** (~L6113) — `frame()`: computes `_aliveEnemies` → updatePlayer → updateEnemies → updateAllies → updateParticles → render → HUD

### Shared Infrastructure (added during refactoring)

**`setupEntityDraw(vpMat)`** (L596) — Sets up entity shader: useProgram, VP matrix, fog/AO uniforms, binds cubeVBO, enables vertex attribs. Call this instead of repeating 8 lines of GL setup.

**`box(m, sx,sy,sz, r,g,b)`** (L607) — Global entity box renderer. Uses entity shader uniforms. `drawWeapon3D()` has a local `boxW()` override that adds gold skin color remapping.

**`findOpenCells(cx, cz, minDist)`** (L1604) — Returns array of `{x,z}` world positions for empty MAP cells at least `minDist` from center point. Used by all wave spawn and init functions.

**`_dom`** (L372) — Cached DOM element references (~35 elements). Use `_dom.hpFill`, `_dom.objTxt`, etc. instead of `document.getElementById()` in per-frame code.

**`_aliveEnemies`** (L1411) — Per-frame cached count of living enemies. Computed once at start of `frame()`. Use instead of `enemies.filter(e=>e.hp>0).length`.

**`effectiveFogDist`** (L1417) — Actual fog distance used in shaders. Weather system modifies this, not `gfxFogDist` (which stores user's setting). Synced on preset change via `resizeGfx()`.

**`MAP_ORIGINAL`** (L352) — Deep copy of initial MAP state. `initGame()` restores MAP from this on every restart to undo door/editor mutations.

### Entity Rendering Pattern (entityProg)
All 3D characters (enemies, allies, first-person weapon) use the same unit cube VBO with `box(matrix, sx,sy,sz, r,g,b)`. Body parts are composed via matrix multiplication chains:
```
baseM = T(pos) * RY(-angle)
limb  = baseM * T(pivot) * RX(swing) * T(offset) * S(size)
```

### Weapon System
`WEAPONS[]` array (index 0-4): assault rifle, shotgun, pistol, sniper, glitch gun. Each has: `name`, `dmg`, `mag`, `reserve`, `firerate`, `reload`, `spread`. Glitch gun (index 4) has `glitch:true` flag — converts killed enemies into allies. `currentWeapon` selects active weapon. `player._weaponAmmo[]` / `player._weaponReserve[]` persist ammo across switches (5 slots). `drawWeapon3D()` renders unique 3D models per weapon.

### Enemy States
- `idle` — patrols, transitions to `combat` on line-of-sight
- `combat` — faces player, shoots, moves toward/strafes
- `hurt` — brief stun after taking damage, stores `_prevState` to return to correct state
- `dead` — Verlet ragdoll (14 particles, 21 constraints), pushable body physics
- `blinded` — bag on head (G key), wanders randomly, shoots randomly
- `captured` — handcuffed (G key), follows player as shield

### Game Modes
`gameMode`: `'normal'` | `'juggernaut'` | `'math'` | `'zombie'` | `'boss'` | `'battleroyale'` | `'wavedefense'` | `'infection'`

### Graphics Settings System
4 presets (low/medium/high/ultra) with individual slider overrides. `gfxFogDist` is the user's setting; `effectiveFogDist` is what shaders actually use (weather-modified). Key functions: `applyGfxPreset()`, `resizeGfx()`, `applyGfxVisuals()`, `updateGfxUI()`

### Cheat Code System
`activateCheat(code)` — 4 codes entered in settings screen input field (case-insensitive):
- `BLOOD A NOT 5 YERS OLD A 18 YERS OLD` — unlock gore
- `free pls 150` — +150 coins
- `skelet with black gun plssss` — skeleton mode + 3rd person camera
- `black world spawn` — teleport to boss arena fight

### Gore System
Hidden behind cheat code. Directional blood spray, blood→decal conversion, persistent floor/wall decals, blood pools under dead bodies, giblets (tumbling chunks with Verlet), screen blood droplets, dismemberment flags (`headGone`/`armLGone`/etc.), screen shake.

## Key Technical Gotchas

- **mat4RY sign convention**: `mat4RY(a)` maps +Z to `(-sin(a), 0, cos(a))`. For entities using `atan2(dx,dz)`, always negate: `mat4RY(-angle)`
- **Vertex attrib state leaks** — call `resetAttribs()` (disables attribs 0-3) before switching shader programs
- **Use `setupEntityDraw(vpMat)`** instead of manually repeating GL setup for entity rendering
- **Use `_dom.xxx`** instead of `document.getElementById()` in per-frame code paths
- **Use `_aliveEnemies`** instead of `enemies.filter(e=>e.hp>0).length` — already cached per frame
- **Use `effectiveFogDist`** in shader uniforms, not `gfxFogDist` — weather modifies effective, user setting stays in gfxFogDist
- **Every `getElementById()` target must exist in DOM** — missing elements crash the entire script
- **Scene brightness** — ambient factor in world vertex shader must stay >= 0.55
- **Procedural textures** — must be power-of-two for WebGL1 mipmap/repeat
- **`_prevState` for enemy state recovery** — when setting enemy state to `hurt`, save `_prevState` first so `blinded`/`captured` enemies return to correct state after hurt recovery
- **Dead body `_fallSide`/`_limbPose`** — generated once on death via `undefined` check, don't reset
- **boneMat unit cube** — cube is -0.5..0.5, so Y-column scale must be `len` (not len/2)
- **boneMat column-major** — WebGL expects `[rx,ry,rz,0, ux,uy,uz,0, fx,fy,fz,0, tx,ty,tz,1]`
- **Bullet time** — `gameDt=dt*0.3` for enemies/physics, player always uses real `dt`
- **Zombie flag** — enemies with `e._zombie=true` render differently (green skin, no weapon, reaching pose) and use melee instead of shooting
- **TDZ with `gfxResScale`** — `resize()` is called at ~L365, before `let gfxResScale` at ~L1413. Use `var _gfxRes=1.0` (declared at ~L358) in `resize()`, synced via `resizeGfx()`
- **Fog/AO uniforms** — handled by `setupEntityDraw()`. For `worldProg`, set `uFog`/`uAO` manually.
- **Boss damage resistance** — in `singleRaycast()`: body damage ×0.5, headshot capped at 150, hurt timer 0.1s
- **Glitched enemies** — `e._glitched=true` + `e._isAlly=true` makes them fight other enemies; must check `_glitched` in enemy AI to avoid targeting allies
- **MAP restoration** — `initGame()` restores MAP from `MAP_ORIGINAL`. Always mutate MAP (doors/editor) knowing it will be reset.
- **Inline particles must include `grav` and `maxLife`** — `updateParticles()` does `p.vy -= p.grav * dt`; missing `grav` causes NaN propagation

## Controls

| Key | Action |
|-----|--------|
| WASD | Move |
| Mouse | Look (Pointer Lock) |
| LMB | Shoot |
| RMB | Scope/ADS/Shield |
| R | Reload |
| Space | Jump |
| Ctrl | Crouch toggle |
| Shift | Sprint |
| Q/E | Lean left/right |
| F | Interact (loot, heal ally, doors, defuse) |
| G | Capture (bag then cuffs on enemy) |
| H | Use medkit |
| 1-6 | Weapon switch (rifle/shotgun/pistol/sniper/glitch/gravgun) |
| T | Throw grenade |
| Z | Bullet time |
| N | Night vision toggle |
| L | Flashlight toggle |
| V | Drone toggle (fly with WASD) |
| C | Place mine |
| B | Stealth kill (behind enemy) |
| M | Akimbo toggle (dual pistols) |
| X (hold) | Command wheel for allies |
| Tab | Camera view toggle |
| Shift+Ctrl | Slide (while sprinting) |

## Persistent State (survives between missions)
`playerCoins`, `playerMedkits`, `weaponBonus`, `armorBonus`, `lootCount`, `playerBags`, `playerCuffs`, `jugUnlocked`, `playerXP`, `playerLevel`, `hasShotgun`, `hasPistol`, `hasSniper`, `hasShield`, `hasBulletTime`, `hasNightVision`, `hasFlashlight`, `goldSkin`, `hasGlitchWeapon`, `bossDefeated`, `skeletonMode`, `thirdPersonCam`, `goreEnabled`, `hasAkimbo`, `hasEquipHelmet`, `hasEquipVest`, `hasEquipKnees`, `playerMines`

Reset per mission in `initGame()`: `capturedEnemy`, `evacTimer`, `particles`, `helmetCracks`, `allies`, `enemies`, `barrels`, `doors`, `grenades`, MAP (restored from `MAP_ORIGINAL`)
