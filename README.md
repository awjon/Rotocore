# ROTOCORE

A browser brawler set inside a chamber that **rolls 90° on command**. Four rivals share one
tumbling room; grab the glowing core to freeze your clock, then roll the room until *your* base
is the floor and bank the time — or dump the core on an enemy's base to drain theirs. Last clock
still running wins.

Built with [three.js](https://threejs.org) (WebGL). No build step, no dependencies to install —
it's a single self-contained `index.html` that pulls three.js from a CDN at runtime.

---

## Play

**Desktop**

| Action | Keys |
| --- | --- |
| Move (camera-relative) | `W` `A` `S` `D` or arrow keys |
| Jump | `Space` |
| Push | `Shift` |
| Rotate the room | move your character into a glowing **side wall** |

**Mobile (portrait)** — touch controls appear automatically on touch devices:

| Action | Touch |
| --- | --- |
| Move | **drag anywhere** — a joystick springs up under your thumb (either hand) |
| Jump | **tap anywhere** |
| Push | the **PUSH button**, centered at the bottom (reach with your other thumb) |
| Rotate the room | steer your character into a glowing **side wall** |

Rotation is triggered by a **character touching a side wall** (you or an AI) — not by the control
itself — so it works the same on desktop and mobile. The room can only roll once every few seconds
(see the rotation meter).

You are **RED**. Three AI rivals hold the other three bases.

- **Carry** the cyan core and your timer freezes — everyone else keeps ticking down.
- **Roll** the chamber by running into a glowing side wall. Everyone tumbles to the new floor
  (brief landing stun). The carrier keeps the core through a tumble.
- **Score** by standing on whichever base is currently the floor while carrying:
  your color → **+25s** to you; an enemy color → **−20s** to that player.
- **Push** (`Shift`) sends a foe flying and **steals the core straight out of their hands** if they
  were carrying it — but airborne players are immune, so **jump to dodge**.
- Everyone starts with **60 seconds** and **3 lives**. Hit **0:00** and you lose a life and respawn;
  lose all three and you're eliminated. Last player standing wins. If you're knocked out you can
  spectate or skip to the finish.

Three AI tiers on the start screen: **Rookie**, **Rival**, **Ruthless** (Ruthless rolls the room
deliberately to bring its own base down).

### Stage codes (seeded layouts)

Every match builds its obstacle layout from a short **stage code** (e.g. `K3F9`). Leave the
**Stage Code** box on the start screen blank for a fresh random arena, or type a code to reproduce
an exact layout — same code, same stage, every time. The code also fixes which base starts on the
floor, so a stage is fully reproducible.

The active code is shown in the bottom-left during play and on the results screen, so you can note
a layout you liked and replay it. Codes are case-insensitive, and you can type your own (any text
works — `PORT`, `ARENA1`, a friend's name — it's hashed to a layout). The composition is fixed for
balance (a pair of jump-up platforms, one or two pillars, a few crates, sometimes a long bench);
only the positions, sizes, and a couple of optional pieces change between seeds.

---

## Deploy to Vercel

It's a static site, so any of these work. All files live in this folder.

**Drag-and-drop (fastest, ~1 min)**
1. Go to [vercel.com](https://vercel.com) → **Add New… → Project**.
2. Drop this whole folder (or just `index.html`) onto the upload area.
3. Deploy. That's the live URL.

**Git**
1. Push this folder to a GitHub/GitLab repo.
2. In Vercel, **Import** the repo. No framework, no build command, output dir = root.
3. Every push redeploys.

**Vercel CLI**
```bash
npm i -g vercel
cd this-folder
vercel        # preview deploy
vercel --prod # production
```

`vercel.json` is intentionally minimal: clean URLs and a no-cache header on the HTML so updates
show up immediately instead of being cached. You can delete it and the site still deploys fine.

---

## Tuning

Every gameplay value lives in the `CFG` object at the top of the `<script type="module">` block in
`index.html`. Current defaults:

| Key | Value | What it does |
| --- | --- | --- |
| `START_TIME` | `60` | Starting seconds per life |
| `LIVES` | `3` | Lives per player (0:00 spends one and respawns) |
| `SCORE_OWN` | `25` | Seconds gained delivering to your own base |
| `SCORE_ENEMY` | `20` | Seconds drained from an enemy whose base you deliver to |
| `GRAV` | `-42` | Gravity (more negative = heavier) |
| `MOVE` | `11` | Run speed |
| `CARRY_MOVE_MULT` | `0.88` | Speed multiplier while carrying the core |
| `JUMP_V` | `15` | Jump launch velocity |
| `PUSH_RANGE` | `3.4` | How close a push reaches |
| `PUSH_KNOCK` | `76` | Push knockback strength |
| `PUSH_STAGGER` | `0.8` | Seconds a pushed player is stunned |
| `PUSH_CD` | `0.55` | Push cooldown (seconds) |
| `ROLL_DUR` | `1.3` | Seconds a room roll takes |
| `ROLL_CD` | `4.0` | Cooldown between rolls (seconds) |
| `TUMBLE_STUN` | `0.7` | Landing stun after a tumble |
| `STEP_UP` | `0.55` | Ledges shorter than this can be walked up; taller need a jump |

AI behavior is in the `DIFFS` table just below the AI section:

```js
easy:   { speed:0.82, pushChance:0.25, dodge:0.15, smartRotate:false, react:0.32 }
normal: { speed:0.95, pushChance:0.55, dodge:0.45, smartRotate:false, react:0.20 }
hard:   { speed:1.0,  pushChance:0.85, dodge:0.8,  smartRotate:true,  react:0.12 }
```

- `speed` — movement multiplier vs. the player
- `pushChance` — likelihood of going for a steal when in range
- `dodge` — chance to jump away from an incoming push while carrying
- `smartRotate` — computes which slam panel brings its own base to the floor
- `react` — think interval in seconds (lower = sharper)

---

## How it works (architecture)

Single file, one module script. Roughly in order:

- **`CFG` / `TEAMS` / `BASE_ANGLES`** — all tunables and the four team colors/spawn faces.
- **Audio** — tiny procedural WebAudio (klaxon, slam, score stings, low-time tick). No audio files.
- **Renderer/scene** — `PCFSoftShadowMap`, ACES tone mapping, fog, one shadow-casting key light
  plus hemisphere + fill. One point light (on the core) to keep the light count low.
- **Chamber** — a `Group` whose pivot sits on the tube's central axis (`y = 15`). Four decorated
  faces hang off it, each with a glowing base pad and its own **seeded obstacle layout** (a tiny
  `mulberry32` PRNG seeded from the stage code; layouts are generated per match with rejection
  sampling so pieces never overlap each other or the base). The room "rolls" by tweening
  `chamber.rotation.z` by ±90°.
- **Rotation handoff** — during a roll, players and a free core are re-parented to the chamber so
  they ride the spin rigidly; at the end they're detached back to world space and gravity drops
  them onto the new floor. Gravity is always world −Y; only the geometry rotates, which keeps the
  physics simple and the floor/walls as fixed planes.
- **Floor detection** — whichever base anchor has the lowest world-Y is the active (scoring) base.
- **Collision** — capsule-vs-plane for walls/floor, plus box collision against only the *current
  floor face's* obstacles (stand on tops, get blocked by tall ones, auto-step over short ones).
- **AI** — greedy state machine (`seek` / `deliver` / `rotate` / `chase` / `roam`). No multi-step
  planning; Ruthless additionally picks the correct slam panel to bring its own base down.
- **Game loop** — fixed-ish timestep with a `timeScale` (the spectate "skip" runs it at 4×).

All art is procedural: geometry from three.js primitives and canvas-generated panel textures.
There are no external image, model, or audio assets.

---

## Browser support

Needs a modern browser with WebGL2 and ES-module `importmap` support (current Chrome, Edge,
Firefox, Safari), on desktop or mobile. Desktop uses keyboard; touch devices get on-screen
controls automatically (designed for portrait). On phones it caps the pixel ratio and uses a
lighter shadow map for performance.

## Not in this version

Moving hazards, a second arena, gamepad support, deeper AI planning, and local human multiplayer
were scoped out of v1. The rotation mechanic was the priority.

## License

No license is set. Add one (e.g. `LICENSE` file) before sharing publicly if you care how others
may reuse it.
