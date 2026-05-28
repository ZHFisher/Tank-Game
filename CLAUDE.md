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
- **Damage**: `DAMAGE_TANK[weapon][tankClass]` for tank-vs-tank; `WEAPON_DAMAGE[weapon]` flat for everything else. Bullet `speeds` dict + `life`/`radius` ternaries are in `makeBullet`.
- **Audio**: Web-Audio synth fallback (`playMG/playCannon/playRocket`) + real MP3 clips decoded to AudioBuffers (`SOUND_FILES`, `SOUND_OPTS`, `playClip`). Music uses an HTML5 `Audio` element.

## Onslaught mode (current design)

Vampire-Survivors / Disfigure style:
- **Continuous spawn**, time-scaled (cadence tightens, batch grows, enemy mix hardens over time). No waves.
- Enemies drop **cyan XP gems**; player collects them (magnet radius) → fills XP bar → **level up** → upgrade-pick card screen.
- **YESKING mini-boss every 5:00**; **FINAL BOSS at 30:00** (bullet hell); killing it unlocks endless **free play** with HP creep.
- HUD center: count-up timer `M:SS / 30:00`, then "FINAL BOSS" / "FREE PLAY"; plus `LVL n` and an XP bar.

## Current contents (save-file inventory — what exists right now)

**Controls**: WASD move · Mouse aim · LMB primary · RMB secondary · Q ability · P/Esc pause.

**Tank classes** (`TANKS`, player HP from `PLAYER_HP`):
| key | name | HP | speed | primary | secondary | ability (CD) |
|---|---|---|---|---|---|---|
| light | Scout | 200 | 160 | cannon 1.2s | MG | Dash, 6s |
| medium | Striker | 150 | 120 | cannon 0.9s | MG | Smoke, 12s |
| mbt | Iron Wall | 125 | 88 | cannon 0.6s | 50-cal heavy_mg 0.18s | Armor (2.5s invuln), 14s |
| ifv | Spearhead | 105 | 136 | autocannon 0.18s | rocket (homing+stun) 4.5s | Overdrive (fire×2 3s), 14s |
| laser | Phoenix | 130 | 130 | laser 0.45s (pierces 3) | MG | Overload (fire×2 3s), 14s |

**Onslaught enemies** (`ENEMIES`): soldier (30hp, pistol), rusher (25hp, fast, smg single-shot), monkey (80hp, throws watermelons, sprite `monkey.png`, 5 random screeches), rpg (40hp, slow homing missile that fizzles if you kill the trooper), humvee (120hp, MG burst), super (350hp, purple, heavy cannon), mega (900hp, pink, 3-shot volley), **yesking** mini-boss (1800hp, spinning `yesking.webp` sprite, radial 4/5/6/8 orb patterns, every 5:00), **finalboss** "YESKING ASCENDED" (14000hp, 80px, triple bullet-hell pattern, at 30:00).

**Modes** (`MODES`): onslaught (continuous XP survival — flagship), tdm (5v5 bots to 50 kills), ctf (3v3 bots, 3 captures).

**Upgrades** (`UPGRADES`, 13): damage, reload, hp, speed, ability-CD, regen, thorns, pierce, secondary-rate, **burn** (fire DoT), **chain** (lightning arc), **ricochet** (wall bounce), **stun** (periodic pulse).

**Audio files** (repo root): `meow.mp3` (ability, pitched per class), `cannon/ifv_cannon/ifv_rocket/laser/heavy_mg/mg.mp3` (weapons), `allahu/death_npc1/death_npc2/death_female.mp3` (soldier death pool, random), `monkey1-5.mp3` (monkey, random), `level_up.mp3`, `yesking.mp3` (boss entrance), `music/desert1-3.mp3` (desert bg). **Images**: `yesking.webp`, `monkey.png`.

## Known issues / TODO (not yet fixed)

1. **Player cannon/autocannon/mg/rocket aren't in `WEAPON_DAMAGE`.** Against non-tank Onslaught enemies (which have `enemyKind`, not `tankKey`), `damageFor` falls back to the default **20**. So the cannon does only 20 to soldiers/monkeys/etc., while `laser` (60) and `heavy_mg` (14) use their real table values. Net effect: cannon underperforms vs the main enemy roster; laser overperforms. Fix = add cannon/autocannon/mg/rocket to `WEAPON_DAMAGE` with intended values.
2. **Boss HP no longer scales.** `makeEnemy` scales boss HP by `state.wave - 5`, but the continuous Onslaught redesign never increments `state.wave` (it's time-based now). So bosses stay at base HP (yesking 1800, finalboss 14000). Free-play *enemy* HP creep (time-based, in `spawnContEnemy`) works fine. Fix = scale boss HP off `state.matchTime` instead of `state.wave`.

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
