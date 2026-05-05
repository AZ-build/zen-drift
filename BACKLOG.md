# ORBit — Backlog

**What this file is:** A single checklist of every idea, enhancement, and known issue for ORBit. Dispatch and Code should reference this when picking up work. Adrian adds ideas here too. Items are grouped by category and roughly prioritized within each group (top = higher priority). Check the box when shipped.

_Last updated: 2026-05-04_

---

## Gameplay & Balance

- [ ] **Perfect circle mechanic — too hard** — Adrian has never achieved one. Review tolerance/detection logic. Consider adding assist or forgiveness. (Source: Changelog 05-03 Dispatch)
- [ ] **Giant orbit deterrent** — Players can pan out and make massive orbits that break the feel. Options to explore: hard cutoff, diminishing returns by size, reward tight orbits more, or tighten blue-line qualification. Needs design discussion before committing. (Source: Changelog 05-03 Dispatch)
- [ ] **Moon capture difficulty** — Gravity force `0.003/(d²+20)` may be too weak. Try `0.005` or `0.008`. (Source: Handoff)
- [ ] **Scoring balance pass** — Once perfect-circle and giant-orbit issues are resolved, do an overall scoring balance review.

## UI & Controls

- [ ] **Pause button** — Menu button should be larger and styled as a standard pause icon (two vertical bars). (Source: Changelog 05-03 Dispatch)
- [ ] **Launcher boundary — menu collision** — Launcher shouldn't travel behind the menu. Menu acts as a hard wall. (Source: Changelog 05-03 Dispatch)
- [ ] **Edge indicators — simplify** — Off-screen planet indicators are too busy. Reduce to a small arrowhead pointing toward the planet, keep color logic. (Source: Changelog 05-03 Dispatch)
- [ ] **Moveable joystick** — Add a drag handle below the joystick so player can reposition it. Watch out for iOS swipe-up gesture. Bump default position up slightly. (Source: Changelog 05-03 Dispatch)

## Visuals & Polish

- [ ] **Space fabric / background edge** — Fabric doesn't extend to screen edges. Everything outside fabric boundary should be pitch black, no gaps. (Source: Changelog 05-03 Dispatch)
- [x] **Planet rotation + axial tilt** — Planets should spin on their own axis (varied speeds). Some with dramatic tilt (Uranus-style). Richer textures to differentiate same-template planets. (Source: Changelog 05-03 Dispatch)
- [ ] **Pan boundary ring polish** — Dashed red circle at `BND_R=700`. Currently fades in at ~25% pan. Verify it feels right and doesn't distract.

## Features Not Yet on GitHub (from Handoff)

These exist in local dev but haven't been pushed to `az-build.github.io/zen-drift/ORBit.html`. Add incrementally, test after each:

- [ ] Moon system (hopper, BB moons, asteroid field, moon capture)
- [ ] Joystick pan system (replaced zoom)
- [ ] Edge of universe boundary visual
- [ ] Slingshot indicator on trajectory
- [ ] Screen shake on stellar evolution transitions
- [ ] Sun surface ripples on planet absorption
- [ ] Chill mode moon/planet toggle
- [ ] Evan Brammer quote in menu

## Removed / On Hold (don't re-add without discussion)

- [x] ~~Streak bonus system~~ — Added then removed
- [x] ~~`+5 Moon!` capture bonus~~ — Added, may be removed
- [x] ~~Sunspot visuals~~ — Removed (looked weird)
- [x] ~~Black hole accretion disc~~ — Tried it, looked wrong. Keep simple collapse + singularity
- [x] ~~Zoom~~ — Replaced with joystick pan. No zoom.

## Known Bugs & Tech Debt

- [ ] **Launch after panning** — Physics launch direction may feel off when panned far. Launcher is screen-space, physics is world-space. Needs mobile testing. (Source: Handoff)
- [ ] **Firebase PWA "Script error"** — Now wrapped in try/catch. If it recurs, check for duplicate `var` declarations at IIFE top level. (Source: Handoff)
- [ ] **Moon penalty pops** — `-5/-10` pops when moons bump planets. If it recurs: ensure `penaltyPops` with `isBonus:true` show green, collision function never calls `penaltyPops.push()`. (Source: Handoff)
- [ ] **Comment-swallowing bug pattern** — Not a single bug but a recurring trap. Never put `//` comments on a line that doesn't end with `;`, `{`, `}`, `,`, or a closing bracket. (Source: Handoff)

---

_To add an idea: just append it to the right section with `- [ ]` and a source note._
