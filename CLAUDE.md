# Tank Buster — Edge of Fate

Top-down browser tank game. **Single self-contained `index.html`** (~2700 lines: inline CSS + JS, HTML5 Canvas). No framework, no build step, no dependencies, no `package.json`.

## How it's hosted & deployed

- **Repo**: `github.com/ZHFisher/Tank-Game` (private), branch `main`.
- **GitHub Pages** serves it at **https://zhfisher.github.io/Tank-Game/**. Every push to `main` auto-deploys in ~30s. **No build = zero build minutes** (we moved off Netlify because it ate the user's class build credits — do NOT reintroduce a build step or a Netlify dependency).
- **Assets** (`.mp3`, `.webp`, `.png`) live at the **repo root**, plus a `music/` subfolder. Loaded by relative path, so they must sit next to `index.html`.

## Local preview (STANDALONE — never touch Exercise3)

Tank-Game is fully self-contained. Preview it WITHOUT any dependency on another project:

- **Static server in this folder**: `python -m http.server 8123` (Python is installed) → open `http://localhost:8123/`. Or `npx serve`.
- **Or just push and test live**: https://zhfisher.github.io/Tank-Game/ (auto-deploys ~30s after a push to `main`).

**Hard rule: do NOT mirror game files into `../Exercise3`.** Exercise3 is finished, graded coursework and must stay 100% tank-game-free (no `game.html`, no game assets, no links). It was de-mirrored on purpose. The source of truth is `Tank-Game/index.html`; deploy = commit + push in this repo.

## Architecture (all inside index.html)

- `state` — single global holding everything: `scene`, `units`, `bullets`, `particles`, `gems`, `smokes`, `lightnings`, `shockwaves`, `camera`, timers, etc.
- **Scenes**: `'menu' | 'playing' | 'paused' | 'upgrade-pick' | 'gameover'`. The main `loop()` only updates game logic when `scene === 'playing'` (so pause freezes everything — particles too).
- `TANKS` — player/bot tank classes: `light, medium, mbt, ifv, laser`.
- `ENEMIES` — Onslaught enemy types: `soldier, rusher, monkey, rpg, humvee, super, mega, yesking` (mini-boss), `finalboss`.
- `MAPS` — 5 maps (`desert, snow, tropical, forest, fulda`), each 2400×1600 with camera follow.
- `MODES` — `onslaught` (continuous XP survival, the flagship), `tdm`, `ctf`.
- `UPGRADES` — roguelike pool picked on level-up.
- **Damage**: `DAMAGE_TANK[weapon][tankClass]` for tank-vs-tank; `WEAPON_DAMAGE[weapon]` flat for everything else, **then × `ARMOR_MULT[tankClass]` when an enemy weapon hits the player** (mbt 0.6 … light 1.25 — keeps MBT the tanky class in Onslaught despite its low raw HP). Bullet `speeds` dict + `life`/`radius` ternaries are in `makeBullet`.
- **Pathing**: `FLOW` flow field (40px grid, walls inflated 16px, BFS from player ~3×/s in `updateOnslaught`). Enemies use it whenever they have no LOS (`flowDirFrom`), else hold their `OPTIMAL` distance band with `steerClearOfWalls` whisker steering. `separateEnemies()` (called from `updateUnits`) un-stacks the swarm; bosses are immovable in that pass. TDM/CTF bots use whisker steering only (their goals aren't always the player). `buildFlowGrid()` runs in `buildMatch`.
- **Audio**: Web-Audio synth fallback (`playMG/playCannon/playRocket`) + real MP3 clips decoded to AudioBuffers (`SOUND_FILES`, `SOUND_OPTS`, `playClip`). Music uses an HTML5 `Audio` element.

## Onslaught mode (current design)

Vampire-Survivors / Disfigure style:
- **Continuous spawn**, time-scaled (cadence tightens, batch grows, enemy mix hardens over time). No waves.
- Enemies drop **cyan XP gems**; player collects them (magnet radius) → fills XP bar → **level up** → upgrade-pick card screen.
- **YESKING mini-boss every 5:00**; **FINAL BOSS at 30:00** (bullet hell); killing it unlocks endless **free play** with HP creep.
- HUD center: count-up timer `M:SS / 30:00`, then "FINAL BOSS" / "FREE PLAY"; plus `LVL n` and an XP bar.

## Current contents (save-file inventory — what exists right now)

**Controls**: WASD move · Mouse aim · LMB primary · RMB secondary · Q ability · **V camera toggle** · P/Esc pause. First person: WASD is view-relative (A/D strafe), mouse-look via pointer lock (click canvas to grab; Esc releases lock first, so pausing from locked FP takes Esc-Esc or P), arrow keys turn as a no-lock fallback.

**Camera modes** (`viewMode`: `'top' | 'fp'`, persisted as `tb-view`): pause-menu Camera buttons + V hotkey. First person is a **Wolfenstein-style raycaster over the unchanged 2D sim** — only the renderer swaps (`render()` early-returns into `renderFirstPerson()`). Pieces: `FP` (fov 1.15, per-column z-buffer), `rayHitRect` (slab ray-vs-AABB), `state.fpWalls` (map walls + 4 render-only border walls, built in `buildMatch`), biome `sky`/`skyHorizon` gradients on every map, texture-mapped 2px wall columns (per-biome 64×64 tiles, u from the ray hit point) with fog, distance-sorted billboards (`drawSpriteFP`: images for boss/monkey, **pre-rendered sprite canvases** for infantry/vehicles/bot tanks via `fpSprite` cache, bullets as glow orbs, gems/flags/smokes, decoration billboards + flat ground decals), parallax hills/sun/clouds sky dressing (`FP_SKY`/`buildFpMapArt`), class viewmodel with hull edge, recoil, muzzle flash and build themes (`drawViewmodelFP`), crosshair, threat arrows as relative bearings (ahead=up, in-cone skipped), minimap + vignette kept. Not drawn in FP: particles, damage floaters. Sprite occlusion is a centre-column z-test (slight edge popping — acceptable).

**Tank classes** (`TANKS`, player HP from `PLAYER_HP`):
| key | name | HP | speed | primary | secondary | ability (CD) |
|---|---|---|---|---|---|---|
| light | Scout | 200 | 160 | cannon 1.2s | MG | Dash, 6s |
| medium | Striker | 150 | 120 | cannon 0.9s | MG | Smoke, 12s |
| mbt | Iron Wall | 125 | 88 | cannon 0.6s | 50-cal heavy_mg 0.18s | Armor (2.5s invuln), 14s |
| ifv | Spearhead | 105 | 136 | autocannon 0.18s | rocket (homing+stun) 4.5s | Overdrive (fire×2 3s), 14s |
| laser | Phoenix | 130 | 130 | laser 0.45s (pierces 3) | MG | Overload (fire×2 3s), 14s |

**Onslaught enemies** (`ENEMIES`): soldier (30hp, pistol), rusher (25hp, fast, smg single-shot), monkey (80hp, throws watermelons, sprite `monkey.png`, 5 random screeches), rpg (40hp, slow homing missile that fizzles if you kill the trooper), **sniper** "Marksman" (35hp, from 5:00, stands still 0.9s with a red dashed laser-sight line to the player, then a fast 520px/s tracer, 22dmg, range 450), humvee (120hp, MG burst), super (350hp, purple, heavy cannon), mega (900hp, pink, 3-shot volley), **yesking** mini-boss (1800hp base, scales +12%/min past 5:00 → ~3960 at 15:00, spinning `yesking.webp` sprite, radial 4/5/6/8 orb patterns, every 5:00; **past 10:00 adds one aimed orb per volley**), **finalboss** "YESKING ASCENDED" (14000hp flat, 80px, triple bullet-hell pattern, at 30:00).

**Elites**: past 8:00 any non-boss spawn has a growing chance (4% → cap 22%) to roll elite — 3× HP, 3× bounty (= 3× XP, via `enemy.bounty` which `onEnemyKilled` now uses instead of `def.bounty`), 1.12× speed, pulsing gold ring, gold threat arrow + gold minimap dot.

**Modes** (`MODES`): onslaught (continuous XP survival — flagship), tdm (5v5 bots to 50 kills), ctf (3v3 bots, 3 captures).

**Upgrades** (`UPGRADES`, 17): damage, reload, hp, speed, ability-CD, regen, **thorns** (reworked: shooters take 15 reflect dmg when their rounds hit you — was dead code after melee removal), pierce, secondary-rate, **burn** (fire DoT), **chain** (lightning arc), **ricochet** (wall bounce), **stun** (periodic pulse), **crit** "Sabot Core" (+10%/stack, 2× dmg, gold floater), **vampire** "Nano Salvage" (heal 3%/stack of damage dealt), **explosive** "HE Shells" (cannon/autocannon/laser hits splash 25%/stack in 80px, orange shockwave), **magnet** "Magnet Coil" (+80 gem radius/stack). One **Reroll** per level-up screen (`#reroll-btn`, `state.rerollLeft`).

**HUD/feel**: floating damage numbers (`state.floaters`, capped at 70; white=hit, gold big=crit, cyan=chain, orange=HE splash), **minimap** bottom-right (walls, red hostiles, gold elites, big red bosses, white player, camera rect), low-HP red vignette under 35% HP, game-over **run recap** (kills / damage / DPS + build icon chips in `#run-stats`).

**Audio files** (repo root): `meow.mp3` (ability, pitched per class), `cannon/ifv_cannon/ifv_rocket/laser/heavy_mg/mg.mp3` (weapons), `allahu/death_npc1/death_npc2/death_female.mp3` (soldier death pool, random), `monkey1-5.mp3` (monkey — **death only**, throws use a quiet synth `playFwip()`; user found screech-per-throw annoying), `level_up.mp3`, `yesking.mp3` (boss entrance), `music/desert1-3.mp3` (desert bg). **Images**: `yesking.webp`, `monkey.png`.

**Firing/movement FX**: every shot sets `unit.recoil`/`muzzleFlash`/`muzzleWeapon` in `fireWeapon` (decayed in `updateUnits`); turrets kick back and draw a per-weapon-tinted flash star at `BARREL_TIP[class]` (super/mega enemies too). Cannon-family projectiles render as rotated shells (tracer glow + casing + dark tip) with smoke trails; cannon hits/wall impacts push small shockwave rings; kills push colour-coded expanding rings. Tread links scroll with distance travelled (`u.treadOffset`), tanks kick up biome-coloured dust.

**Build-themed player visuals** (`renderPlayerUpgradeFx` + bullet overlays): burn = exhaust flames + ember emitter + fire-haloed rounds; chain = cyan arcs crawling the hull + spark emitter + crackle on bullets; explosive = orange barrel bands + bigger muzzle flash + orange-glow shells; crit = gold trim ring; vampire = pulsing crimson core; laser bolts scale width with `damageMult` and go violet **plasma** at 2+ damage stacks. Particle EMISSION happens in update functions (pause-safe); render only draws.

## Known issues / TODO (not yet fixed)

_(none currently logged)_

_Resolved: `hasLineOfSight` is now an exact segment-vs-rect slab test (`segHitsRect`) — no more corner-clipping between 20px samples (2026-07-03); dead `melee_light`/`melee_heavy` entries deleted (2026-07-03); player cannon/autocannon/mg/rocket are in `WEAPON_DAMAGE` now (was: default-20 fallback); boss HP scaling was rewired from the dead `state.wave` to `state.matchTime` (2026-06-10)._

## How to do common changes

- **Add a sound**: copy mp3 to repo root → add to `SOUND_FILES` + `SOUND_OPTS` → route it in `playWeaponSound` or call `playClip('key')`. For full-auto weapons give a short `duration` slice + `variance`; for one-shots no throttle needed.
- **Add an enemy**: add to `ENEMIES` → add a render branch in `renderEnemy` → add to `spawnContEnemy` weight list → add a threat-indicator color → if it uses a new weapon, add that weapon to `speeds`, `WEAPON_DAMAGE`, the `life`/`radius` ternaries in `makeBullet`, and a bullet-render branch.
- **Add a weapon**: `speeds` dict + `WEAPON_DAMAGE` (+ `DAMAGE_TANK` if it hits tanks) + `life`/`radius` ternaries + bullet render branch + `playWeaponSound` route.
- **Add a tank class**: `TANKS` entry + `PLAYER_HP` + `TEAM_COLORS.red/.blue` + `renderTank` body & turret branches + menu shows it automatically.

## Gotchas — DO NOT REPEAT THESE

1. **Sound `duration` too short = silence.** `heavy_mg` at 0.12s chopped before the bang; needed 0.5s. Give clips enough duration.
2. **Enemy weapon `range` > ~480px = off-screen "invisible damage"** (camera shows ~480px each way). Keep enemy ranges ≤ ~460 so the shooter is on-screen before it fires.
3. **Melee enemies that walk into the player's hitbox render on top of the tank → "sparks from nowhere."** All enemies are now ranged + get pushed out of the player footprint. Don't reintroduce contact/melee damage without the push-out, and prefer visible projectiles.
4. **Stuck WASD**: a held key whose `keyup` fires off-window stays "pressed" → tank drifts forever. Always `clearAllInput()` on window `blur`, `visibilitychange`, pause toggle, and on resuming play from a modal.
5. **Music pitch** needs `preservesPitch=false` (+ `moz`/`webkit` variants) on the Audio element, else `playbackRate` won't bend pitch.
6. **MG/full-auto sound must be a single-shot slice played once per round** (not the multi-shot burst file). Otherwise audio desyncs from fire rate.
7. **Enemy bullets fizzle when their shooter dies**: `if (b.owner && !b.owner.alive && !b.owner.isPlayer)`. This is why killing a boss clears its orbs — keep it.
8. **AudioContext starts suspended**; `ensureAudio()` resumes it on first click/keydown. New audio won't play before a user gesture.
9. Bosses fire **regardless of line-of-sight** (bullet hell); normal enemies require `hasLineOfSight` so they don't shoot walls.

## localStorage keys
`tankbuster-high`, `tb-sfx`, `tb-music`, `tb-music-pitch`, `tb-death-pitch`.

## Pause-menu options
SFX volume, Music volume (sliders), Skip-track button, Enemy-Death-Pitch (Troll/Reset/Chipmunk), Music-Pitch (Troll/Reset/Chipmunk). All persisted.

---

## Change log (newest first — append after each major change: what changed + lesson)

_Append a tight bullet here whenever you ship something. Keep the "lesson" so future sessions don't repeat mistakes._

### 2026-07-04 (FP animation pass — "less like PNGs floating about")
- **World FX now exist in FP**: particles (explosions/muzzle smoke/dust) and damage floaters project into the first-person view with z-buffer occlusion — particles as ground-level puffs (cap 400), floaters rising in screen space from a frozen world origin (`f.wx/f.wy` added in `addFloater`; the top-down `f.y` drift is a world-axis move and would slide sideways in FP).
- **Everything animates**: infantry two-frame stride (`buildInfantrySprite(kind, frame)`, frame from `u.animDist`/14) + sine bob; vehicles/bot tanks chassis-rock; monkey lopes; **bosses spin** (billboard rotated by `spinAngle`); enemies + bot tanks draw an 8-point **muzzle-flash star** on their sprite when firing (`u.muzzleFlash` was already set generically in `fireWeapon`). `u.animDist` accumulates per-unit distance in the `updateUnits` FX loop.
- **Tank shells fly again in FP**: cannon-family + autocannon rounds project a tail point (`s.tail`) and render as an oriented shell (casing + dark tip) with a tracer streak from tail to head; own-bullet hide radius cut 90→45. Note: for off-axis shots the tail can project to the *right* of the head — that's correct perspective, not a bug.
- **Viewmodel sway**: barrel bobs with distance driven (`treadOffset`); crosshair and hull stay anchored to true centre (`scx`) because the crosshair is the actual aim.

### 2026-07-04 (FP art overhaul — "tanks should look like tanks")
- User verdict on FP v1: procedural boxes/blobs read as programmer art, world felt bare. Fix: **pre-render detailed sprites once to offscreen canvases** (`FP_ART.sprites` via `fpSprite(key, builder)`) and billboard those — detail is free per-frame. Lesson: never draw per-frame procedural "art" in FP; build a sprite canvas.
- **New sprites**: infantry with boots/vest/pouches/rifle-across-chest/face+helmet (`buildInfantrySprite`, variants: rpg tube, sniper scope, rusher red helmet), tank fronts with tracks+road wheels/glacis/dome turret/mantlet/muzzle-bore-at-viewer (`buildTankFront` — super, mega ×3 muzzles, bot tanks per team colour), humvee with windshield/grille/headlights/roof MG (`buildHumveeSprite`).
- **Environment**: 64×64 tiling wall textures per biome (`makeWallTexture`: sandstone blocks / snow-capped stone / vined jungle stone / timber logs / bolted concrete panels), texture-mapped per column via hit-point u-coordinate; map decorations now render in FP (tree/rock/hulk billboards via `buildDecorSprite` + `DECOR_FP_H` heights; dunes/craters/puddles as ground ellipses via `FLAT_DECOR_FILL`); parallax hill silhouettes (`makeHills`, 512-entry seamless array), per-biome sun + drifting clouds (`FP_SKY`, `makeClouds`), overcast bands on Fulda; player hull edge with vision blocks along the screen bottom. Per-map art cached in `buildFpMapArt()` (called from `buildMatch`).

### 2026-07-04 (first-person mode)
- **First-person camera** via pause-menu Camera buttons + V hotkey, persisted (`tb-view`). Raycaster over the untouched 2D sim: slab ray-vs-AABB per 2px column against `state.fpWalls` (map walls + border walls so rays always terminate), fisheye-corrected, fog toward each map's new `skyHorizon` colour; per-biome sky/floor gradients (desert dusk, pale snow, jungle haze, forest, overcast Fulda).
- **Billboard sprites** sorted far→near with a per-column z-buffer (centre-column occlusion test): boss/monkey use their images, infantry/vehicles/bot tanks are procedural front-facing silhouettes in their palette, bullets are glow orbs (boss orbs red, watermelon green, laser cyan), gems/flags/smokes included; elites keep gold rings, snipers show a red aim glow, HP bars float above sprites.
- **Viewmodel**: class-specific barrel(s) from the bottom edge (MBT muzzle brake, IFV twin cannons, laser emitter that goes violet plasma at 2+ damage stacks) with recoil kick, 8-point muzzle-flash star, and build themes (flames, arcs, HE bands, gold trim). Crosshair expands with recoil.
- **FP controls**: WASD view-relative + strafe, pointer-lock mouse-look (click to grab), arrow-key turn fallback; threat arrows become relative bearings (ahead = up) and skip the view cone. Lessons: (a) browser Esc exits pointer lock before the page sees it — pausing from locked FP is Esc-Esc or P, don't "fix" this; (b) hide the player's own bullets within 90 units or every shot whites out the screen; (c) sample-verify FP pixels away from the crosshair centre dot — it sits exactly at screen centre and contaminates wall-colour reads.

### 2026-07-03 (visual identity + firing feel pass)
- **Monkey audio fix**: screech now plays once on death only; watermelon throws use a new quiet synth `playFwip()` (bandpass noise sweep). User feedback: screech-on-every-throw was constant noise — don't re-attach screeches to the throw.
- **Firing feel**: barrel recoil + per-weapon muzzle flash (`recoil`/`muzzleFlash`/`muzzleWeapon` set in `fireWeapon`, decayed in the `updateUnits` FX loop; super/mega get it too), cannon rounds are now proper shells with smoke trails, impact/wall/kill shockwave rings.
- **Movement feel**: tread links scroll with distance (`treadOffset` per class spacing: 3/4/4/3.5/3.5 — keep link loop start at track edge minus offset), biome-coloured dust behind moving tanks.
- **Build-themed tank visuals**: burn=flames/embers, chain=hull arcs/sparks/bullet crackle, explosive=barrel bands+bigger flash+glow shells, crit=gold ring, vampire=crimson core, laser→violet plasma at 2+ damage stacks. Lesson: **particle emission must live in update functions, not render** — render runs while paused, so emitting there piles up frozen particles.

### 2026-07-03 (content + juice pass)
- **Marksman sniper enemy** (from 5:00): 0.9s standing telegraph with a red dashed laser-sight line drawn to the player, then a fast white tracer (520 px/s, 22 dmg, range 450 — respects the ≤460 range gotcha). Reuses the soldier render branch with a longer barrel + cyan scope. Lesson: the aim line is drawn in *world* coords, so it must be drawn **before** the `ctx.translate/rotate` in `renderEnemy` or it renders in local space.
- **Elite variants** past 8:00 (4%→22% chance): 3× HP/bounty, 1.12× speed, gold ring/arrow/minimap dot. Required fixing `onEnemyKilled` to use `enemy.bounty` (per-unit) instead of `def.bounty` (class) for score + gem value.
- **4 new upgrades** (pool now 17): Sabot Core (crit 10%/stack, 2×), Nano Salvage (3%/stack lifesteal via `playerLifesteal()`), HE Shells (primary-hit splash 25%/stack, 80px, via `explodeHE()` — restricted to cannon/autocannon/laser so MG spam can't proc it), Magnet Coil (+80 gem radius/stack — the old `p.upgrades.magnet` flag in `updateGems` was checked but never grantable; now stacks). **Thorns reworked** — it was dead code since melee removal; now reflects 15/stack onto shooters whose rounds hit the player.
- **Reroll button** on the level-up screen — 1 per screen (`state.rerollLeft`), `startUpgradePick` split into itself + `renderUpgradeCards()`.
- **Juice/HUD**: floating damage numbers (white/gold-crit/cyan-chain/orange-HE, capped 70, freeze on pause like particles), minimap bottom-right, low-HP vignette (<35%), death-screen run recap (kills/dmg/DPS via `state.stats` + build icon chips). Veteran YESKING fires one aimed orb per volley past 10:00.
- **Fixes**: exact segment-vs-AABB LOS (`segHitsRect`, slab method) replaces 20px sampling — kills known issue #1 and is cheaper; deleted dead `melee_light`/`melee_heavy` damage entries (#2).

### 2026-06-10 (AI pathing + balance pass)
- **Flow-field pathing for Onslaught enemies.** 40px BFS grid from the player, rebuilt every 0.3s in `updateOnslaught` (~2400 cells, cheap). Enemies (bosses too) follow it whenever they lose LOS, so they route *around* walls instead of grinding into them; with LOS they keep the old distance-band behavior, now with `steerClearOfWalls` whisker steering so retreat/strafe doesn't wall-hump. TDM/CTF bots got whisker steering too. Lessons: (a) initialize `FLOW.dist` to **-1**, not 0 — a zero-filled grid reads as "everywhere is the player's cell"; (b) when sampling 8 neighbors for descent direction, reject diagonals whose two orthogonal cells are blocked or units cut corners through walls; (c) the player hugging a wall sits inside the 16px inflated padding — hop the BFS seed to the nearest open cell.
- **Swarm separation + strafe variety.** `separateEnemies()` pairwise push-apart (bosses immovable) ends the single-blob stack; each enemy gets a random `strafeDir` that flips every 2–5s so the band orbit isn't synchronized. Re-resolve wall collision after the push pass — stunned enemies skip their own movement step and would stay clipped.
- **Class armor vs swarm fire (`ARMOR_MULT`).** Enemy damage is flat (`WEAPON_DAMAGE`), so the Scout (200 HP) was the tankiest class and the MBT (125 HP) nearly the squishiest — inverted class identity. Multiplier on damage *taken* from non-tank weapons: light 1.25 / medium 1.0 / mbt 0.6 / ifv 0.95 / laser 0.9 → effective swarm HP: mbt 208, light 160, medium 150, laser 144, ifv 111. `DAMAGE_TANK` (TDM/CTF) untouched.
- **Weapon rebalance vs enemies:** `mg` 14→8 (at 0.08s reload it was 175 DPS — out-damaging every primary as a free secondary; now ~100), `heavy_mg` 14→20 (MBT's identity weapon, ~111 DPS), `smg` 3→5 (rusher closes to 170px, should sting).
- **Boss HP scaling fixed** (known issue #2): yesking now scales off `state.matchTime` — ×(1 + 0.12/min past 5:00) → 1800 / 3960 / 6120 at 5/15/25 min. Final boss stays flat 14000 (it spawns at a fixed time). Confirmed known issue #1 (cannon missing from `WEAPON_DAMAGE`) was already fixed in code; cleared both from Known Issues.
- **Verified headless** by driving the update functions manually with fixed dt (1800+ frames, no errors; wall-flank test: a soldier spawned across a wall walked around it and re-acquired LOS). Lesson: the preview browser throttles `requestAnimationFrame` when hidden — `state.matchTime` stays 0 and nothing moves; don't mistake that for a bug, just call the update functions directly via eval.
- Added `.claude/launch.json` (python http.server 8123) matching the documented preview command.

### 2026-05-28
- **CLAUDE.md became the save file.** Added full current-contents inventory + known-issues section + this dated logbook so a fresh session can continue cold. Logged 2 latent bugs (cannon not in WEAPON_DAMAGE; boss HP scales off dead `state.wave`).
- **De-mirrored from Exercise3.** Exercise3 is graded coursework — removed all mirrored game files + references; it must stay tank-game-free. Preview is now standalone (static server here or the live Pages URL). **Never copy game files into Exercise3 again.**

### 2026-05-27 (the big build day)
- **Onslaught redesign** → continuous XP-pickup leveling (gems + magnet), time-scaled spawns, 5-min YESKING, 30-min final boss + endless free play; new upgrades: burn / chain-lightning / ricochet / stun-grenade. Lesson: gate XP/level logic to `mode.key === 'onslaught'`; guard `onEnemyKilled` with `if (!enemy.alive) return` (burn + bullet can kill same frame).
- **Watermelon monkey enemy** (sprite + 5 random screeches on throw/death).
- **YESKING boss** (spinning sprite, radial 4/5/6/8 orb patterns, entrance sound).
- **Audio pitch controls** in pause menu: enemy-death pitch + music pitch (Troll/Reset/Chipmunk), both persisted. Music pitch needs `preservesPitch=false`.
- **Pause menu mixer**: SFX/Music volume sliders + skip-track button. Line-of-sight gating so enemies don't shoot walls.
- **Real MP3 weapon sounds** + **Laser Phoenix** tank + MBT 50-cal secondary. Lesson: clip `duration` too short = silence (heavy_mg needed 0.5s, not 0.12s); full-auto = single-shot slice per round, not the burst file.
- **Balance/feel passes**: removed melee (all enemies ranged + pushed out of player footprint), removed screen shake, off-screen threat indicators, bullets fizzle when shooter dies, fixed stuck-WASD via `clearAllInput()`, slower/visible enemy projectiles, level-up jingle + "CONGRATS PATRIOT YOU WELFARED UP".
- **Earlier**: IFV class, pause, per-target damage table, 2400×1600 maps, terrain decorations, tank-class visual variants, desert background music + allahu soldier-death sound.

### 2026-05-26 (origin)
- Built from the Exercise-3 hobbies project as a separate "play my game" idea. Top-down tank game → grew into modes (Survival→Onslaught, TDM, CTF), 3 then 5 tank classes, 5 biome maps. Moved to its own repo + GitHub Pages (off Netlify to stop burning class build credits).
