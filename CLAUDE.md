# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Single-file browser games. There's no build step, package manager or dependencies; each `.html` file holds its own HTML, CSS and JS. The only external resources are Google Fonts.

- `got-chess.html`: **Throne Chess**, the main project. It's Game of Thrones-themed chess: House Targaryen (white) against House Lannister (black).
- `tic-tac-toe.html`: a standalone tic-tac-toe game.
- `index.html`: a meta-refresh redirect to `got-chess.html`, so the GitHub Pages root opens the chess game.

## Git and deployment workflow

- Remote: public repo `Marvinfacio/throne-chess`, branch `main`.
- **GitHub Pages serves `main`.** Every push updates the live game at https://marvinfacio.github.io/throne-chess/ about a minute later. Don't push anything broken.
- The user wants every change committed with a clean message and pushed.
- Commit identity is set in the repo config: Marvinfacio, with the GitHub no-reply email. Don't change it.
- `.gitignore` excludes `*.ics` (a personal file in the working folder) and `.claude/settings.local.json`. Never force-add them.
- Git prints LF→CRLF warnings on Windows. They're harmless.

## Running and testing

- **Play:** open the file directly in a browser (`Start-Process got-chess.html`).
- **Browser automation:** the Claude-in-Chrome tools refuse `file://` URLs, and Python is not installed. Serve the folder with a small Node static server on `127.0.0.1` and open `http://127.0.0.1:<port>/got-chess.html`. App state is reachable from the console as top-level globals, e.g. `H`, `S`, `newGame()` and `tryMove(from, to)`.
- **Engine tests (no test files are committed):** the engine sits between the markers `/*ENGINE-START*/` and `/*ENGINE-END*/`, so Node can load it without a DOM:
  ```js
  const src = html.split('/*ENGINE-START*/')[1].split('/*ENGINE-END*/')[0];
  const E = new Function(src + '; return ENGINE();')();
  ```
  - Check move generation with perft. From the start position, depths 1–4 must give 20 / 400 / 8902 / 197281; use "Kiwipete" for castling, en passant and promotion edge cases.
  - Time `E.search(E.fromFEN(fen), level)` for each difficulty.
  - Re-run perft after any change to `gen`, `make`, `unmake` or `attacked`.

## got-chess.html architecture

### Engine: `function ENGINE()`
- **Fully self-contained.** At runtime `ENGINE.toString()` is turned into a Blob **Web Worker**, so code inside it must never reference anything outside the function: no DOM, no app globals.
- **Board:** a 64-element array. The index is `row*8 + col`, with row 0 = rank 8. Uppercase letters are white, lowercase are black, and `''` is an empty square.
- **State:** `{b, turn, castle, ep, half, full, kings}`.
  - `castle` is a bitmask: 1 = white kingside (K), 2 = white queenside (Q), 4 = black kingside (k), 8 = black queenside (q). The `CM` table clears rights when a piece moves from or onto a king or rook home square.
  - `kings` caches both king squares, so `make` and `unmake` must keep it current.
- **Moves:** `{from, to, promo?, flag?}`, where `flag` is `'d'` (double pawn push), `'e'` (en passant), `'k'` (kingside castle) or `'q'` (queenside castle).
- **Search:** `make` returns an undo record that `unmake` restores in place.
  - `gen` produces pseudo-legal moves, and legality is checked after `make`.
  - The search is negamax with alpha-beta, quiescence search and MVV-LVA ordering.
  - Iterative deepening runs within a time budget, and an unfinished depth is thrown away.
  - Difficulty is set in `LEVELS` (depth, quiescence on/off, random noise, blunder rate, time limit). `LEVEL_INFO` in the app holds the UI names and descriptions and must use the same keys.
- **Fallback:** if the Worker can't be created, `requestAI` runs `E.search` on the main thread.

### App layer (rest of the script)
- **`cfg` vs `G`:** `cfg` holds the persisted settings. `G` is a snapshot of mode, side and level taken when a game starts. Changing mode or difficulty mid-game only updates `cfg`; it is applied when a new game starts, or right away if the human hasn't moved yet.
- **Game state:** `S` is the live state. `H` is the history, `{u: undo record, san}`, and undo works by calling `E.unmake`. `keys` holds position keys for threefold repetition. `curLegal` is the cached list of legal moves.
- **AI cancellation:** `aiToken` is incremented by new game, undo and game end, and stale Worker replies whose token doesn't match are ignored. In vs-computer mode, undo removes two plies.
- **Ending a game:** all game endings go through `endGame()`, which records statistics once per game (`recorded`).
- **Rendering:** each render rebuilds all the HTML (`renderBoard`, `renderPanel`, `renderStats`). Board input uses pointer events on `#board` with pointer capture. Dragging uses a fixed-position `.ghost` element.

### Pieces and themes
- **Pieces:** `ART[side][type]` holds inline SVG portraits on a `0 0 100 100` viewBox. `CHAR` holds the character names.
- **SVG colour classes:** `.f` is the side's main colour and `.i` is its ink/outline colour (CSS vars `--fill` / `--ink`, from `--p1/--p1i` or `--p2/--p2i`). The other classes (`.s` skin, `.g` gold, `.m` metal, `.h1`–`.h3` hair, etc.) are fixed colours.
- **Outline classes:** shapes inside `g.o` get an ink stroke. Use `.ns` to remove it, `.ln` for unfilled detail lines and `.e` for eyes.
- **Themes:** `THEMES` sets the CSS variables `--light, --dark, --p1, --p1i, --p2, --p2i, --acc, --bg` through `applyTheme()`. The page UI is always dark by design. When adding or editing a theme, make sure dark pieces stay readable on dark squares.

### Persistence and audio
- **Storage:** `localStorage` keys `throneChess.settings` and `throneChess.stats`, accessed through the `store` helper (wrapped in try/catch). Stats are per browser.
- **Music:** `Music` is a Web Audio IIFE that synthesizes an original D-minor 3/4 loop (cello drone, plucked arpeggio, lead melody, drum, convolver reverb), plus sound effects. The music must stay original; never add the real Game of Thrones theme or any other copyrighted audio. Audio starts only after the first user gesture.
