# ORBit — Claude Code Handoff Document

## What the game is
ORBit is a single-file HTML5 canvas gravity sandbox game. The player launches planets into orbit around a star. Planets that fall into the sun cost points, stable orbits earn points. The sun evolves over time (yellow → red giant → neutron star → black hole), ending the game.

**Live at:** `az-build.github.io/zen-drift/ORBit.html`  
**File:** `zen-drift.html` (single self-contained HTML file)

---

## Architecture — Critical to understand

**Everything is in one file.** No build system, no modules, no external JS (except Firebase CDN). The game logic is wrapped in a strict-mode IIFE:

```js
(function(){
"use strict";
// ... entire game ...
})();
```

**This causes a persistent false positive in Node.js** — `node --check` and `new Function()` both choke on IIFE syntax at top level. The file is valid for browsers. Do NOT trust Node syntax checks on this file. Use brace-counting and manual inspection instead.

---

## The #1 recurring bug: Comment-swallowing

Inline `//` comments on the last line of a statement with no terminating character cause the next line to be treated as part of the comment. This has crashed the game ~10 times across sessions.

**Pattern that kills it:**
```js
var CAM_MAX=W // pan limit    ← no semicolon before comment
sun.homeX=W/2;                ← this line gets swallowed into comment
```

**Safe pattern:**
```js
var CAM_MAX=W;
sun.homeX=W/2; // pan limit
```

**Also kills it:** Inline comments on the last property of an object literal:
```js
stars.push({
  spike:bright // cross spike   ← NO! Parser reads }); as unexpected
});
```

**Rule:** Never put `//` comments on a line that doesn't end with `;`, `{`, `}`, `,`, or a closing bracket.

---

## Physics

- **Time step:** `TS=0.05`, each frame runs 2 sub-steps of `dt/2` each
- **Gravity:** `getGM()` returns current GM value, scales with stellar evolution
  - Normal: 4.0–4.8
  - Red giant: 4.8–8.0  
  - Neutron star: 8.0–30.0
- **Orbital velocity formula:** `orbV(r, dist) = sqrt(GM / dist)`
- **Boundary:** 700px circular from `sun.x, sun.y` — push-back force applies beyond 92% of radius
- **All coords are screen pixels** — the game doesn't use a separate world unit

## Key constants
```js
TS=0.05           // physics timestep
SUN_BASE_R=28     // sun base radius
SUN_MAX_MASS=8    // mass triggers black hole
PLATFORM_R=22     // launcher platform radius
LAUNCH_ZONE_R=55  // tap zone radius
MAX_PULL=160      // max launcher pull distance
TRAIL_LEN=350     // trail point count
JOY_R=52          // joystick radius
CAM_MAX=max(W,H)  // max pan distance
```

---

## Camera / Pan system

The game has a **joystick pan** system — no zoom. Camera offset is `camX, camY`.

**World transform in frame loop:**
```js
ctx.save();
ctx.translate(-~~camX + shake.x, -~~camY + shake.y);
// ... draw world ...
ctx.restore();
// ... draw UI (score, joystick, launcher) in screen space ...
```

**Coordinate conversion:**
```js
function gXY(e){
  var t = e.touches ? e.touches[0] : e;
  var sx = t.clientX, sy = t.clientY;
  return { x: sx + ~~camX, y: sy + ~~camY, sx: sx, sy: sy };
}
```

**UI elements** (score, menu button, launcher platform, joystick) are drawn **after** `ctx.restore()` in raw screen space. `inLZ()`, `inHandle()`, `inMenuZone()` all use `p.sx/p.sy` (screen coords), not world coords.

**The launcher** lives in screen space — `launcher.x = W/2`, `launcher.y ≈ H*0.75`. It does NOT move with the camera. This is intentional.

---

## Stellar Evolution

Sun evolves based on `sun.mass / sun.maxMass`:
- **0–38%:** Yellow G-type star, GM 4.0–4.8
- **38–82%:** Red giant, orange-crimson, expands to 2.2× base radius, GM 4.8–8.0
- **82–100%:** Neutron star, blue-white, collapses, GM 8.0–30.0

Phase transitions trigger: screen shake, flash, `addRing()`, `disturb()`
- Red giant transition: shake.mag = 6
- Neutron star transition: shake.mag = 18

When `sun.mass >= sun.maxMass`: black hole sequence begins.

---

## Black Hole sequence

1. `blackHole.active = true`, sun shrinks/implodes over 600ms
2. After 600ms: tiny pulsing singularity dot
3. Planets get suctioned one by one every 80ms
4. "GAME OVER" fades in after 1000ms
5. Score screen appears after all planets devoured

Launching is blocked during black hole (`blackHole.active || blackHole.done`).

---

## Moon system (NEW — may want to rebuild this)

Adrian wanted to add moons. The concept:
- During planet cooldown, a "moon hopper" fills (1 moon per 3s, max 5)
- Moons are 1px dark grey BBs (`rgba(80,90,110)`)
- Player can shoot them during cooldown
- If not shot, they auto-release as slow drifters → asteroid field
- Planets attract moons gravitationally (force 0.003/(d²+20))
- Capture: moon passes within 8× planet radius at <50% orbital speed
- Captured moon = green flash on planet, `moonCaptured=true`
- Scoring: 0.5pt per captured moon per bar fill
- Moon eaten by sun = −1 pt (not −10)

**Problems encountered:**
- Moons were showing leader lines (hard blocked via `if(o.isMoon)return`)
- Moons were getting −5 when bumping planets (fixed: collision doesn't score, only sun absorption)
- Large orbit rings appearing (from captured moon orbit indicator — REMOVED)
- Slingshot target ring was showing for all planets, not just moon shots

**Moon toggle:** `moonsEnabled` var, toggled via menu button.

---

## Scoring

- Each scoring period: `pts = stableOrbCount()` — only blue/stable orbits count
- Scoring period length decreases as more planets orbit
- Planet absorbed by sun: **−10 pts**
- Moon absorbed by sun: **−1 pt**
- Captured moon: **+0.5 per bar fill**
- Black hole devours planets: −10 each, −1 for moons

`penaltyPops[]` array drives floating text popups. Each item: `{x, y, vy, life, val, isBonus?, label?}`

---

## Firebase integration

Firebase is loaded as ES module from CDN. **Wrapped in try/catch** because it fails in offline/PWA mode and throws a cross-origin "Script error." that iOS Safari shows with no line number.

```js
try {
  const app = initializeApp(firebaseConfig);
  // ... setup ...
  window._fbReady = true;
} catch(e) {
  window._fbReady = false;
  console.warn('Firebase unavailable:', e.message);
}
```

Game checks `window._fbReady` before any Firebase calls.

---

## Key functions reference

| Function | Purpose |
|----------|---------|
| `resize()` | Sets W, H, resets canvas, places sun/launcher |
| `frame(now)` | Main render loop |
| `queueNext()` | Sets up next launch (planet or moon) |
| `launchPlanet()` | Fires current orb/moon |
| `autoLaunch()` | Auto-fires when launch window expires |
| `updateOrb(o, dt)` | Physics for one orb |
| `collide()` | Orb-orb elastic collision (3 passes) |
| `applyG()` | Sun gravity + moon-planet gravity |
| `drawOrb(o)` | Renders planet (or moon as 1px dot) |
| `drawLeaderLine(o)` | Trajectory prediction line (blocked for moons) |
| `drawSun(t)` | Sun with convection cells, plasma loops |
| `drawBlackHole(t)` | Singularity + collapse animation |
| `drawJoystick()` | Left-thumb pan joystick UI |
| `drawGrid(t)` | Spacetime fabric grid |
| `wAt(px, py)` | Wave field value at world coord |
| `disturb(px, py, str)` | Ripple the wave field |
| `getGM()` | Returns current gravity constant |
| `orbV(r, dist)` | Ideal orbital velocity |
| `stabCol(t)` | Color for instability value 0–1 |

---

## Orb object structure

```js
{
  x, y, vx, vy,
  r,              // current radius (may squish)
  baseR,          // base radius
  mass,           // baseR³ × 0.0005
  angle, spin,
  glow,           // 0–1, decays
  squishX, squishY, squishVX, squishVY,
  trail: [],      // {x,y,age,col} array
  isMoon,         // true for moons
  moonCaptured,   // true if orbiting a planet
  moonHostIdx,    // index into orbs[] of host planet
  moonCount,      // how many moons this planet has
  captureFlash,   // 0–1 green glow on capture
  dead,
  respawnTimer,
  instability,    // 0=stable blue, 1=falling red
  ptype,          // PTYPES entry with colors
  craterSeeds,    // 4 random angles for crater rendering
  ringTilt,       // 0.15–0.40 for ringed planets
  idleTime,
  suctioning,     // true when being absorbed
  suctionT,
  suctionSpeed,   // ms to complete suction
  _leaderCache,
  _leaderFrame
}
```

---

## Decisions Adrian made / preferences

- **No zoom** — replaced with joystick pan (bottom-left)
- **Launcher in screen space** — always at bottom-center regardless of pan
- **No trail or leader lines for moons** — ever, even captured
- **Moons are 1px dark grey pixels** — not small planets
- **Black hole is simple** — just collapse + singularity, no accretion disc (tried it, looked wrong)
- **No sunspot visuals** — removed (looked weird)
- **Boundary is circular, 700px from sun center** — not screen-edge based
- **Background (stars/nebula/galaxies) never zoom or pan** — fixed to screen
- **Grid extends 120% beyond screen** for pan coverage
- **Double-tap joystick resets camera to home**
- **Evan Brammer quote** in menu: "I was sick and dying and Orbit saved my life."

---

## Known issues to address

1. **Launch after panning** — launcher is in screen space so always hittable, but physics launch direction may feel off when panned far. Worth testing.
2. **Script error on PWA** — Firebase offline issue, now wrapped in try/catch. If still occurs, check for duplicate `var` declarations at IIFE top level.
3. **Moon capture is very hard** — gravity force may need further tuning. Current: `0.003/(d²+20)`. Try 0.005 or 0.008.
4. **-5/-10 pops when moons bump planets** — was a `scorePop` rendering issue. If it recurs: ensure `penaltyPops` with `isBonus:true` show green, and collision function never calls `penaltyPops.push()`.
5. **Joystick visibility** — was accidentally not being called. `drawJoystick()` must be called in the UI section (after `ctx.restore()`).

---

## Debug tips

**Finding syntax errors:**
```python
# Count braces in full script
content = open('zen-drift.html').read()
lines = content.split('\n')
start = next(i for i,l in enumerate(lines) if l.strip()=='<script>' and i>500)
end = next(i for i,l in enumerate(lines) if l.strip()=='</script>' and i>2900)
code = '\n'.join(lines[start+1:end])
bd=0
for ch in code:
    if ch=='{': bd+=1
    elif ch=='}': bd-=1
print(bd)  # should be 0
```

**Finding comment-swallowing:**
```python
for i in range(start, end-1):
    line = lines[i]
    if '//' not in line: continue
    before = line[:line.index('//')].rstrip()
    if not before: continue
    last = before[-1]
    next_line = lines[i+1].strip()
    if last not in (';','{','}',',','(',')','"',"'",'[',']') and next_line:
        print(f'Line {i+1}: {line[:80]}')
```

**Runtime error on iOS:** If "Script error." appears with no line number, it's cross-origin (Firebase). If it appears with a line number, it's a real JS error. Add `window.onerror = function(msg,src,line){ alert(line+': '+msg); }` temporarily to catch it.

---

## What's NOT in the live GitHub version (things to add fresh)

Based on Adrian's notes, the **live GitHub version** predates:
- Moon system (hopper, BB moons, asteroid field, moon capture)
- Joystick pan system (replaced zoom)
- Edge of universe boundary visual
- Slingshot indicator on trajectory
- Screen shake on stellar evolution transitions
- Sun surface ripples on planet absorption
- Streak bonus system (was added then removed)
- `+5 Moon!` capture bonus (was added, may be removed)
- Chill mode moon/planet toggle
- Evan Brammer quote in menu

Start from the GitHub version and add these features incrementally, testing after each one. Don't add them all at once.
