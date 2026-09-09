# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A two-player sight-word bingo game for a child (Adele) and an adult caller (Nana). The entire app lives in a single **`index.html`** file — no build step, no dependencies to install, no package manager. Open the file in a browser or deploy it as a static page.

Real-time sync between the two players uses **Firebase Realtime Database** (compat SDK v10, loaded from CDN). The Firebase project is `adele-nana-bingo`.

## Running the app

Open `index.html` directly in a browser, or serve it with any static file server:

```
npx serve .
# or
python3 -m http.server
```

There are no tests, no lint step, and no build process.

## Architecture

### Single-file layout

The file is structured as: CSS (`<style>`) → HTML screens → inline `<script>`. Everything is global — no modules, no bundler.

### Screens (HTML `div.screen`)

Only one screen is `.active` at a time; `showScreen(name)` swaps them by toggling that class.

| Screen id | Who sees it | Purpose |
|---|---|---|
| `screen-welcome` | Both | Player selection |
| `screen-setup` | Nana | Word list / board size / pool mode |
| `screen-nana-waiting` | Nana | Displays session code while waiting |
| `screen-adele-join` | Adele | Code entry |
| `screen-adele-game` | Adele | Her bingo card |
| `screen-nana-game` | Nana | Current word + draw button + her own card |

### State

All mutable state lives in the single `local` object. Firebase is the source of truth; local copies are updated by `attachListener` callbacks.

Key fields:
- `local.calledWords` — array of all words drawn so far (newest first)
- `local.lastCalledWord` — the most recently drawn word
- `local.adeleCard` / `local.nanaCard` — 2-D arrays `[row][col]` of strings
- `local.adeleMarked` / `local.nanaMarked` — 2-D boolean arrays, same shape as the cards
- `local.gridSize` — 3 or 5 (determines board dimensions everywhere)
- `local.extraWords` — pool mode flag (see below)

### Firebase data shape (`sessions/<code>`)

```
adeleCard        // 2-D string array
adeleMarked      // 2-D bool array
nanaCard
nanaMarked
calledWords      // array, newest-first
remainingWords   // array (draw stack, pop from end)
lastCalledWord   // string | null
lastMarkedWord   // string | null  (written by Adele when she marks; Nana shows a toast)
winner           // 'Adele' | 'Nana' | null
adeleJoined      // bool
gridSize         // 3 | 5
listIdx          // 0–2
extraWords       // bool
```

Listeners are registered in `startAdeleGame` / `startNanaGame` and tracked in the `listeners` array so they can all be detached on new-game via `detachAllListeners()`.

### Game flow

1. Nana picks settings → `nanaCreateSession()` writes the full session to Firebase and waits on `adeleJoined`.
2. Adele enters the code → `adeleJoin()` sets `adeleJoined: true`; Nana's listener fires → `startNanaGame()`.
3. Nana calls `drawWord()` — pops from `remainingWords`, pushes to `calledWords`, updates `lastCalledWord`. Draw is blocked if Adele hasn't yet marked the last-called word that's on her card (`checkLookAgain`).
4. Adele taps a cell → `toggleAdeleCell(r, c)` — only succeeds if the word is in `calledWords` (or is the FREE square). Writes `adeleMarked` and `lastMarkedWord` to Firebase.
5. `checkBingo` tests all rows, columns, and both diagonals. A win writes `winner` to Firebase; both clients' listeners call `showWin`.

### Word pool modes

- **Exact mode** (`extraWords: false`): both cards are dealt from the same `boardWords`-sized pool, so every word that can be called is guaranteed to be on both boards.
- **Extra mode** (`extraWords: true`): each card picks from the full list independently; the draw pool includes all card words plus extras up to ~1.5× the board size.

### Card validation rule

Cells are only tappable when:
- the square is the FREE center square, **or**
- the cell's word appears in `local.calledWords`

Uncalled cells get the `.not-called` CSS class (50% opacity, default cursor).

### Word lists

Three static arrays in `WORD_LISTS` (Kindergarten 1, Kindergarten 2, First Grade), each with exactly 100 unique words. Each list must have ≥ `gridSize²` words. Add new lists by pushing another `{ name, words }` object into `WORD_LISTS`.
