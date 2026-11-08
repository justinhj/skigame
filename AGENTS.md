# SLOPE RACER '84 — Agent Notes

Standalone 80s-style skiing game built with **Three.js**. Everything lives in a single file: `index.html` (HTML + CSS + ES module JS). There is no build step.

## How to run

Load the single `index.html` file directly in the browser (e.g. by double-clicking it or using a `file://` path). **Do not start a local web server.**

Three.js is loaded from unpkg via an import map:

```html
<script type="importmap">
  { "imports": { "three": "https://unpkg.com/three@0.160.0/build/three.module.js" } }
</script>
```

## Game controls

| Input | Action |
|--------|--------|
| **←** / **A** | Steer left (on ground) / Spin counter-clockwise (in ramp air) |
| **→** / **D** | Steer right (on ground) / Spin clockwise (in ramp air) |
| **Space (Hold & Release)** | Hold to crouch & charge jump; release on the light green zone before a ramp for Super Jump |
| **Pointer Drag / Touch** | Drag horizontally to steer on ground or rotate/spin in ramp air |
| **Enter** / **Space** / Any non-modifier key | Start game (from menu) or retry (game over) |
| **Escape** | End run early → game over screen |
| Click / Tap screen | Start game from menu; buttons for **TRY AGAIN**, **MENU**, and **MUSIC: ON/OFF** |

There is **no manual speed control**. Speed is fixed per level and increases automatically.

## Gameplay rules & mechanics

- Chase-camera skier going **downhill** (world scrolls toward the camera).
- **Manual Jump (`Space` Hold & Release)**:
  - Holding `Space` causes the skier to crouch ready for the jump (visual squash and forward lean).
  - Releasing `Space` executes the jump (`velY = 6.8`, apex height ~1.05m).
  - Can clear other skiers (requires `playerY >= 0.95`), rewarding +75 points (`comboEl: CLEARED SKIER! +75`).
  - Cannot clear trees (trees require height >= 2.0 to clear, so manual jumps will crash into trees).
  - Manual jumps do not allow spinning or tricks (player remains facing forward; tricks and spins are reserved for ramp jumps).
- **Jump ramps & Super Jump**:
  - Ramps launch skier into the air.
  - Each ramp has a visible **light green approach zone** with chevron markings directly in front of it.
  - If you are fully charged (hold `Space` for $\ge 0.15$s) and release `Space` while over this green zone, you trigger a **SUPER JUMP**:
    - Launches 3x higher into the air (`velY * 1.732`).
    - Awards high points (`speed * 4.0`).
    - Plays a distinct futuristic retro power-chord arpeggio SFX (`sfx.playSuperJump()`).
    - Displays `SUPER JUMP! +[points]` banner.
- **Dodge trees** and **other skiers** (people). Hitting one costs a life. Other skiers drift left/right and carve downhill at 1/3 player speed.
- **Air tricks & landing alignment**:
  - While airborne, steering controls rotate player yaw instead of horizontal position.
  - Spinning grants continuous points (`yawChange * 20`) and spin SFX (`sfx.playSpin()`).
  - Safe landings require landing facing forward (within ±25° of 0°) or backward (within ±25° of 180°).
  - Landing forward yields air time bonus + spin bonus (`comboEl: LANDING / SPIN LANDING`).
  - Landing backward yields air time bonus + spin bonus + 150 extra backward bonus (`comboEl: BACKWARD LANDING`).
  - Landing sideways (> 25° deviation from forward or backward) results in a **WIPEOUT** (costs 1 life via `crash()`).
- **3 lives**. After the 3rd crash → game over menu.
- **Level up every 30 seconds** (`LEVEL_INTERVAL`). Speed increases; hazard density and fog density scale with level.
- Hi-score is stored in `localStorage` under key `slopeRacerHi`.

### Level / speed formula

```js
BASE_SPEED = 36
SPEED_PER_LEVEL = 7
MAX_LEVEL = 20
speed = BASE_SPEED + (level - 1) * SPEED_PER_LEVEL
```

| Level | Speed |
|-------|-------|
| 1 | 36 |
| 2 | 43 |
| 3 | 50 |
| … | +7 each |

## File layout

```
skigame/
  index.html   # entire game
  AGENTS.md    # this file
  favicon.ico  # icon
```

### Structure inside `index.html`

1. **CSS** — retro typography (Google Font 'Press Start 2P'), CRT scanlines, vignette, HUD, animated banners, menu styling with responsive text and retro buttons.
2. **DOM** — menus, canvas `#menu-skier`, twinkling `#stars`, cycle-animated instructions `#menu-instructions`, HUD, banners (`#air-banner`, `#level-up-banner`), crash flash, `#mute-btn`.
3. **JS module** — Three.js scene, Web Audio sound system, game state, animation loop.

Important code regions (search for the section comments):

| Section | Role |
|---------|------|
| Palette `C` | Shared hex colors for 80s neon palette |
| DOM lookups | Elements and mobile detection flags |
| Menu Instructions | Cycling help text banner system (`startInstructions`, `stopInstructions`) |
| 2D Menu Skier | Canvas pixel-art doodle on the title screen |
| Twinkling Stars | CSS-animated star elements on title screen |
| Game state | `state` object + level constants |
| Audio engine `sfx` | Web Audio API chiptune music (bass + melody scheduler) and procedural SFX (jump, spin, land, crash, level-up) |
| Three.js setup | Scene, camera, renderer, lights |
| Track group | Pitched slope + side banks + subtle OutRun road bands |
| Parallax layers | Side evergreen tree silhouettes + distant purple mountain backdrop (`bgMountainGroup`) |
| Player skier | Root `player` group + child `playerMesh` (skis with outline base, boots, body, arms, poles, head, hat, pom-pom, face, oval shadow, snow spray particles) |
| Object factories | `makeTree`, `makePerson`, `makeJump`, snow particles |
| Spawning | `spawnAhead`, `clearObjects`, spacing validation (`isNearJump`, `isJumpNearTree`) |
| Game flow | `startGame`, `endGame`, `showMenu`, `crash`, `levelUp`, `updateHUD`, `updateHigh` |
| Input handlers | Keyboard (`keydown`, `keyup`), Pointer/Touch (`pointerdown`, `pointermove`, `pointerup`), UI clicks |
| Main loop | `animate` → `tickPlay` / menu idle bob and drift |

## Coordinate system & “illusion of motion”

The player stays near **local origin** on the track. Obstacles move in **+Z** toward the camera; new ones spawn at more **negative Z**.

| Axis | Meaning |
|------|---------|
| **+X** | Right on the slope |
| **+Y** | Up (air jumps increase `playerY`) |
| **−Z** | Down the mountain (into the distance) |
| **+Z** | Toward / past the camera |

### Track pitch (downhill look)

Playable content is parented under a `track` group:

```js
const TRACK_PITCH = -0.34; // ~19° downhill
track.rotation.x = TRACK_PITCH;
```

Negative X rotation lowers far (−Z) points in world space so the road appears to drop away. The camera sits high behind the skier and looks further down the run (wide FOV ~68°).

### Pixel look

- Renderer internal resolution is `window / PIXEL_SCALE` (`PIXEL_SCALE = 3`), then CSS-scaled full screen with `image-rendering: pixelated`.
- Flat-shaded `MeshLambertMaterial` boxes for characters/props.
- CSS scanlines + radial vignette overlays.

## Core systems

### Audio system (`sfx`)

Self-contained procedural Web Audio API synthesizer:
- **`startMusic()` / `stopMusic()` / `toggleMute()`**: 8-step bassline sequencer (triangle wave) + melody (square wave) at 130 BPM with lookahead scheduling.
- **Procedural sound effects**:
  - `playJump()`: upward frequency sweep (triangle wave).
  - `playSpin()`: quick blip on rotation steps in air (square wave).
  - `playLanding()`: two-tone chime (C5 + E5 square waves).
  - `playCrash()`: filtered white noise buffer with lowpass sweep.
  - `playLevelUp()`: ascending C-major arpeggio.
- Must resume `AudioContext` on user interaction (`sfx.init()`).

### Modes (`state.mode`)

| Mode | Behavior |
|------|----------|
| `menu` | Title screen; idle skier bob; mountains drift slowly; instructions cycling |
| `play` | Full `tickPlay` simulation; music active |
| `gameover` | Final score + retry / menu buttons |

### Input handling

- **Keyboard**:
  - `state.keys.left` / `state.keys.right` updated by keydown/keyup.
  - Enter / Space restarts game from gameover or menu.
  - Escape ends run immediately into gameover.
- **Touch / Pointer**:
  - `pointerdown`, `pointermove`, `pointerup`, `pointercancel` track drag deltas (`state.touchDeltaX`).
  - Converted to horizontal position delta on the ground, or yaw rotation delta in the air.
  - Consumed and reset each frame in `tickPlay`.

### Spawning & clearance validation

`spawnAhead()` places trees, other skiers, and jump ramps at `state.nextSpawnZ`:
- Trees and jumps use `isNearJump` and `isJumpNearTree` to avoid impossible obstacle placements right next to or in front of ramps (min 14 units in Z, 5 units in X).
- Trees scale in density with `difficulty` (`1 + Math.floor(Math.random() * (2.4 + Math.min(4.4, diff * 0.65)))`).
- Skiers (`makePerson`) have random color suits and random lateral drift (`o.drift`), changing carving direction periodically.
- Gaps between waves shrink as difficulty rises: `Math.max(7, 17 - diff * 0.9) + Math.random() * 6`.
- The loop keeps enough objects ahead (min Z &lt; −95 or count &lt; 8).

Objects are stored as:

```js
{ mesh, type: 'tree'|'person'|'jump', x, z, radius, drift?, used? }
```

### Collision & jump mechanics

When an object’s Z is near the player (`≈ -3 … 2.5`):

- **Jump Ramp**: Launches player (`velY = 8 + speed * 0.12`). Requires skier to be on the ground (`playerY < 0.6`); airborne skiers flying above the ramp clear it cleanly without re-triggering. If the player releases a fully-charged jump while in the light green approach zone in front of the ramp, a **SUPER JUMP** is triggered with 3x height (`velY * 1.732`), high bonus score (`speed * 4.0`), and distinct power-chord arpeggio SFX. Hitting a ramp without releasing Space gives a standard ramp jump.
- **Tree**: Crashes if `playerY < 2.0` (cannot be cleared with manual jump).
- **Person / Other Skier**:
  - If airborne at `playerY >= 0.95`, safely clears the skier and awards a +75 "CLEARED SKIER!" bonus (once per skier).
  - If `playerY < 0.95`, triggers `crash()`.
- **Landing Evaluation**:
  - When airborne player returns to `playerY <= 0`:
    - Check `playerYaw` angle deviation from 0° (forward) or 180° (backward).
    - If deviation &le; 25°, safe landing bonus awarded (`comboEl` callout, landing SFX).
    - If deviation &gt; 25°, triggers `crash()` with wipeout callout.

### Scoring (play)

- Continuous: `score += speed * 0.4 * dt`
- In-air rotation: `score += abs(yawChange) * 20`
- Continuous air time: `airTime * speed * 0.8 * dt * 10`
- Jump launch: `Math.floor(speed * 1.2)` (or 3x for timed Super Jump)
- Cleared skier bonus: +75 points
- Safe forward landing: air bonus (`airTime * speed * 2.5`) + spin bonus (`cumulativeAirRotation * 50`)
- Safe backward landing: above + 150 extra backward points

## Important functions

| Function | Purpose |
|----------|---------|
| `speedForLevel(n)` | Level → numeric speed |
| `startGame()` | Reset state, clear/spawn obstacles, show HUD, start music |
| `endGame()` | Game over UI, save hi-score, stop music |
| `showMenu()` | Return to title screen, start instruction cycles |
| `levelUp()` | Increment level, bump speed, show level banner, play arpeggio |
| `crash()` | Lose a life, screen flash, wobble; if 0 lives → endGame |
| `spawnAhead()` | Add next wave of hazards/jumps with spacing clearance |
| `isNearJump` / `isJumpNearTree` | Spacing validation helpers to prevent unfair hazard overlaps |
| `tickPlay(dt)` | Physics, steering/spinning, object movement, collisions, landings, camera |
| `animate()` | `requestAnimationFrame` loop, snow animation, render |
| `makeTree` / `makePerson` / `makeJump` | Procedural 3D obstacle meshes |
| `box(w,h,d,color,x,y,z)` | Helper for chunky geometry boxes |
| `startInstructions()` / `stopInstructions()` | Cycles animated menu hints |

## Tuning knobs

| Constant | Effect |
|----------|--------|
| `LEVEL_INTERVAL` | Seconds between level-ups (default 30) |
| `BASE_SPEED` / `SPEED_PER_LEVEL` | Pace curve (default base 36, +7 per level) |
| `MAX_LEVEL` | Cap on level / speed growth (default 20) |
| `TRACK_PITCH` | Downhill angle (default -0.34 rad ~ 19°) |
| `PIXEL_SCALE` | Pixelation factor (default 3) |
| `SLOPE_W` | Playable width of the slope (default 28) |
| `rotSpeed` in `tickPlay` | Aerial spin rate for keyboard (default 7.5 rad/s) |
| `maxError` in landing check | Tolerance window in degrees for safe landing (default 25°) |

## UI elements (DOM ids)

| Id | Role |
|----|------|
| `#hud` | Score, lives hearts, level, speed readout |
| `#menu` / `#gameover` | Title and wipe-out screens |
| `#menu-instructions` | Cycling retro tutorial text banner on title screen |
| `#menu-skier` | 2D pixel art canvas preview |
| `#stars` | Twinkling starfield background for menu |
| `#air-banner` | Real-time spin angle and air bonus readout while jumping |
| `#level-up-banner` | Level-up notification flash |
| `#crash-flash` | Red full-screen hit flash |
| `#combo` | Jump, trick, and landing callout feedback |
| `#mute-btn` | Sound mute/unmute toggle |

## Conventions for agents making changes

- **Do not start a local web server** to run or verify changes; load the single `index.html` file directly in the browser instead.
- Keep the game a **single standalone HTML file** unless there is a strong reason to split.
- **Preserve the 8-bit / neon aesthetic**: low internal resolution (`PIXEL_SCALE`), flat colors (`MeshLambertMaterial` / `MeshBasicMaterial`), chunky box geometry, scanlines/vignette.
- **Audio changes**: Keep Web Audio synthesizer completely self-contained without external audio files. Check `sfx.init()` on user interactions.
- **Adding new obstacles / entities**:
  1. Create a factory function returning a `THREE.Group` (like `makeTree()`, `makePerson()`, `makeJump()`).
  2. Set `g.userData.radius` and `g.userData.type`.
  3. Parent meshes to **`track`** (not `scene`) so they inherit the slope pitch.
  4. Spawn in `spawnAhead()` ensuring clearance with jump ramps.
  5. Handle movement and collision detection in `tickPlay`.
  6. Clean up in `clearObjects()` and when `o.z > 25` using `track.remove(o.mesh)`.
- **Coordinate reference reminder**:
  - `track` has pitch rotation `track.rotation.x = -0.34`.
  - Content on `track` moves toward +Z.
  - Sky and fog live directly on `scene` so they don't pitch with the slope.
- After behavior changes, verify:
  1. Ground steering and aerial trick rotation (keyboard + touch/pointer).
  2. Jump ramp launching, air scoring, safe landing vs bad landing crash.
  3. 3-crash game over transition and retry.
  4. Music playback and mute button toggle.
  5. Parallax layers scrolling and horizon mountain group framing.
