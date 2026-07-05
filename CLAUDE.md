# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Terra Incognita is a globe-spinning geography game: a single, self-contained
`index.html` (no build step, no backend, no package install). Country
geometry is embedded directly in the file, so it works offline once loaded
and can be hosted on any static file server.

## Commands

There is no build/lint/test tooling — it's one HTML file.

- Run locally: open `index.html` directly in a browser, or serve it if the
  browser blocks local file access:
  ```bash
  python3 -m http.server 8000   # then visit http://localhost:8000
  ```
- Deploy: pushing to `beta` (the default branch) triggers
  `.github/workflows/static.yml`, which uploads the repo root as-is to
  GitHub Pages. No other CI exists.
- There is no automated test suite. Verify changes by loading the page and
  playing a round (see "How to play" in README.md).

## File structure

Everything lives in `index.html`:

- **Lines ~1–223**: `<style>` — all CSS (the "Liquid Glass" visual theme:
  frosted/blurred glass cards, gold/coral accent palette, starfield
  background).
- **Lines ~225–296**: page markup — masthead (title, difficulty buttons,
  score readout), the globe canvas, the "console" sidebar (target card,
  guess input, recent-guesses list), and the win overlay dialog.
- **Line 298**: a single line holding `const WORLD = {...}` — a GeoJSON
  `FeatureCollection` with 269 features: 174 countries (Natural Earth
  1:110m admin-0, public domain, simplified), plus the 50 U.S. states,
  13 Canadian provinces/territories, and 32 Mexican states, each set
  standing in place of that country's single whole-country entry (all
  sourced separately, simplified to a comparable point density). This
  line is very long (~500K characters); don't try to eyeball-diff it,
  and don't hand-edit it unless you're deliberately changing map data —
  regenerate it with a script instead. After any geometry change, sanity
  check with `d3.geoArea()` per feature (see "Verifying map data" below)
  before assuming it's fine — a visually-valid-looking polygon can still
  render as "fill nearly the whole globe" if a ring's winding is backwards.
- **Lines 300–860ish**: the game's JS, in one IIFE. This is the part you'll
  actually be editing for feature work. (Exact end line drifts as the game
  grows — it's whatever's left after the two `<script>` blocks.)

## Country feature schema

Each `feature.properties` in `WORLD` has:

```js
{
  name: "Afghanistan",       // canonical/short name, used as the internal key
  long: "Afghanistan",       // display name (may differ from `name`, e.g. long-form)
  cont: "Asia",              // continent, shown as an easy-mode hint
  subreg: "Southern Asia",   // subregion, shown as an easy-mode hint
  aliases: ["af","afg", ...],// accepted alternate spellings/abbreviations
  capital: "Kabul"           // optional; drives hard-mode capital follow-up
}
```

`feature.properties.name` (not `long`) is the key used in the `answered` /
`wrong` / `partial` sets and in `best`/localStorage bookkeeping — `long` is
purely for display. Because countries, U.S. states, Canadian provinces, and
Mexican states all share one flat feature list keyed by this name, two
subdivisions had to be disambiguated from countries of the same name:
`name: "Georgia (US)"` / `long: "Georgia"`, and Mexico's `name`/`long` are
both `"State of Mexico"` (not `"México"`, which collides with the country).
Every other subdivision has `name === long`. If you add more regions later,
check for this kind of collision the same way (compare against every
existing `name`/`long` before merging).

`regionNoun(feature)` maps `properties.cont` to the word used in prompts:
`"state"` for United States/Mexico, `"province"` for Canada, `"country"`
otherwise. Extend this if you add another subdivided country.

### Verifying map data

Whenever you touch `WORLD` geometry (adding/removing/re-simplifying a
region), verify with the real d3 build before trusting it — a `JSON.parse`
succeeding doesn't mean d3 will render it correctly. The failure mode that
actually happened: Virginia's fetched geometry included a degenerate
4-point collinear ring (a coastal-boundary artifact) wound backwards, so
d3 computed its area as ~4π steradians (the whole sphere) instead of ~0,
and it silently filled almost the entire visible globe every frame. Check:

```js
// same technique used to catch the Virginia bug — flag anything close to
// the whole-sphere area (4*PI ≈ 12.566); real regions are well under 1.0
WORLD.features.forEach(f => { if (d3.geoArea(f) > 1.0) console.log(f.properties.long); });
```

Also worth a syntax check and a full-rotation render sweep (call
`d3.geoPath` for every feature across many `rotate()` angles against a
stubbed canvas context) to catch exceptions/NaN before it reaches a
browser — there's no test suite, so this is the closest thing to one.

## Game architecture

All state and logic is in the single script IIFE starting at line 301.
Key pieces, in the order you'll likely touch them:

- **Rendering** (`draw`, `frame`, `resize`): d3-geo orthographic projection
  drawn on a 2D canvas. `frame` drives the auto-spin animation and re-draws
  every frame while spinning or while a target is pulsing. Land polygons are
  colored based on membership in `answered` / `wrong` / `partial` sets — no
  React-style re-render, just direct canvas painting each frame.
- **Picking** (`featureAt`): converts a canvas point to lon/lat via
  `projection.invert` and finds the containing feature with
  `d3.geoContains`. Used for both hover (when stopped) and click-to-mark.
- **Answer checking** (`normalize`, `compact`, `acceptedSet`, `isCorrect`,
  `isCapitalCorrect`): normalization strips accents/punctuation/a leading
  "the" and compares against a per-feature `Set` built from `name` + `long`
  + `aliases` (cached on `feature.properties._acc`).
- **Round flow** (`markTarget` → `submitGuess`/`submitCapitalGuess` →
  `finalizeRound`): a target is armed on click; guessing routes through
  difficulty-specific branches (easy gives one free hint before penalizing,
  hard requires a follow-up capital guess for full credit) before
  finalizing score/round bookkeeping and re-arming the UI for the next pick.
- **Difficulty** (`setDifficulty`, `loadDifficulty`): three modes — easy
  (region hint + one retry), medium (plain name guess), hard (name + capital
  for full 5 points, partial 3 points for name-only). Persisted to
  `localStorage` under `terraIncognita.difficulty` and triggers a full
  `resetGame()` on change.
- **High scores** (`loadBest`/`saveBest`): best score and best (fewest)
  winning-round-count are tracked **per difficulty** in
  `localStorage['terraIncognita.best']` as `{easy,medium,hard}` each with
  `{score, rounds}`.
- **Win condition**: reaching 100 points shows the win overlay
  (`showWin`); `resetGame()` (via "Play again" or a difficulty switch)
  clears round state but keeps persisted best scores.
- **Countdown timer** (`startTimer`/`clearTimer`/`tickTimer`/`handleTimeout`):
  each difficulty gets a fixed number of seconds (`DIFF_TIMERS`) to name the
  marked feature; running out is scored exactly like a wrong guess. Only
  runs during the name-guess phase — hard mode's capital follow-up is
  untimed (`clearTimer()` is called when entering `pendingCapital`).

## Conventions to preserve

- Keep the game dependency-free beyond the D3 CDN script tag and Google
  Fonts — no bundler, no npm packages.
- Keep all game logic in the single script block; don't split into
  multiple files (the project's whole value proposition is being one
  portable `index.html`).
- `feature.properties.name` is the stable identity key throughout (sets,
  localStorage, DOM lookups) — never key off `long` or array index.
- Wrap all `localStorage` access in try/catch (it's already done
  throughout) since the game must keep working with storage disabled.
