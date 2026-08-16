# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A GitHub Pages site (`onelaview.github.io`) hosting self-contained kids' learning web games. There is no build system, no package manager, no tests, and no dependencies — every page is a single `index.html` with inline `<style>` and `<script>`. Nothing is transpiled or bundled; what is committed is what is served.

Deploy = push to `main`. GitHub Pages serves the repo root, so `bcc185-global/color-count-quest/index.html` is live at `/bcc185-global/color-count-quest/`.

There is currently no root `index.html`; `bcc185-global/index.html` is the hub landing page.

## Working on it

- Preview by opening the file directly (`open bcc185-global/color-count-quest/index.html`) or serving the root with `python3 -m http.server`. Speech synthesis and `localStorage` both work from `file://` in Safari/Chrome.
- Target device is a phone in portrait: the game page is `position: fixed; inset: 0` with `overflow: hidden`, sized in `vw` and `clamp()`. Any new UI must fit without scrolling the page — flex the stage, don't add height.
- Test icon and layout changes against edge cases baked into the data: the longest word (`backpack`), `white` (`#ffffff`) on a white card, and `black`. Several past commits exist purely because artwork disappeared on a white fill or two objects rendered identically.

## Architecture of the game page

`bcc185-global/color-count-quest/index.html` is one script with four layers:

1. **Data** — `COLORS`, `NUMWORDS` (zero…twenty), `OBJECTS`. Adding an entry to `OBJECTS` requires a matching key in `SVG`; every mini game samples these arrays, so one addition flows into all six games.
2. **Icons** — `SVG` maps an icon name to inner markup for a `0 0 100 100` viewBox. `iconSVG(name, hex, cls)` wraps it with `fill="currentColor"` and `color: <hex>`, so **the object's fill is the quiz's color answer**. Details that must stay visible regardless of that fill are drawn with explicit `#000000xx` / `#ffffffxx` overlays, or as a light halo stroke under a dark stroke (see `book`). Never hardcode a fill on the main body shape.
3. **Shared shell** — `screenFrame(title, onRepeat)` rebuilds `#app` (top bar + optional progress pill) and returns the screen container; `choiceButtons()` renders answer buttons and handles right/wrong styling, star award, and the callback; `speak()`, `burst()`, `addStar()`, `finish()`.
4. **Mini games** — `startListen`, `startWhats`, `startCount`, `startYesNo`, `startSpell`, `startAge`, registered in `MODES` (menu cards) and `START` (id → function).

### Round flow

`launch(fn)` resets `roundNum`, sets `quizMode`, and remembers `currentStart` for "Play again". Each mini game's `start*` renders exactly one question and calls `advance(sameStartFn)` after a correct answer; `advance` increments `roundNum` and either re-renders or shows `finish()` at `ROUNDS` (10). A mini game that is not a 10-question quiz opts out by setting `quizMode = false` and calling `finish({...})` itself — `startAge` does this, which is also why it is special-cased in `menu()` rather than going through `launch`.

Star count persists in `localStorage` under `ccq_stars`.

### Audio

`speak()` uses the Web Speech API. Mobile browsers only allow it after a user gesture, hence the `#start` overlay whose button calls `speak(" ")` before showing the menu — do not remove that unlock. Each screen speaks its prompt via `setTimeout(repeat, 350)` and exposes the same `repeat` through the 🔊 button in the top bar. Don't speak the answer before the child has picked it (a past commit removed exactly that).

## Architecture of Touch Math

`bcc185-global/touch-game-math/index.html` — Grade 1 addition/subtraction, one blank per equation, 4 choices, 10 questions, 30s each.

Unlike Color & Count Quest this page does **not** rebuild the whole screen per question: `#app` is static markup and `nextQuestion()` only refills `#eqEl` / `#choicesEl`, so the timer bar animates continuously across rounds.

- `makeQuestion()` picks the operands first and derives the result, which is what keeps every number inside `0..MAX_RESULT` and subtraction non-negative. Both operands are forced `>= 1` so no `n + 0` freebies. `seen` blocks a repeat within one game. Blank slot is 0/1/2 = a/b/result, weighted toward the result.
- `buildChoices()` returns the answer plus 3 near misses (±1, ±2, then ±3/±5/±10), clamped to range and de-duplicated.
- Timer is one `requestAnimationFrame` loop over `endsAt`/`totalMs`; the "+5s" power calls `addTime()` which bumps **both**, so the bar stays proportional instead of jumping past 100%.
- Scoring: `BASE_POINTS + SPEED_BONUS × whole seconds left`, doubled when `x2Armed`. Only a **first-try** correct answer scores — a wrong tap sets `q.missed`, disables that tile, and leaves the clock running so the child can still find the answer.
- Powers live in `game.powersLeft` (one use each per game) and are applied by the `POWERS` map; a handler returning `false` (50:50 with nothing left to remove) keeps the power unspent.
- `game.active` guards the queued `setTimeout(nextQuestion)` so leaving via 🏠 mid-round doesn't restart the timer behind the start screen.
- Persisted keys: `tgm_best` (high score), `tgm_sound` (speech on/off).

## Adding a game

Add a card to the `projects` array in `bcc185-global/index.html` and create `bcc185-global/<slug>/index.html`. Match the existing look: gradient `#6a4cff → #00c9c9`, white rounded cards with a hard `box-shadow` offset that collapses on `:active`, `Baloo 2 / Comic Sans MS` stack, emoji `data:image/svg+xml` favicon.

To add a mini game inside Color & Count Quest: write `startX()` that renders one round and calls `advance(startX)`, then register it in both `MODES` and `START`.

## Commits

Plain imperative subject, no Conventional Commits prefix ("Add a clock to the school icon", "Fix Spell It layout: slots wrapping off-screen"). Body explains what was wrong and what the fix draws/does, in prose.
