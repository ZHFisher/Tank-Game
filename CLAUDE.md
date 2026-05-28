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

## Change log (append after each major fix: what changed + what not to repeat)

- **Onslaught redesign** → continuous XP-pickup leveling, time-scaled spawns, 5-min YESKING, 30-min final boss + free play; added burn/chain-lightning/ricochet/stun-grenade upgrades. Lesson: gate XP/level logic to `mode.key === 'onslaught'`; guard `onEnemyKilled` against double-kill (burn + bullet same frame) with an `if (!enemy.alive) return`.
- **De-mirrored from Exercise3** → Exercise3 is graded coursework and must stay tank-game-free. Removed all mirrored game files (`game.html`, audio, `music/`, sprites) and any references from that repo. Preview is now standalone (static server here or the live Pages URL). **Never copy game files into Exercise3 again.**
