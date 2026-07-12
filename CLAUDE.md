# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single self-contained HTML file — `data-engineering-core-course.html` — that renders an interactive, 26-module data engineering interview-prep course with a built-in mock interview engine. There is no build step, no package manager, no test suite, and no framework. All CSS, JavaScript, content, and SVG diagrams live inline in that one file.

## Running it

Open the file directly in a browser:

```powershell
start data-engineering-core-course.html
```

The only external dependency is Google Fonts (Fraunces + JetBrains Mono) loaded via `<link>`; everything else works offline.

## Architecture

The `<script>` block has three sections, in file order: **content data** → **player/router** → **mock interview engine**.

### Content data

`TRACKS` is a flat list of 8 track groupings. `MODS` is 26 module objects, each conforming to this shape:

```js
{ id, num, track, title, tagline,
  analogy: {label, text},   // text may contain raw HTML
  stage: `<svg>…</svg>`,    // inline animated diagram, template literal
  body: `<p>…</p>`,         // raw HTML
  concepts: [{t, d}, …],    // exactly 4 — rendered in a 2-col grid
  qa: [{q, a, say}, …],     // feeds both the module page and the mock question bank
  oneLiner, pitfalls: [] }
```

`MODS` is assembled in pieces: a `const MODS = [...]` literal with four modules, then one `MODS.unshift(...)` for the foundations track and eight `MODS.push(...)` calls appending the rest. **Array position is therefore meaningless.**

### `ORDER` is the single source of truth for sequence

The `ORDER` array of module ids drives numbering, sidebar order, prev/next navigation, and home-page cards, via `orderedMods()` and `numOf()`. Consequences worth knowing before you edit content:

- **A module missing from `ORDER` is silently dropped** from every view — `orderedMods()` filters it out. But `MODS.length` (26) is still used for the progress denominator and the last-module check, so an unlisted module quietly corrupts progress math. Add to both.
- The `num:` field on each module object is **dead data**. Nothing reads it; displayed numbers come from `numOf()`. Don't trust it, and don't bother renumbering it.
- `renderModule()` derives the track label with `.name.split('·')[1]` — every `TRACKS` entry must contain a `·` separator or module pages throw.

### Rendering and routing

Everything is string-concatenated `innerHTML`. Content is intentionally treated as trusted raw HTML so authors can write `<strong>` inside analogy and answer text — note `esc()` is an identity function (`s => s`), a deliberate no-op, not an oversight. Only edit content you control.

Navigation is two delegated `click` listeners on `document`, dispatching on data attributes: `data-go` (route to a module / `home` / `mock`), `data-done` (toggle completion), `data-qa` (expand an answer), and a separate listener for the `data-mock-*` family. There is no URL/hash routing — state lives in the `current` variable.

Module completion is a plain in-memory `Set` and **resets on reload by design** — don't add persistence to it without asking. The one thing that *does* persist is the dark/light theme choice (`localStorage` key `de-theme`, set in `setTheme()`, read by `initTheme()` which falls back to the OS `prefers-color-scheme`).

### Mock interview engine

`QBANK` is derived at load time by flattening every module's `qa` array, so **adding a Q&A to a module automatically adds it to the mock exam** — the two never drift. Three modes: Quick Round (10 questions, stratified across tracks via `quickRound()` so a short round tests breadth, not one topic), Full Gauntlet (all questions shuffled), and Track Focus.

Scoring is self-reported: `nailed` = 1.0, `partial` = 0.5, `missed` = 0. Verdict thresholds are 85% (interview-ready) and 60% (almost there). Results show a per-track breakdown and chips linking back to weak modules.

## Diagrams and styling

Each module's `stage` is a hand-written inline SVG using a fixed `viewBox` and the shared animation classes defined in CSS: `dotflow` (flowing dashed line), `pulse`, `slide`, `rise`, `spin-slow`, `blink`, `fillup`. All are disabled under `prefers-reduced-motion`. Reuse these classes rather than adding per-diagram keyframes.

Colors come exclusively from the `:root` custom properties (`--teal`, `--bronze`, `--gold`, `--ink*`, `--paper*`, `--line`, `--danger`, `--ok`). SVGs hardcode the same hex values inline since CSS variables don't reliably reach SVG presentation attributes here — if you change a token, grep for the hex too.
