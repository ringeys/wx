# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

WeatherVibe (repo name "wx") — a single-page weather app showing current conditions, a 7-day forecast, an "outside score" heuristic, and hourly graphs.

## Running the app

There is no build system, no package manager, no test suite, and no dependencies. The entire app is [index.html](index.html). To develop:

- Open [index.html](index.html) directly in a browser, **or**
- Serve the directory with any static server (e.g. `python3 -m http.server 8000`) — preferred when testing geolocation, since some browsers gate `navigator.geolocation` to secure contexts.

When testing UI changes, exercise the golden paths in a browser: allow geolocation, search for a city, toggle the four detail modes (conditions / wind / humidity / precip), expand a day card, change settings, and verify both light and dark system themes (the `:root` and `@media (prefers-color-scheme: light)` CSS variable blocks must stay in sync).

## Architecture

Everything lives in one file ([index.html](index.html)) with three sections:

- **Inline `<style>`** ([index.html:12-782](index.html#L12-L782)) — design tokens are CSS custom properties on `:root`, with a parallel light-theme override under `prefers-color-scheme: light`. Canvas drawing reads these via `getComputedStyle(document.documentElement).getPropertyValue(...)`, so any new color used in graphs must exist in **both** theme blocks.
- **Body** ([index.html:784-793](index.html#L784-L793)) — a single `<div id="app">` mounting point with a fallback loading screen. Everything else is rendered by JS.
- **Inline `<script>`** ([index.html:795-1614](index.html#L795-L1614)) — all logic, ending with a top-level `init()` call that bootstraps geolocation → reverse geocode → forecast.

### State and rendering

There is one global `state` object ([index.html:799-814](index.html#L799-L814)). Every UI change follows the pattern: mutate `state`, then call `renderApp()` ([index.html:1160](index.html#L1160)), which rewrites `document.getElementById('app').innerHTML` from scratch using a single template literal. There is no virtual DOM, no diffing, no framework — full re-render every time. After re-rendering, any per-day canvases are drawn imperatively by `drawDayGraphs()` ([index.html:1027](index.html#L1027)).

Because the entire app is replaced on each render, **every event handler is wired through inline `onclick="funcName(...)"` strings inside the innerHTML template**. This means handler functions must be top-level (global) — do not move them into closures, modules, or `addEventListener` blocks unless you also rework the rendering model.

### Persistence

`localStorage` key `wx_times` stores `{ t1, t2, mode, tempLow, tempHigh }` ([index.html:817-825](index.html#L817-L825), [index.html:1546](index.html#L1546), [index.html:1560](index.html#L1560)). When adding a user-tunable setting, persist it through this same key — both the load block at the top of the script and the `setMode`/`applyTimes` write sites must be updated together.

### External APIs (no auth, no keys)

- **Open-Meteo forecast** — `api.open-meteo.com/v1/forecast` ([index.html:887](index.html#L887))
- **Open-Meteo geocoding** (city search) — `geocoding-api.open-meteo.com/v1/search` ([index.html:922](index.html#L922))
- **Nominatim** (reverse geocoding for "Your Location" label) — `nominatim.openstreetmap.org/reverse` ([index.html:950](index.html#L950))
- **weather.gov alerts** — `api.weather.gov/alerts/active` ([index.html:905](index.html#L905)). US-only; failures are swallowed and treated as "no alerts" so non-US users still get a working app.

Open-Meteo has historically returned daily fields under either `weather_code`/`weathercode` and `time`/`date`. The render path defensively reads both ([index.html:1168-1169](index.html#L1168-L1169), [index.html:1186-1187](index.html#L1186-L1187)). Preserve this pattern when touching daily/hourly field access.

### "Outside score"

`outsideScore()` ([index.html:976](index.html#L976)) is a hand-tuned 0–10 heuristic combining temperature distance from the user's `[tempLow, tempHigh]` comfort range, rain probability, wind, and WMO weather code. It is **the** product feature that distinguishes this from a generic forecast app — treat its weights as intentional, not arbitrary, and discuss before retuning.

The score is rendered in two places (current conditions and per-day cards) and color-coded via `scoreColor()` ([index.html:998](index.html#L998)). Both call sites read from `state.tempLow`/`state.tempHigh`.
