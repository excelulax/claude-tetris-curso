# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **vanilla JavaScript Tetris game** with no dependencies, build process, or external frameworks. The game is playable directly in a web browser with all classic Tetris mechanics: piece rotation, wall kicks, soft/hard drops, ghost pieces, line clearing, scoring, and progressive difficulty levels.

The game is ~300 lines of JavaScript using HTML5 Canvas for rendering and `requestAnimationFrame` for the game loop.

## How to Run the Game

Choose one of two options:

### Option 1: Direct File Open
```bash
# Windows
start index.html

# macOS
open index.html

# Linux
xdg-open index.html
```

### Option 2: Local Server (Recommended)
Any static server works:
```bash
# Python 3
python3 -m http.server 8000

# Node.js
npx serve .

# PHP
php -S localhost:8000
```
Then open `http://localhost:8000` in your browser.

## Project Structure

```
03-tetris/
├── index.html      # DOM structure: main canvas, sidebar panel, overlay for paused/game-over states
├── style.css       # Dark theme styling with flexbox layout and backdrop blur effects
├── game.js         # All game logic (~305 lines)
└── README.md
```

## Game Controls

| Key | Action |
|-----|--------|
| `←` / `→` | Move piece left/right |
| `↑` or `X` | Rotate clockwise |
| `↓` | Soft drop (accelerated fall) |
| `Space` | Hard drop (instant fall) |
| `P` | Pause/resume |

## Architecture Overview

### State Model (`game.js` globals)
The game state is held in module-level variables:
- `board`: 2D array (ROWS × COLS) where each cell holds `0` (empty) or a color index (1–7)
- `current`: active piece object with `{type, shape, x, y}`
- `next`: preview piece object (same structure)
- `score`, `lines`, `level`: game metrics
- `paused`, `gameOver`: game states
- `dropInterval`: current fall speed in milliseconds (decreases as level increases)
- `dropAccum`: accumulator for frame-to-frame timing

### Core Game Loop
The game uses `requestAnimationFrame(loop)` driven by delta-time (`dt`):

1. **Accumulate time** in `dropAccum`
2. **When `dropAccum >= dropInterval`**: try to move piece down by one row
3. **If collision detected**: lock piece to board, clear lines, spawn new piece
4. **Call `draw()`**: render board, grid, ghost piece, current piece
5. **Schedule next frame**

This approach decouples piece-fall speed from browser frame rate (60 FPS → variable drop speeds).

### Key Functions

**Collision Detection** (`collide(shape, ox, oy)`)
- Checks if a piece (at offset x, y) overlaps board edges or locked blocks
- Returns `true` if collision, `false` if valid position
- Used by movement, rotation, and spawn checks

**Rotation** (`rotateCW(shape)`)
- Applies 90° clockwise rotation: transpose then reverse rows
- Returns new shape matrix (original unchanged)

**Wall Kick** (`tryRotate()`)
- Attempts rotation; if blocked, tries 5 horizontal offsets: `[0, -1, 1, -2, 2]`
- Allows pieces to "kick" off walls during rotation (standard Tetris behavior)
- Stops at the first valid kick position

**Line Clearing** (`clearLines()`)
- Iterates from bottom to top (ROWS-1 down to 0)
- Removes complete lines and inserts blanks at top
- Updates score and level (level increases every 10 lines)
- Recalculates `dropInterval` using formula: `max(100, 1000 − (level − 1) × 90)`

**Ghost Piece** (`ghostY()`)
- Finds the Y position where current piece will land
- Used by `draw()` to render semi-transparent preview (alpha = 0.2)

**Hard Drop** (`hardDrop()`)
- Moves piece instantly to ghost position
- Adds `(ghostY - currentY) * 2` points
- Calls `lockPiece()` immediately

**Soft Drop** (`softDrop()`)
- Moves piece down one row if possible
- Adds 1 point
- If blocked, calls `lockPiece()`

### Rendering

**Main Canvas** (`draw()`)
- Clears canvas and draws grid (light lines at 0.5px width)
- Renders board blocks (locked pieces)
- Renders ghost piece at 20% opacity
- Renders current piece at full opacity
- Each block is drawn with `drawBlock()` which adds a white highlight stripe

**Next Piece Preview** (`drawNext()`)
- Renders next piece in a 4×4 grid on a separate small canvas
- Centers the piece within the grid

### Piece Definitions

Pieces are stored as 4×4 matrices with `0` for empty, `1–7` for color indices:

```javascript
const PIECES = [
  null,
  [[0,0,0,0],[1,1,1,1],[0,0,0,0],[0,0,0,0]], // I (cyan)
  [[2,2],[2,2]],                               // O (yellow)
  [[0,3,0],[3,3,3],[0,0,0]],                  // T (purple)
  [[0,4,4],[4,4,0],[0,0,0]],                  // S (green)
  [[5,5,0],[0,5,5],[0,0,0]],                  // Z (red)
  [[6,0,0],[6,6,6],[0,0,0]],                  // J (indigo)
  [[0,0,7],[7,7,7],[0,0,0]],                  // L (orange)
];
```

### Scoring System

- **Line Clears**: `[0, 100, 300, 500, 800]` points × level
  - 1 line = 100 × level
  - 2 lines = 300 × level
  - 3 lines = 500 × level
  - 4 lines = 800 × level (Tetris bonus)
- **Soft Drop**: +1 point per row
- **Hard Drop**: +2 points per row

### Customization Constants

At the top of `game.js`:

| Constant | Default | Purpose |
|----------|---------|---------|
| `COLS` | `10` | Board width |
| `ROWS` | `20` | Board height |
| `BLOCK` | `30` | Pixel size per cell |
| `COLORS` | 7 hex values | Color palette |
| `LINE_SCORES` | `[0,100,300,500,800]` | Points per line |

**Note**: If you change `COLS`, `ROWS`, or `BLOCK`, update the canvas dimensions in `index.html` (`width` and `height` attributes must equal `COLS × BLOCK` and `ROWS × BLOCK`).

## Event Handling

All keyboard input routes through `document.addEventListener('keydown', ...)`:
- `P` key toggles pause (works from any state)
- During pause or game over, all other keys are ignored
- Movement/rotation/drop keys call movement functions and update HUD
- Hard drop prevents default spacebar behavior

## Testing & Verification

Since this is a visual game:
1. **Manual play-test**: Verify basic mechanics (movement, rotation, dropping, line clearing)
2. **Edge cases**: Test wall kicks, ghost piece positioning, hard/soft drops, pause/resume, restart
3. **Difficulty progression**: Confirm pieces fall faster as level increases
4. **Game over**: Verify game ends when a spawned piece collides immediately

No automated test suite exists; use the browser console or player observation for validation.

## Development Notes

- The code uses `'use strict'` to catch common mistakes
- No external libraries or bundlers—works in any modern browser with Canvas 2D API
- The game state is entirely in `game.js` globals; the DOM is only for display and input
- `updateHUD()` syncs DOM values with game state (score, lines, level)
- The overlay (paused/game-over) is shown/hidden by adding/removing the `hidden` class
