# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Operation Abandoned** ("Операция Заброшка") — a hardcore browser-based 3D FPS built as a single `index.html` file (~6350 lines). Full spec in `ТЗ.md` (Russian). Target hardware: Nvidia RTX 3000 8GB / 64GB RAM.

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

## Architecture (single `<script>` block, ~6050 lines JS)

### Section Order (match this when adding code)
1. **CONFIG** (~L304) — constants: `CELL`, `WALL_H`, `MOVE_SPEED`, `BULLET_DMG`, `ENEMY_DMG`, etc.
2. **MAP** (~L332) — 22x17 grid: 0=empty, 1=wall, 2=crate, 3=enemy, 4=money, 5=car, 6=spawn
3. **GL INIT** (~L357) — WebGL context, canvas setup, `resize()`, early `var _gfxRes` for resolution scaling
4. **SHADERS** (~L376) — 4 shader programs compiled and linked:
   - `worldProg` (L532): textured + lit + fog via `uFog` uniform (walls/floor/crates)
   - `billProg` (L538): billboard sprites (unused legacy)
   - `partProg` (L543): point sprites with additive blending (particles)
   - `entityProg` (L549): 3D colored boxes with model matrix `uM`, `uFog`, `uAO` uniforms (enemies, allies, weapons)
5. **MATH** (~L557) — `perspective()`, `viewMatrix()`, `mat4Mul()`, `mat4T/S/RX/RY/RZ` (column-major Float32Arrays)
6. **PROCEDURAL TEXTURES** (~L586) — canvas-generated, power-of-two dimensions (64/128)
7. **LEVEL GEOMETRY** (~L686) — VBOs built from MAP grid; `pushQuad()` = 6 verts per face; `cubeVBO` unit cube (L758)
8. **COLLISION & RAGDOLL** (~L785) — `isWall()`, `collide()`, `lineOfSight()` (grid raymarching), 14-particle Verlet ragdoll with 21 constraints, `boneMat()` (L941), `createRagdoll()` (L876)
9. **SOUND** (~L969) — Web Audio procedural synthesis `playSound()`, SpeechSynthesis `playVoice()` (L1058) with 6s cooldown
10. **PARTICLES** (~L1075) — `spawnParticles()`, `spawnDirectionalBlood()` (L1100), giblets, grenades, `updateParticles()`
11. **GAME STATE** (~L1300) — variables, `WEAPONS[]` array (L1327), graphics settings (`gfx*` vars L1353), init functions:
    - `initGame()` (L1540), `initJuggernaut()` (L1584), `initMathMode()` (L1635), `initZombieMode()` (L1691), `initBossMode()` (L1731)
    - `initBattleRoyale()` (L1796), `initWaveDefense()` (L1822), `initInfection()` (L1854)
    - `spawnBarrels()` (L1753), `spawnDoors()` (L1765), `addXP()` (L1787)
12. **MAP EDITOR** (~L2225) — `drawEditorMap()`, editor tool handling
13. **PLAYER UPDATE** (~L2277) — `updatePlayer(dt)`: movement, collision, shooting, reload, F/G interaction, doors, objectives
14. **SHOOTING** (~L2675) — `playerShoot()`, `singleRaycast()` (L2706) with head/body hitboxes, boss damage resistance, glitch weapon conversion, `explodeBarrel()` (L3006)
15. **ENEMY AI** (~L3065) — `updateEnemies(dt)`: state machine `idle→combat→hurt→dead→blinded→captured`, boss charge AI, glitched enemy ally AI
16. **ALLY AI** (~L3284) — `updateAllies(dt)`: medics heal, defenders fight, dumb wanders; `allyShoot()` (L3493)
17. **RENDER** (~L3615):
    - `drawHelmetCracks()` (L3615)
    - `drawWorld()` (L3647) — walls, floor, ceiling, crates, cars
    - `drawEnemies()` (L3675) — enemy/zombie/ally/boss bodies (~20 boxes each), zombie/boss-specific appearance
    - `drawParticles()` (L4093), `resetAttribs()` (L4124)
    - `drawWeapon3D()` (L4130) — 5 weapon models with local `boxW()` for gold skin support
    - `drawMinigun()` (L4514), `drawMenuCharacter()` (L4583)
    - `drawGiblets()` (L4650), `drawBloodPools()` (L4680), `drawBloodDecals()` (L4713)
    - `drawPlayerSkeleton()` (L4766) — 3rd person skeleton player model (cheat mode)
    - `render()` (L4871) — orchestrates all draw calls, inline rendering of grenades/barrels/doors/bombs/mines/cameras
18. **MINIMAP** (~L5091) — `drawMinimap()`: 2D canvas minimap with enemies, allies, barrels, doors, boss
19. **HUD** (~L5300) — `updateHUD()`: HP, ammo, objectives, scope overlay, bodycam overlay, mission prompts; `drawScreenBlood()` (L5254)
20. **INPUT** (~L5730) — keydown/keyup/mouse/contextmenu event listeners
21. **GAME CONTROL** (~L5756) — `game` object: `start()`, `startJuggernaut()`, `startMath()`, `startZombie()`, `restart()`, `toMenu()`; shop/trade functions; cheat code system `activateCheat()` (L6016)
22. **SCREEN EFFECTS** (~L6101) — `drawRain()`, `drawFlashlight()` (L6142)
23. **MAIN LOOP** (~L6175) — `frame()`: updatePlayer → updateEnemies → updateAllies → updateParticles → render → HUD

### Entity Rendering Pattern (entityProg)
All 3D characters (enemies, allies, first-person weapon) use the same unit cube VBO. Each draw function (drawEnemies, drawWeapon3D, etc.) defines a local `box(m, sx,sy,sz, r,g,b)` helper that sets `uM` uniform and draws the cube. `drawWeapon3D()` has a local `boxW()` override that adds gold skin color remapping. Body parts are composed via matrix multiplication chains:
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
4 presets (low/medium/high/ultra) with individual slider overrides. Key functions: `applyGfxPreset()` (L1364), `resizeGfx()` (L1387), `applyGfxVisuals()` (L1397), `updateGfxUI()` (L1407)

### Cheat Code System
`activateCheat(code)` (L6016) — 4 codes entered in settings screen input field (case-insensitive):
- `BLOOD A NOT 5 YERS OLD A 18 YERS OLD` — unlock gore
- `free pls 150` — +150 coins
- `skelet with black gun plssss` — skeleton mode + 3rd person camera
- `black world spawn` — teleport to boss arena fight

### Gore System
Hidden behind cheat code. Directional blood spray, blood→decal conversion, persistent floor/wall decals, blood pools under dead bodies, giblets (tumbling chunks with Verlet), screen blood droplets, dismemberment flags (`headGone`/`armLGone`/etc.), screen shake.

## Key Technical Gotchas

- **mat4RY sign convention**: `mat4RY(a)` maps +Z to `(-sin(a), 0, cos(a))`. For entities using `atan2(dx,dz)`, always negate: `mat4RY(-angle)`
- **Vertex attrib state leaks** — call `resetAttribs()` (L4124, disables attribs 0-3) before switching shader programs
- **Every `getElementById()` target must exist in DOM** — missing elements crash the entire script (was root cause of black screen bug)
- **Entity shader setup boilerplate** — each draw function (drawEnemies, drawWeapon3D, etc.) manually sets up entityProg: `useProgram`, VP matrix, fog/AO uniforms, binds cubeVBO, enables attribs. Keep this pattern consistent.
- **Scene brightness** — ambient factor in world vertex shader must stay >= 0.55
- **Procedural textures** — must be power-of-two for WebGL1 mipmap/repeat
- **`_prevState` for enemy state recovery** — when setting enemy state to `hurt`, save `_prevState` first so `blinded`/`captured` enemies return to correct state after hurt recovery
- **Dead body `_fallSide`/`_limbPose`** — generated once on death via `undefined` check, don't reset
- **boneMat unit cube** — cube is -0.5..0.5, so Y-column scale must be `len` (not len/2)
- **boneMat column-major** — WebGL expects `[rx,ry,rz,0, ux,uy,uz,0, fx,fy,fz,0, tx,ty,tz,1]`
- **Bullet time** — `gameDt=dt*0.3` for enemies/physics, player always uses real `dt`
- **Zombie flag** — enemies with `e._zombie=true` render differently (green skin, no weapon, reaching pose) and use melee instead of shooting
- **TDZ with `gfxResScale`** — `resize()` is called at ~L362, before `let gfxResScale` at ~L1353. Use `var _gfxRes=1.0` (declared at ~L357) in `resize()`, synced via `resizeGfx()`
- **Fog/AO uniforms** — `uFog` and `uAO` must be set every time `worldProg` or `entityProg` is used. Missing uniform = stale value from previous draw call
- **Boss damage resistance** — in `singleRaycast()`: body damage ×0.5, headshot capped at 150, hurt timer 0.1s
- **Glitched enemies** — `e._glitched=true` + `e._isAlly=true` makes them fight other enemies; must check `_glitched` in enemy AI to avoid targeting allies
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

Reset per mission in `initGame()`: `capturedEnemy`, `evacTimer`, `particles`, `helmetCracks`, `allies`, `enemies`, `barrels`, `doors`, `grenades`
