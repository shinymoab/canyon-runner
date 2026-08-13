# Canyon Runner — project notes for Claude Code

Single-file iOS-web game: a third-person plane flies down procedurally
generated canyons, dodging rock. Built with Three.js r128, no build step,
no framework, no bundler.

## File & deploy shape

- Everything lives in one file: `index.html` (HTML, CSS, JS all inline).
  Keep it that way — don't split into modules, don't add a bundler, don't
  add npm/package.json unless explicitly asked. The whole point is that it
  runs by opening a single static file.
- `main` branch auto-deploys to GitHub Pages at
  https://shinymoab.github.io/canyon-runner/. Never commit straight to
  `main`. Always branch off `dev`, PR into `dev`.
- Test a `dev`-branch change on a phone via `raw.githack.com` or
  `htmlpreview.github.io` pointed at the raw `dev` URL — don't merge to
  `main` to test.
- One feature per branch/PR. Don't bundle an unrelated tweak into a PR
  because you noticed it while in there — flag it instead and ask.

## Units and conventions

- All distances are metres, all speeds are metres/second (HUD converts to
  km/h for display only — `Math.round(state.speed*3.6)`).
- Z increases "forward" (down the canyon). X is left/right. Y is up.
- Angles (yaw, pitch, roll) are radians, stored in `state`.
- User preference: metric units in anything user-facing (labels, docs,
  comments) — no imperial.

## Level data — `LEVELS` array

Each entry is one procedurally generated sector. Don't hand-author
geometry — everything is derived from these numbers by the functions
below the array (`centerX`, `floorY`, `halfWidth`, `ridge`).

- `seed` — must be unique per level (feeds `rng()`), don't reuse.
- `length` — sector length in metres.
- `speed` — base cruise speed in m/s.
- `hw` / `hwv` — canyon half-width and its variance.
- `a1/f1 … a4/f4` — amplitude/frequency pairs controlling canyon meander
  (`a1/f1`, `a2/f2`) and floor undulation (`a3/f3`, `a4/f4`). Higher
  frequency + higher amplitude = tighter, more technical flying.
- `density` — obstacle spacing multiplier (higher = more obstacles).
- `cave` / `caveH` — whether the sector has a roof and its height above
  the floor.
- `sky`, `horizon`, `fog`, `rock`, `sun` — visual palette for the sector,
  not gameplay.

New levels: add an entry, keep the naming/tone of existing level names
(terse, evocative, geology-flavoured — "Ochre Run", "Needle's Eye" — not
generic like "Level 6").

## Collision is math, not mesh

This is the thing most likely to get missed: collision does **not** use
Three.js raycasting or mesh intersection. It's checked analytically in
`checkCollision()` against:

- the canyon's generating functions (`centerX`, `floorY`, `halfWidth`) for
  walls/floor/roof, and
- the `obstacles` array (built in `buildCanyon()`, binned into
  `canyon.userData.bins` by z-position) for columns, spires, and
  boulders.

**Any new obstacle type must push a matching entry into `obstacles`** with
the right shape data (cylinder: `{t:0,x,z,r,y0,y1}`, sphere-ish:
`{t:1,x,y,z,r}`) — not just a visual mesh. A decorative rock with no
`obstacles` entry is invisible to collision and the plane will fly
straight through it. If a new obstacle shape doesn't fit the existing
cylinder/sphere checks in `checkCollision()`, extend that function
too, don't skip the check.

## Flight feel — tune carefully, flag for review

`YAW_RATE`, `PITCH_RATE`, `YAW_LIMIT`, `PITCH_LIMIT`, `BOOST_ADD`,
`BOOST_DRAIN`, `BOOST_REGEN` are hand-tuned by feel on a real phone, not
derived from anything. If a task involves changing these, call it out
explicitly in the PR description rather than folding it quietly into a
larger diff — these get reviewed by playtesting, not by reading the diff.

## HUD / UI

- HUD elements are plain DOM + CSS, not canvas-drawn — see `#hud` in the
  `<style>` block and the `$()`/`showHUD()`/`openPanel()` helpers.
- Design language: dark ground, amber (`--amber: #f5c45e`) as the single
  accent, condensed uppercase display type for headings, monospace for
  telemetry numbers. Keep new UI consistent with this rather than
  introducing new colours or fonts.
- Respect safe-area insets (`--pad-t/b/l/r`) for anything placed near a
  screen edge — this runs full-screen on notched iPhones via
  Add to Home Screen.

## Controls

- Touch: virtual joystick (left) for pitch/yaw, hold-to-boost button
  (right). Built with raw Pointer Events, not a library.
- Keyboard (desktop testing only): arrow keys / WASD + Shift to boost.
- `state.invertY` defaults to `true` (pull back = climb) and is
  user-toggleable from the title and sector-select screens — don't change
  the default without being asked.

## What not to do

- Don't add a build step, framework, or dependency beyond the Three.js
  r128 CDN script already in `<head>`.
- Don't rename or restructure `index.html` into multiple files.
- Don't change the default `invertY` value or `YAW_RATE`/`PITCH_RATE`
  without flagging it — these were tuned against real playtesting.
- Don't commit to `main` directly.
