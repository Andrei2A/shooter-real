# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Operation Abandoned** ("Операция Заброшка") — a hardcore browser-based 3D FPS built as a single `index.html` file (~4750 lines). Full spec in `ТЗ.md` (Russian). Target hardware: Nvidia RTX 3000 8GB / 64GB RAM.

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

## Architecture (single `<script>` block, ~4550 lines JS)

### Section Order (match this when adding code)
1. **CONFIG** (~L211) — constants: `CELL`, `WALL_H`, `MOVE_SPEED`, `BULLET_DMG`, `ENEMY_DMG`, etc.
2. **MAP** (~L239) — 22x17 grid: 0=empty, 1=wall, 2=crate, 3=enemy, 4=money, 5=car, 6=spawn
3. **GL INIT** (~L263) — WebGL context, canvas setup, `resize()`
4. **SHADERS** (~L281) — 4 shader programs compiled and linked:
   - `worldProg`: textured + lit + fog (walls/floor/crates)
   - `billProg`: billboard sprites (unused legacy)
   - `partProg`: point sprites with additive blending (particles)
   - `entityProg`: 3D colored boxes with model matrix `uM` (enemies, allies, weapons)
5. **MATH** (~L457) — `perspective()`, `viewMatrix()`, `mat4Mul()`, `mat4T/S/RX/RY/RZ` (column-major Float32Arrays)
6. **PROCEDURAL TEXTURES** (~L486) — canvas-generated, power-of-two dimensions (64/128)
7. **LEVEL GEOMETRY** (~L584) — VBOs built from MAP grid; `pushQuad()` = 6 verts per face; `cubeVBO` unit cube at L652
8. **COLLISION & RAGDOLL** (~L679) — `isWall()`, `collide()`, `lineOfSight()` (grid raymarching), 14-particle Verlet ragdoll with 21 constraints, `boneMat()`, `createRagdoll()`
9. **SOUND** (~L844) — Web Audio procedural synthesis `playSound()`, SpeechSynthesis `playVoice()` with 6s cooldown
10. **PARTICLES** (~L951) — `spawnParticles()`, `spawnDirectionalBlood()`, giblets, grenades, `updateParticles()`
11. **GAME STATE** (~L1183) — variables, `WEAPONS[]` array (L1205), `ALLY_DEFS[]` (L1259), init functions:
    - `initGame()` (L1286), `initJuggernaut()` (L1330), `initMathMode()` (L1381), `initZombieMode()` (L1437)
    - `spawnBarrels()` (L1477), `spawnDoors()` (L1489), `addXP()` (L1511)
12. **PLAYER UPDATE** (~L1522) — `updatePlayer(dt)`: movement, collision, shooting, reload, F/G interaction, doors, objectives
13. **SHOOTING** (~L1869) — `playerShoot()`, `singleRaycast()` with head/body hitboxes, `explodeBarrel()` (L2147)
14. **ENEMY AI** (~L2206) — `updateEnemies(dt)`: state machine `idle→combat→hurt→dead→blinded→captured`
15. **ALLY AI** (~L2370) — `updateAllies(dt)`: medics heal, defenders fight, dumb wanders; `allyShoot()` (L2568)
16. **RENDER** (~L2670):
    - `drawWorld()` (L2713) — walls, floor, ceiling, crates, cars, barrels, doors
    - `drawEnemies()` (L2740) — enemy/zombie/ally bodies (~20 boxes each), zombie-specific appearance
    - `drawParticles()` (L3117), `resetAttribs()` (L3148)
    - `drawWeapon3D()` (L3154) — 4 weapon models: rifle, shotgun, pistol, sniper + muzzle flash
    - `drawMinigun()` (L3501), `drawMenuCharacter()` (L3570)
    - `drawGiblets()` (L3635), `drawBloodPools()` (L3663), `drawBloodDecals()` (L3694)
    - `render()` (L3745) — orchestrates all draw calls
17. **MINIMAP** (~L3886) — 2D canvas minimap with enemies, allies, barrels, doors
18. **HUD** (~L4041) — `updateHUD()`: HP, ammo, objectives, scope overlay (L4294), bodycam overlay, mission prompts
19. **INPUT** (~L4366) — keydown/keyup/mouse/contextmenu event listeners
20. **GAME CONTROL** (~L4390) — `game` object: `start()`, `startJuggernaut()`, `startMath()`, `startZombie()`, `restart()`, `toMenu()`; shop/trade functions (L4466)
21. **SCREEN EFFECTS** (~L4580) — `drawRain()`, `drawFlashlight()`
22. **MAIN LOOP** (~L4654) — `frame()`: updatePlayer→updateEnemies→updateAllies→updateParticles→render→HUD

### Entity Rendering Pattern (entityProg)
All 3D characters (enemies, allies, first-person weapon) use the same unit cube VBO with a `box(matrix, sx,sy,sz, r,g,b)` helper. Body parts are composed via matrix multiplication chains:
```
baseM = T(pos) * RY(-angle)
limb  = baseM * T(pivot) * RX(swing) * T(offset) * S(size)
```

### Weapon System
`WEAPONS[]` array (index 0-4): assault rifle, shotgun, pistol, sniper, minigun. Each has: `name`, `dmg`, `mag`, `reserve`, `firerate`, `reload`, `spread`. `currentWeapon` selects active weapon. `player._weaponAmmo[]` / `player._weaponReserve[]` persist ammo across switches. `drawWeapon3D()` renders unique 3D models per weapon.

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

### Gore System
Directional blood spray, blood→decal conversion, persistent floor/wall decals (cap 150), blood pools under dead bodies, giblets (tumbling chunks with Verlet), screen blood droplets, dismemberment flags (`headGone`/`armLGone`/etc.), screen shake.

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
| 1-4 | Weapon switch (rifle/shotgun/pistol/sniper) |
| T | Throw grenade |
| Z | Bullet time |
| N | Night vision toggle |
| L | Flashlight toggle |

## Persistent State (survives between missions)
`playerCoins`, `playerMedkits`, `weaponBonus`, `armorBonus`, `lootCount`, `playerBags`, `playerCuffs`, `jugUnlocked`, `playerXP`, `playerLevel`, `hasShotgun`, `hasPistol`, `hasSniper`, `hasShield`, `hasBulletTime`, `hasNightVision`, `hasFlashlight`, `goldSkin`

Reset per mission in `initGame()`: `capturedEnemy`, `evacTimer`, `particles`, `helmetCracks`, `allies`, `enemies`, `barrels`, `doors`, `grenades`
