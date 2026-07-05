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
- Deploy: pushing to `main` triggers `.github/workflows/static.yml`, which
  uploads the repo root as-is to GitHub Pages. No other CI exists.
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
  `FeatureCollection` with 226 features: 176 countries (Natural Earth
  1:110m admin-0, public domain, simplified) plus the 50 U.S. states in
  place of a single "United States of America" entry (states use the
  same feature schema, sourced separately at comparable simplification).
  This line is very long (~300K characters); don't try to eyeball-diff
  it, and don't hand-edit it unless you're deliberately changing map data
  — regenerate it with a script instead.
- **Lines 300–805**: the game's JS, in one IIFE. This is the part you'll
  actually be editing for feature work.

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
purely for display. Because countries and U.S. states share one flat
feature list keyed by this name, the state of Georgia is stored with
`name: "Georgia (US)"` / `long: "Georgia"` to avoid colliding with the
country of the same name; every other state has `name === long`. If you
add more regions later, check for this kind of collision the same way.

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
