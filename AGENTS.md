# SLOPE RACER '84 — Agent Notes

Standalone 80s-style skiing game built with **Three.js**. Everything lives in a single file: `index.html` (HTML + CSS + ES module JS). There is no build step.

## How to run

Serve the folder over HTTP (ES modules + CDN import map need a server):

```bash
python3 -m http.server 8765
```

Open `http://localhost:8765`.

Three.js is loaded from unpkg via an import map:

```html
<script type="importmap">
  { "imports": { "three": "https://unpkg.com/three@0.160.0/build/three.module.js" } }
</script>
```

## Game controls

| Input | Action |
|--------|--------|
| **←** / **A** | Steer left |
| **→** / **D** | Steer right |
| **Enter** / **Space** | Start game (menu) or retry (game over) |
| **Escape** | End run early → game over screen |
| Click **PRESS START** / **TRY AGAIN** / **MENU** | Same as above via UI |

There is **no manual speed control**. Speed is fixed per level and increases automatically.

## Gameplay rules

- Chase-camera skier going **downhill** (world scrolls toward the camera).
- **Dodge trees** and **other skiers** (people). Hitting one costs a life.
- **Jump ramps** (pink-sided) launch you into the air.
- **Score** ticks while skiing; higher speed (later levels) = more points. Air time and landings add bonuses.
- **3 lives**. After the 3rd crash → game over menu.
- **Level up every 30 seconds** (`LEVEL_INTERVAL`). Speed increases; density of hazards also scales with level.
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
```

### Structure inside `index.html`

1. **CSS** — retro UI (Press Start 2P), CRT scanlines, menus, HUD.
2. **DOM** — menus, HUD, banners (`#air-banner`, `#level-up-banner`), crash flash.
3. **JS module** — Three.js scene, game state, loop.

Important code regions (search for the section comments):

| Section | Role |
|---------|------|
| Palette `C` | Shared hex colors |
| Game state | `state` object + level constants |
| Three.js setup | Scene, camera, renderer, lights |
| Track group | Pitched slope + banks + stripes |
| Parallax mountains | Side peaks that scroll slower than the run |
| Player skier | Cute box-mesh character |
| Object factories | `makeTree`, `makePerson`, `makeJump` |
| Spawning | `spawnAhead`, `clearObjects` |
| Game flow | `startGame`, `endGame`, `crash`, `levelUp` |
| Main loop | `animate` → `tickPlay` |

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

- Renderer internal resolution is `window / PIXEL_SCALE` (`PIXEL_SCALE = 3`), then CSS-scaled full screen.
- Flat-shaded `MeshLambertMaterial` boxes for characters/props.
- CSS scanlines + vignette overlays.

## Core systems

### Modes (`state.mode`)

| Mode | Behavior |
|------|----------|
| `menu` | Title screen; idle skier bob; mountains drift slowly |
| `play` | Full `tickPlay` simulation |
| `gameover` | Final score + retry / menu |

### Input

Keydown/keyup set `state.keys.left` / `state.keys.right`. Steering is applied each frame in `tickPlay` from those flags (not from discrete key repeats).

### Spawning

`spawnAhead()` places trees, people, and occasional jumps at `state.nextSpawnZ` (and slightly behind it). Gaps shrink as `state.difficulty` (level) rises. The loop keeps enough objects ahead (min Z roughly &lt; −95).

Objects are stored as:

```js
{ mesh, type: 'tree'|'person'|'jump', x, z, radius, drift?, used? }
```

### Collision

When an object’s Z is near the player (`≈ -3 … 2.5`):

- **Jump** + on ground + not used → launch (`velY`), air scoring.
- **Tree / person** + low air height (`playerY < 1.1`) → `crash()`.

Invincibility frames (`state.invincible`) and `crashCooldown` prevent multi-hit spam. Crashes do **not** change speed (speed is level-locked).

### Jump / air scoring

- Hitting a ramp sets `onGround = false` and upward `velY` (scales with speed).
- While airborne: continuous air points ∝ air time × speed; UI shows `AIR +…`.
- On landing after meaningful air time: landing bonus.

### Parallax mountains

`mountainLayers` — arrays of peak meshes on both sides of the run:

- **Far layer** scrolls at ~22% of run speed.
- **Mid layer** scrolls at ~45%.

`scrollMountains(baseMove)` advances and wraps Z. Peaks use `MeshBasicMaterial` with `fog: false` so they stay visible.

### Scoring (play)

- Continuous: `score += speed * 0.4 * dt`
- Air: scaled by air time and speed
- Jump touch / landing bonuses

### Persistence

`updateHigh()` writes hi-score to `localStorage` when the run ends or score is shown on menus.

## Important functions

| Function | Purpose |
|----------|---------|
| `speedForLevel(n)` | Level → numeric speed |
| `startGame()` | Reset state, clear/spawn obstacles, show HUD |
| `endGame()` | Game over UI, save hi-score |
| `showMenu()` | Return to title |
| `levelUp()` | Increment level, bump speed, show banner |
| `crash()` | Lose a life; 0 lives → end game |
| `spawnAhead()` | Add next wave of hazards/jumps |
| `tickPlay(dt)` | Physics, scroll, collide, level timer, camera |
| `animate()` | `requestAnimationFrame` loop; snow + render |
| `makeTree` / `makePerson` / `makeJump` | Procedural meshes |
| `box(w,h,d,color,x,y,z)` | Helper for chunky geometry |

## Player model

- Root `player` (on `track`): position, lean (`rotation.z`), crash wobble.
- Child `playerMesh` with `rotation.y = Math.PI` so the face points **downhill (−Z)**; camera sees the back of the character.
- Built from stacked boxes (skis, body, pink hat, pom-pom, blush).

## Tuning knobs

| Constant | Effect |
|----------|--------|
| `LEVEL_INTERVAL` | Seconds between level-ups (default 30) |
| `BASE_SPEED` / `SPEED_PER_LEVEL` | Pace curve |
| `MAX_LEVEL` | Cap on level / speed growth |
| `TRACK_PITCH` | How steep the downhill read feels |
| `PIXEL_SCALE` | Chunkier pixels when higher |
| `SLOPE_W` | Half-width playable bounds scale with this |
| Spawn rates in `spawnAhead` | Tree count, person/jump probability, gap size |
| Mountain `speed` in `buildMountainLayer` | Parallax amount |

## UI elements (DOM ids)

| Id | Role |
|----|------|
| `#hud` | Score, lives, level, speed readout |
| `#menu` / `#gameover` | Title and wipe-out screens |
| `#air-banner` | Air time feedback |
| `#level-up-banner` | Level-up flash |
| `#crash-flash` | Red full-screen hit flash |
| `#combo` | Jump / landing point callouts |

## Conventions for changes

- Keep the game a **single standalone HTML file** unless there is a strong reason to split.
- Prefer adjusting constants and `tickPlay` / `spawnAhead` over adding frameworks.
- New obstacles: add a factory like `makeTree`, spawn in `spawnAhead`, handle type in the collision loop, parent meshes to **`track`** (not `scene`), remove with `track.remove`.
- Anything that should scroll with the run belongs on `track`; sky stays on `scene` so it does not pitch with the slope.
- Preserve the 8-bit aesthetic: low internal resolution, flat colors, simple box characters, CRT overlays.
- After behavior changes, verify: steer, jump air score, 3-crash → menu, level-up at 30s, mountains visible and parallax scrolling.
