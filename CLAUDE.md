# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Game

Open `shooter.html` directly in any modern browser — no build step, no server, no dependencies.

```bash
# Windows
start shooter.html

# Or via the shell from this directory
start "" "shooter.html"
```

After any code change, reload the browser tab (F5) to see the update. There is no hot-reload.

## Git Workflow

Every meaningful change to the game should be committed with a descriptive message and pushed:

```bash
git add shooter.html
git commit -m "feat: short description of what changed"
git push
```

To revert `shooter.html` to a previous commit:

```bash
git log --oneline          # find the commit hash
git checkout <hash> -- shooter.html
git commit -m "revert: reason"
git push
```

Remote: `https://github.com/BrutusFilip/pixel-shooter` (branch `main`)

## Architecture

Everything lives in `shooter.html` as a single file: inline `<style>`, HTML overlay screens, and a `<script>` block. The script is divided into labeled sections A–K:

| Section | Responsibility |
|---------|----------------|
| A | Constants (`CANVAS_W/H`, `TILE`, `STATES`) |
| B | Input — `keys{}` keyboard state, `mouse{}` position/click |
| C | `Particle` class + spawn helpers (`spawnWalkerDeath`, `spawnMuzzleFlash`, etc.) |
| D | `Bullet` class — owner `'player'`\|`'enemy'`, trail rendering |
| E (sprites) | `drawPixelBody`, `drawPixelGun`, `drawWalker`, `drawDasher`, `drawTank` — all canvas primitives |
| E (enemies) | `Enemy` base, `Walker`, `Dasher`, `Tank` classes |
| F | `Player` class |
| G | `LEVEL_DEFINITIONS` array + `WaveSystem` class |
| H | `circleCircle()`, `resolveCollisions()`, `applyContactDamage()` |
| I | `drawFloor()`, `drawHUD()`, `renderFrame()` |
| J | Game state machine (`setState`), `gameLoop` (rAF), global arrays |
| K | Boot — canvas/ctx setup, button wiring, `clamp()` utility |

### Game Loop (Section J)

Each frame: `waveSystem.update` → `player.update` → `enemies[].update` → `bullets[].update` → `particles[].update` → `resolveCollisions` → `applyContactDamage` → prune dead objects → `renderFrame` → check state transitions.

`dt` is capped at `0.05s` to prevent entity teleportation when the tab loses focus.

### State Machine

`MENU → PLAYING → LEVEL_COMPLETE → PLAYING (level++)` or `GAME_OVER`. All screen overlays are `<div class="screen hidden">` elements; `setState()` removes/adds the `hidden` class. Beating level 5 increments `currentLevel` to 6, which signals the win variant of `GAME_OVER`.

### Damage System

- **Bullet hits** — `player.takeDamage(amount, flash=true)`: applies damage immediately, triggers `flashTimer` for visual flicker.
- **Contact damage** — `applyContactDamage` calls `player.takeDamage(enemy.damage * dt, ...)` every frame while overlapping. Flash is only triggered when `flashTimer` is already at zero (prevents constant flicker).
- `flashTimer` is visual only — it does **not** block damage.

### Wave System (Section G)

`LEVEL_DEFINITIONS` is a plain array of level objects. Each level has a `waves` array of `{ type, count, delay, edge }` entries. `WaveSystem.loadLevel()` flattens these into a time-sorted `spawnQueue` with absolute `spawnAt` timestamps. The level is complete when `allQueued === true && enemies.length === 0`.

To add a level: append a new object to `LEVEL_DEFINITIONS`. To add a new enemy type: implement a class extending `Enemy`, add a `draw<Type>` sprite function, and register it in `spawnEnemy()`.

### Sprite Conventions

All sprites are drawn procedurally with `ctx.fillRect` / `ctx.arc`. They are drawn centered at the origin, with `ctx.save/translate/restore` wrapping each call. The Dasher sprite additionally applies `ctx.rotate(angle)` so it faces its direction of travel. Always reset `ctx.globalAlpha = 1` after any semi-transparent draw.
