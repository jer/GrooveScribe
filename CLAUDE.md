# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Groove Scribe is a WYSIWYG drum-notation authoring tool and practice player that runs **entirely in the browser** — no backend, no server-side calls. It is plain ES5 JavaScript, HTML, and CSS with no module system and no package manager.

## Running and "building"

- **Run it**: serve the repo root over any static HTTP server and open `index.html` (e.g. `python3 -m http.server` then visit `localhost:8000`). Opening files via `file://` breaks soundfont/MIDI loading.
- **No build step** is required for development — `index.html` loads the unminified `js/*.js` directly. There is no npm/package.json, no bundler, no linter config, and no automated test runner (the README mentions tests aspirationally; none exist).
- **Minification** (optional, for deploy) is done by `js/RunYUIMinify.bat`, a Windows batch file that concatenates sources and runs `jstools/yuicompressor-2.4.8.jar`. The produced `*.min.js` files are gitignored. Paths in the `.bat` are hardcoded to the original author's machine.
- **Mobile**: `cordova/` wraps the app as a Cordova/PhoneGap mobile app (`cordova/www/` mirrors the web app).

## Architecture

The codebase is two logical apps sharing one core library. Everything attaches to a few **global singletons/constructors** (no modules): `GrooveUtils`, `GrooveWriter`, `GrooveDisplay`, and `grooves`.

### Core library — `js/groove_utils.js` (`GrooveUtils`, ~3400 lines)
The heart of the system; both apps instantiate `new GrooveUtils()`. Responsible for:
- Parsing/serializing groove state to and from URL query parameters (`getQueryVariableFromString`, `getGrooveDataFromUrlString`, `getUrlStringFromGrooveData`).
- Converting a groove into **ABC notation** and rendering it to SVG via the bundled `abc2svg-1.js` library.
- **MIDI playback**: builds MIDI files with `jsmidgen.js` and plays them through `MIDI.js/` + the General MIDI `soundfont/`; manages tempo, swing, metronome, looping, and note highlighting synced to playback.
- Each instance gets a `grooveUtilsUniqueIndex` used to namespace DOM element IDs, so **multiple grooves can be embedded on one page** without collisions.
- Defines the shared `constant_ABC_*` / `constant_OUR_MIDI_*` note constants referenced across files.

### Authoring app — "Groove Writer" (`index.html` + `js/groove_writer.js`, ~4600 lines)
The point-and-click editor. Renders an HTML grid of clickable note cells for the current subdivision, translates clicks into a note array, and calls `GrooveUtils` to regenerate sheet music and MIDI live. Owns the editing UX: undo/redo stacks, time-signature/subdivision changes, stickings, permutations, context menus, and instrument voices (hi-hat, snare, kick, toms).

### Display/embed app — "Groove Display" (`js/groove_display.js`, ~280 lines)
A thin wrapper exposing a function to embed a read-only groove (sheet music + MIDI player) in any page, with no authoring UI. Used by `GrooveEmbed.html` and `GrooveMultiDisplay.html`. Dynamically lazy-loads the scripts/CSS it needs.

### Preset library — `js/grooves.js` (`grooves`)
Plain data: named groove presets (Rock, Triplet, World, Ostinatos…) stored as URL-parameter strings, used to populate the in-app grooves menu.

## The groove data model (URL parameters)

A groove is fully described by URL query parameters — this is the serialization format, the sharing/permalink format, and how presets in `grooves.js` are stored. Key parameters:

- `TimeSig` (e.g. `4/4`), `Div` (subdivision: 8/16/12/24…), `Tempo`, `Measures`, `Swing`.
- Instrument voice strings, one character per grid cell: `H` (hi-hat), `S` (snare), `K` (kick), `T1`–`T4` (toms), `Stickings`. Characters encode both whether a note is present and its articulation (e.g. `x`/`o`/`O`/`g`/`r`/`-` for off). Voices are wrapped in `|...|` measure delimiters.
- Metadata: `Title`, `Author`, `Comments`. Modes: `Debug`, `GDB_Author` (Groove DB authoring mode).

When changing how grooves are parsed/generated, keep `getGrooveDataFromUrlString` and `getUrlStringFromGrooveData` (both in `groove_utils.js`) symmetric, and verify presets in `grooves.js` still load.

## Third-party libraries (vendored, in `js/` and top-level dirs)
`abc2svg-1.js` (ABC→SVG), `pablo.js`/`pablo.min.js` (SVG DOM), `jsmidgen.js` (MIDI file generation), `MIDI.js/` (Web Audio MIDI playback), `soundfont/` (instrument samples), `share-button.min.js`. Avoid editing these directly.

## Conventions
- ES5 only (`var`, IIFE module pattern, `"use strict"`); no transpilation. Match the existing style — singletons guarded by `if (typeof(X) === "undefined")`, `class_`-prefixed instance vars, `constant_`-prefixed constants, `root` alias for the instance/namespace.
- The many standalone `*.html` files at the repo root (`TestAllTimeSigs.html`, `WeirdTimeSigsTest.html`, `grooveDBTest.html`, etc.) are **manual** test/demo harnesses opened in a browser, not automated tests.
- License is GPL v2+. Keep the existing license header on source files.
