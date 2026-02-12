# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Operation Abandoned** ("Операция Заброшка") — a hardcore browser-based 3D FPS built as a single `index.html` file (~6290 lines). Full spec in `ТЗ.md` (Russian). Target hardware: Nvidia RTX 3000 8GB / 64GB RAM.

## Strict Rules

- **Output is ALWAYS a single `index.html` file** — all HTML, CSS, and JS inline, no external dependencies
- **No libraries/frameworks** — pure WebGL, Canvas API, Web Audio API, ES6+
- **Visual quality must be high** — particles, fog, neon glow, smooth 60+ FPS. Do not economize on GPU resources
- **Dark theme** with neon accents (green crosshair, red HP, cyan ammo)

## How to Test

Open `index.html` directly in Chrome. No build step, no server required.

Syntax-check JS without running:
```
node -e "const fs=require('fs');const html=fs.readFileSync('index.html','utf8');const m=html.match(/<script>([\s\S]*)<\/script>/);if(m)try{new Function(m[1]);console.log('OK')}catch(e){console.error(e.message)}"
```

## Architecture (single `<script>` block, ~5050 lines JS)

### Section Order (match this when adding code)
1. **CONFIG** (~L238) — constants: `CELL`, `WALL_H`, `MOVE_SPEED`, `BULLET_DMG`, `ENEMY_DMG`, etc.
2. **MAP** (~L265) — 22x17 grid: 0=empty, 1=wall, 2=crate, 3=enemy, 4=money, 5=car, 6=spawn
3. **GL INIT** (~L290) — WebGL context, canvas setup, `resize()`, early `var _gfxRes` for resolution scaling
4. **SHADERS** (~L347) — 4 shader programs compiled and linked:
   - `worldProg`: textured + lit + fog via `uFog` uniform (walls/floor/crates)
   - `billProg`: billboard sprites (unused legacy)
   - `partProg`: point sprites with additive blending (particles)
   - `entityProg`: 3D colored boxes with model matrix `uM`, `uFog`, `uAO` uniforms (enemies, allies, weapons)
5. **MATH** (~L491) — `perspective()`, `viewMatrix()`, `mat4Mul()`, `mat4T/S/RX/RY/RZ` (column-major Float32Arrays)
6. **PROCEDURAL TEXTURES** (~L520) — canvas-generated, power-of-two dimensions (64/128)
7. **LEVEL GEOMETRY** (~L618) — VBOs built from MAP grid; `pushQuad()` = 6 verts per face; `cubeVBO` unit cube
8. **COLLISION & RAGDOLL** (~L713) — `isWall()`, `collide()`, `lineOfSight()` (grid raymarching), 14-particle Verlet ragdoll with 21 constraints, `boneMat()`, `createRagdoll()`
9. **SOUND** (~L878) — Web Audio procedural synthesis `playSound()`, SpeechSynthesis `playVoice()` with 6s cooldown
10. **PARTICLES** (~L985) — `spawnParticles()`, `spawnDirectionalBlood()`, giblets, grenades, `updateParticles()`
11. **GAME STATE** (~L1217) — variables, `WEAPONS[]` array (L1242), graphics settings (`gfx*` vars, L1267), init functions:
    - `initGame()` (L1419), `initJuggernaut()` (L1463), `initMathMode()` (L1514), `initZombieMode()` (L1570), `initBossMode()` (L1610)
    - `spawnBarrels()`, `spawnDoors()`, `addXP()`
12. **PLAYER UPDATE** (~L1674) — `updatePlayer(dt)`: movement, collision, shooting, reload, F/G interaction, doors, objectives
13. **SHOOTING** (~L2022) — `playerShoot()`, `singleRaycast()` with head/body hitboxes, boss damage resistance, glitch weapon conversion, `explodeBarrel()`
14. **ENEMY AI** (~L2412) — `updateEnemies(dt)`: state machine `idle→combat→hurt→dead→blinded→captured`, boss charge AI, glitched enemy ally AI
15. **ALLY AI** (~L2631) — `updateAllies(dt)`: medics heal, defenders fight, dumb wanders; `allyShoot()`
16. **RENDER** (~L2974):
    - `drawWorld()` — walls, floor, ceiling, crates, cars, barrels, doors
    - `drawEnemies()` — enemy/zombie/ally/boss bodies (~20 boxes each), zombie/boss-specific appearance
    - `drawParticles()`, `resetAttribs()`
    - `drawWeapon3D()` — 5 weapon models: rifle, shotgun, pistol, sniper, glitch gun + muzzle flash
    - `drawMinigun()`, `drawMenuCharacter()`
    - `drawGiblets()`, `drawBloodPools()`, `drawBloodDecals()`
    - `drawPlayerSkeleton()` — 3rd person skeleton player model (cheat mode)
    - `render()` — orchestrates all draw calls, 3rd person camera support
17. **MINIMAP** (~L4334) — 2D canvas minimap with enemies, allies, barrels, doors, boss
18. **HUD** (~L4429) — `updateHUD()`: HP, ammo, objectives, scope overlay, bodycam overlay, mission prompts
19. **INPUT** (~L4818) — keydown/keyup/mouse/contextmenu event listeners
20. **GAME CONTROL** (~L4842) — `game` object: `start()`, `startJuggernaut()`, `startMath()`, `startZombie()`, `restart()`, `toMenu()`; shop/trade functions; cheat code system (`activateCheat()`)
21. **SCREEN EFFECTS** (~L5100) — `drawRain()`, `drawFlashlight()`
22. **MAIN LOOP** (~L5172) — `frame()`: updatePlayer→updateEnemies→updateAllies→updateParticles→render→HUD

### Entity Rendering Pattern (entityProg)
All 3D characters (enemies, allies, first-person weapon) use the same unit cube VBO with a `box(matrix, sx,sy,sz, r,g,b)` helper. Body parts are composed via matrix multiplication chains:
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
- `'normal'` — kill enemies, collect money, evacuate
- `'juggernaut'` — endless waves, 500HP, minigun, unlocked after first win
- `'math'` — multiplication quiz: find+kill enemy with correct answer, 60s timer
- `'zombie'` — wave-based survival, melee zombies (green skin, no weapons, reaching pose)
- `'boss'` — cheat-activated: fight "Чёрная Ошибка" (2000HP black glitch boss), rewards glitch weapon
- `'battleroyale'` — shrinking zone, 15 enemies, zone damages player outside, win=kill all
- `'wavedefense'` — defend point for 5 waves, point has 100HP, enemies attack point
- `'infection'` — 2 zombies + 8 humans, zombies infect on contact, survive 90s to win

### Graphics Settings System
4 presets (low/medium/high/ultra) with individual slider overrides. Two categories:
- **Performance**: `gfxResScale` (resolution), `gfxMaxParticles`, `gfxFogDist` (shader uniform `uFog`), `gfxShadows` (shader uniform `uAO`), `gfxDecalCap`, `gfxGibletCap`
- **Visual**: `gfxBrightness`/`gfxContrast`/`gfxSaturation` (CSS `filter` on canvas), `gfxVignette` (overlay div opacity)

Key functions: `applyGfxPreset()`, `resizeGfx()`, `applyGfxVisuals()`, `updateGfxUI()`

### Cheat Code System
`activateCheat(code)` — 4 codes entered in settings screen input field (case-insensitive):
- `BLOOD A NOT 5 YERS OLD A 18 YERS OLD` — unlock gore (shows confirmation dialog)
- `free pls 150` — +150 coins
- `skelet with black gun plssss` — skeleton mode + 3rd person camera
- `black world spawn` — teleport to boss arena fight

### Gore System
Hidden behind cheat code. Directional blood spray, blood→decal conversion, persistent floor/wall decals, blood pools under dead bodies, giblets (tumbling chunks with Verlet), screen blood droplets, dismemberment flags (`headGone`/`armLGone`/etc.), screen shake.

## Key Technical Gotchas

- **mat4RY sign convention**: `mat4RY(a)` maps +Z to `(-sin(a), 0, cos(a))`. For entities using `atan2(dx,dz)`, always negate: `mat4RY(-angle)`
- **Vertex attrib state leaks** — call `resetAttribs()` (disables attribs 0-3) before switching shader programs
- **Every `getElementById()` target must exist in DOM** — missing elements crash the entire script
- **Scene brightness** — ambient factor in world vertex shader must stay >= 0.55
- **Procedural textures** — must be power-of-two for WebGL1 mipmap/repeat
- **`_prevState` for enemy state recovery** — when setting enemy state to `hurt`, save `_prevState` first so `blinded`/`captured` enemies return to correct state after hurt recovery
- **Dead body `_fallSide`/`_limbPose`** — generated once on death via `undefined` check, don't reset
- **boneMat unit cube** — cube is -0.5..0.5, so Y-column scale must be `len` (not len/2)
- **boneMat column-major** — WebGL expects `[rx,ry,rz,0, ux,uy,uz,0, fx,fy,fz,0, tx,ty,tz,1]`
- **Bullet time** — `gameDt=dt*0.3` for enemies/physics, player always uses real `dt`
- **Zombie flag** — enemies with `e._zombie=true` render differently (green skin, no weapon, reaching pose) and use melee instead of shooting
- **TDZ with `gfxResScale`** — `resize()` is called at ~L297, before `let gfxResScale` at ~L1268. Use `var _gfxRes=1.0` (declared at ~L286) in `resize()`, synced via `resizeGfx()`
- **Fog/AO uniforms** — `uFog` and `uAO` must be set every time `worldProg` or `entityProg` is used (many locations). Missing uniform = stale value from previous draw call
- **Boss damage resistance** — in `singleRaycast()`: body damage ×0.5, headshot capped at 150, hurt timer 0.1s
- **Glitched enemies** — `e._glitched=true` + `e._isAlly=true` makes them fight other enemies; must check `_glitched` in enemy AI to avoid targeting allies

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
