# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file Sudoku game: `index.html` contains all HTML, CSS, and JavaScript. No build step, no package manager, no dependencies. `README.md` has player-facing instructions.

## Running / testing

There is no build or test tooling. To try a change, open `index.html` directly in a browser (e.g. `start index.html` on Windows, or just double-click it). Reload the page to pick up edits.

## Architecture

Everything lives in `index.html` in three inline sections: `<style>`, the body markup, and a single `<script>` block. There are no modules or external files to trace through — reading top to bottom of the script covers the whole app.

**Board and puzzle generation** (`emptyGrid`, `isSafe`, `shuffle`, `fillGrid`, `generateSolution`, `generatePuzzle`): a full valid solution is built with randomized backtracking, then cells are removed to produce the puzzle. The clue count (difficulty) comes from the `#difficulty` `<select>`.

**Game state** is a set of module-level globals, not encapsulated in an object: `solution` (the answer grid), `puzzle` (current grid, 0 = empty), `givens` (boolean grid marking fixed/uneditable cells), `notes` (per-cell array of pencil-marked candidate numbers, max `MAX_NOTES`), `selected` (`[row, col]` or `null`), plus mode flags `notesMode` and `showErrors`.

**Rendering is a single full-repaint function** (`render()`): on every state change it walks all 81 cells and recomputes each `<td>`'s text/notes and CSS classes (`given`, `selected`, `peer`, `same-num`, `error`, `correct`, `neutral`) from scratch — there is no incremental/diffed DOM update. Any new visual state should be added as a class computed inside this loop, with the corresponding CSS rule under `#board td.<class>`.

**Input flows through `enterNumber(num)`**, called from the on-screen keypad, the Erase button, and the `keydown` listener (digits 1-9, Backspace/Delete/0). It branches on `notesMode` (toggle a candidate in the cell's notes array, capped at `MAX_NOTES`) vs. normal mode (write to `puzzle` and clear notes). `showErrors` controls whether wrong/right entries get colored/red-flagged (`error`/`correct` classes) or shown neutrally, and separately gates which sound plays.

**3×3 box math**: box index for cell `(r, c)` is `Math.floor(r/3)*3 + Math.floor(c/3)`, stored as `data-box` on each `<td>` and used both by peer-highlighting logic in `render()` and by CSS (`#board td[data-box="..."]`) for the alternating box background tint. The thick grid-line dividers are done via `nth-child(3)`/`nth-child(6)` border rules, not by box grouping — if the grid size or box shape ever changes, both the JS math and these CSS selectors need updating together.

**Sound is synthesized, not sampled**: `playTone(freq, duration, type, volume, delay)` schedules a Web Audio oscillator+gain envelope; all SFX (`playPlaceSound`, `playErrorSound`, `playNeutralSound`, `playNoteSound`, `playEraseSound`, `playClickSound`, `playWinSound`) are thin wrappers around it with fixed frequencies. Background music (`MUSIC_MELODY`/`MUSIC_BASS` note arrays) is self-rescheduled via `scheduleMusicLoop` + `setTimeout` for the loop length, guarded by `musicPlaying`/`musicTimer`. `getAudioContext()` lazily creates/resumes the shared `AudioContext` (required by browser autoplay policy — must happen inside a user-gesture handler) and opportunistically starts music the first time it's called. The `soundEnabled` flag (toggled by `toggleSound()`) is checked inside `playTone` itself, so muting is a single early-return point.

## Conventions to preserve

- Keep the game dependency-free and in one file unless asked otherwise.
- New UI controls should follow the existing pattern: a button in `#top-toggles` or `#toolbar`, a click listener near the bottom of the script that calls `playClickSound()` then the handler, and any resulting visual state expressed as a class toggled inside `render()`.
