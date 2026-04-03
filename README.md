# ♛ ChessGenius — FAANG-Level Chess Engine

> **ALX Pre-Specialisation Project**  
> A single-file, zero-build chess game powered by a production-quality AI engine.

🎮 **[Play Live →](https://opriolani.github.io/ChessGenius/)**

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Difficulty Levels](#difficulty-levels)
- [Hint System](#hint-system)
- [Engine Architecture](#engine-architecture)
- [Project Structure](#project-structure)
- [Quick Deploy to GitHub Pages](#quick-deploy-to-github-pages)
- [Local Development](#local-development)
- [Libraries Used](#libraries-used)
- [How the AI Thinks](#how-the-ai-thinks)
- [Evaluation Function](#evaluation-function)
- [Author](#author)

---

## Overview

ChessGenius is a browser-based chess game built around a custom JavaScript AI engine. The project's focus is the **decision-making process** of the AI — how it evaluates positions, searches for the best move, and adapts its playstyle across six difficulty levels.

The entire project ships as **one HTML file** (`index.html`). There is no build step, no server, no package manager. Open it in any browser and it works. Everything — the game engine, the UI, the styles, the documentation, and the annotated source code — lives in that single file.

The application uses:
- [`chessboard.js`](https://chessboardjs.com/) for the drag-and-drop board GUI
- [`chess.js`](https://github.com/jhlywa/chess.js) for game rules, move validation, and FEN/PGN handling
- A **custom-written minimax engine** for all AI decision-making

---

## Features

| Feature | Description |
|---|---|
| ♟ **Playable chess game** | Drag-and-drop pieces, full legal move enforcement |
| 🧠 **FAANG-quality AI engine** | Minimax + Alpha-Beta + Quiescence Search + Transposition Table |
| 📐 **Positional evaluation** | Piece-Square Tables for all 6 piece types |
| 🏆 **6 difficulty levels** | Novice to Grandmaster — each with a distinct character |
| 💡 **6 hints per game** | Green highlights show the engine's recommended move |
| 📜 **Move history** | Full algebraic notation, auto-scrolling |
| 🎯 **Visual highlights** | Amber = last move, Red = check, Green = hint |
| ♙ **Captured pieces** | Live unicode piece display for both sides |
| 📱 **Responsive design** | Playable on desktop, tablet, and mobile |
| 📖 **Built-in README tab** | Project documentation inside the app |
| ⚙ **Built-in Source tab** | Fully annotated engine code inside the app |

---

## Difficulty Levels

Each level configures the **search depth** and an optional **randomness factor**. Lower levels inject random moves to simulate human imperfection. Higher levels run pure engine search.

| Level | Search Depth | Randomness | Character |
|---|---|---|---|
| **Novice** | 1 ply | 70% random | Learning — makes frequent blunders |
| **Beginner** | 2 ply | 40% random | Casual — occasionally tactical |
| **Intermediate** | 3 ply | 12% random | Mostly solid, small imperfections |
| **Expert** | 3 ply | Pure engine | Consistent — no random blunders |
| **Master** | 4 ply | Pure engine | Strong — thinks 4 moves ahead |
| **Grandmaster** | 5 ply | Pure engine | Elite — may pause before responding |

> **Ply** = one half-move. A depth of 4 ply means the engine looks 4 half-moves (2 full moves) ahead, evaluating thousands of positions before choosing.

---

## Hint System

Each new game starts with **6 hints**. Click the **Hint** button on your turn to:

1. Highlight the **recommended piece** in light green (the square to move from)
2. Highlight the **recommended destination** in bright green (the square to move to)
3. Display the move in algebraic notation in the status bar

The hint clears automatically after **3.5 seconds**. Hints always use Expert-level evaluation (depth 3, no randomness) regardless of the current game difficulty setting. Once all 6 hints are used, the button disables for the rest of the game.

---

## Engine Architecture

The AI is built on four interconnected algorithms that work together:

```
getBestMove()
    │
    ├── MVV-LVA Move Ordering         (sort best moves first)
    │
    └── minimax() with Alpha-Beta     (search the game tree)
            │
            ├── ttGet() / ttSet()     (transposition table cache)
            │
            └── qSearch()             (quiescence at leaf nodes)
                    │
                    └── evaluate()    (material + PST score)
```

### 1. Minimax with Alpha-Beta Pruning

Minimax is the foundational chess AI algorithm. It builds a **game tree** of all possible moves to a given depth, assuming both players play optimally — White tries to maximise the score, Black tries to minimise it.

**Alpha-Beta pruning** dramatically reduces the search space by tracking two values:
- `alpha` — the best score White can guarantee so far
- `beta` — the best score Black can guarantee so far

When `beta ≤ alpha`, the current branch is **pruned** — no matter what moves follow, the opponent already has a better option elsewhere and would never allow this position. This eliminates up to **60–80% of nodes** in a well-ordered search, effectively doubling usable search depth.

### 2. Quiescence Search

Without additional handling, the engine could stop searching just before a significant piece capture — evaluating a position as "+900cp" right before the queen gets taken. This is called the **horizon effect**.

Quiescence Search solves this by extending the search at leaf nodes to resolve all **capture chains** before calling the static evaluator. Only captures are examined (not quiet moves), with a safety cap of 5 extra plies to prevent infinite loops in endless exchange sequences.

The **stand-pat** principle applies: if the current static score already exceeds beta, we return immediately — we don't need to examine captures because the position is already too good for the opponent to allow.

### 3. Transposition Table

Chess positions are frequently reached via different sequences of moves (called **transpositions**). Without a cache, the engine would redundantly re-evaluate identical positions discovered through different move orders.

The Transposition Table is a `Map` keyed by the **FEN string** (Forsyth-Edwards Notation), which uniquely encodes:
- All 64 squares and their piece occupancy
- Active colour (whose turn it is)
- Castling rights (both sides, both directions)
- En passant target square
- Half-move clock (for fifty-move rule)

When a position is found in the table at the same or greater depth, the cached result is returned instantly. The table is capped at **80,000 entries** with LRU eviction (oldest entry removed when the cap is reached) to prevent unbounded memory growth.

In practice, the transposition table makes a **depth-5 search behave like depth-6 or 7** because entire evaluated sub-trees are skipped.

### 4. MVV-LVA Move Ordering

Alpha-Beta pruning is only as effective as the move ordering. In the worst case (random ordering), it reduces nothing. With **perfect ordering** (best move always searched first), it achieves the theoretical maximum speedup of O(b^(d/2)).

MVV-LVA (Most Valuable Victim — Least Valuable Attacker) scores each move before searching:

```
score = 10 × (captured piece value) − (attacker piece value)
      + 800  if promotion
      +  50  if check
```

This ensures captures of high-value pieces by low-value pieces are searched first (e.g., PxQ before QxP). Promotions and checks follow. Quiet moves come last.

---

## Evaluation Function

The static evaluator assigns a numeric score (in **centipawns**, where 100 = 1 pawn) to any board position:

```
score = Σ (material value + positional bonus) for each piece
```

Positive scores favour White; negative scores favour Black.

### Material Values (centipawns)

| Piece | Value | Rationale |
|---|---|---|
| Pawn | 100 | Baseline unit |
| Knight | 320 | Slightly less than bishop |
| Bishop | 330 | Slightly more — better in open positions |
| Rook | 500 | Worth 5 pawns |
| Queen | 900 | Worth 9 pawns |
| King | 20,000 | Exceeds all material — never sacrificed |

### Piece-Square Tables (PSTs)

Each piece type has a 64-value table encoding a positional **bonus or penalty** per square, adapted from [Sunfish.py](https://github.com/thomasahle/sunfish). Examples:

- **Pawns** get bonuses for advancing and controlling the center; heavy penalties for blocking the d/e pawns
- **Knights** get large bonuses in the center squares (e4, d4, e5, d5) and heavy penalties on the rim ("a knight on the rim is dim")
- **Bishops** get bonuses on long open diagonals
- **Rooks** get bonuses on the 7th rank (attacking the opponent's back rank)
- **Kings** get penalties in the center during the middlegame (exposed to attack)

A PST bonus can be up to ±50cp per piece, meaning **positional play matters as much as being up a pawn**.

### Terminal State Handling

Before material counting, the evaluator checks:
- **Checkmate** → returns ±99,999 (exceeds all material, so the engine always prioritises avoiding/delivering checkmate)
- **Draw, Stalemate, Threefold Repetition** → returns 0 (a dead-equal result)

---

## Project Structure

```
ChessGenius/
│
├── index.html              ← Entire project (game + docs + engine)
├── README.md               ← This file
├── LICENSE                 ← MIT License
├── .nojekyll               ← Tells GitHub Pages to skip Jekyll processing
├── .gitattributes          ← Git line-ending normalisation
│
└── .github/
    └── workflows/
        └── static.yml      ← GitHub Actions: auto-deploy to Pages on push
```

### What was removed from the original multi-file version

The original project had separate files that are now fully superseded by `index.html`:

| Removed | Reason |
|---|---|
| `js/chess.js` | Now loaded from CDN (zero maintenance overhead) |
| `js/main.js` | Engine rewritten and embedded in `index.html` |
| `css/main.css` | Styles rewritten and embedded in `index.html` |
| `img/chesspieces/` | Piece images now loaded from CDN |
| `.github/workflows/jekyll-gh-pages.yml` | Redundant — conflicts with `static.yml`; Jekyll is not needed for a static HTML file |
| `.DS_Store` | macOS metadata — should never be committed |
| `.README.md.swp` | Vim swap file — should never be committed |
| `.gitattribute` | Empty file with a typo (missing `s`) |

---

## Quick Deploy to GitHub Pages

This project auto-deploys via GitHub Actions on every push to `master`. The `.github/workflows/static.yml` workflow handles everything.

### First-time setup

**Step 1 — Create a GitHub repository**

Go to [github.com/new](https://github.com/new), name it `chess-genius` (or anything you like), set it to **Public**, and create it.

**Step 2 — Clone and push**

```bash
git clone https://github.com/YOUR_USERNAME/chess-genius.git
cd chess-genius

# Copy these files into the cloned folder:
# index.html, README.md, LICENSE, .nojekyll,
# .gitattributes, .github/workflows/static.yml

git add .
git commit -m "feat: ChessGenius FAANG engine"
git push origin master
```

**Step 3 — Enable GitHub Pages**

```
Repository → Settings → Pages
  Source: GitHub Actions   ← select this (not "Deploy from branch")
```

**Step 4 — Done**

GitHub Actions runs the `static.yml` workflow automatically. Your game is live at:

```
https://YOUR_USERNAME.github.io/chess-genius/
```

> The `.nojekyll` file is critical. Without it, GitHub Pages attempts to process the HTML with Jekyll, which can break CDN script imports and cause the page to render blank.

### Subsequent deployments

Every `git push origin master` triggers the workflow automatically. No manual steps needed.

---

## Local Development

No tools required. Just open the file:

```bash
# Option A — direct file open (works for this project)
open index.html

# Option B — local server (recommended to avoid browser CORS quirks)
python3 -m http.server 8080
# then visit http://localhost:8080

# Option C — Node.js
npx serve .
# then visit http://localhost:3000
```

---

## Libraries Used

All external libraries load from [cdnjs.cloudflare.com](https://cdnjs.cloudflare.com) at runtime — no local copies, no npm install.

| Library | Version | Purpose |
|---|---|---|
| [chess.js](https://github.com/jhlywa/chess.js) | 0.10.3 | Game logic, legal move generation, check/checkmate/draw detection, FEN/PGN parsing |
| [chessboard.js](https://chessboardjs.com/) | 1.0.0 | Drag-and-drop board UI, piece rendering, animations |
| [jQuery](https://jquery.com/) | 3.7.1 | Required peer dependency of chessboard.js; also used for square highlight manipulation |
| [Google Fonts](https://fonts.google.com/) | — | Playfair Display (headings) + JetBrains Mono (UI text) |

The AI engine, evaluation function, transposition table, and all chess-specific algorithms are **custom-written** — chess.js handles only rule enforcement, not any part of the AI.

---

## How the AI Thinks

Here is a simplified trace of what happens when you make a move:

1. **You drop a piece** → `onDrop()` is called
2. `chess.js` validates the move — if illegal, the piece snaps back
3. If legal, the move is applied and the UI updates
4. `engineMove()` is called after a 60ms delay (to allow the browser to repaint)
5. `getBestMove()` checks the difficulty level:
   - If randomness applies and `Math.random() < rand`, picks a random legal move
   - Otherwise, calls the search engine
6. `orderMoves()` sorts all legal moves by MVV-LVA score (best captures first)
7. For each move, `minimax()` is called recursively at `depth - 1`
8. Inside `minimax()`:
   - Check transposition table — if position already evaluated at this depth, return cached result
   - At depth 0, call `qSearch()` instead of `evaluate()` directly
   - `qSearch()` resolves all captures before calling `evaluate()`
   - `evaluate()` sums material + PST bonuses for all pieces
   - Alpha-beta pruning cuts branches that cannot improve the result
9. The move with the best returned score is played
10. `board.position()` updates the visual board
11. Move history, captured pieces, and status bar all refresh

Total time for a Grandmaster (depth 5) move: typically **0.5–3 seconds** depending on the position and the user's device.

---

## Author

Built by **Olaniyo Opiribo** as part of the **ALX Pre-Specialisation** programme.

The project demonstrates applied Computer Science concepts in a fully playable, deployable product:
- **Graph Search** — Minimax explores a game tree (a directed acyclic graph of board states)
- **Dynamic Programming** — Transposition table memoises sub-problem results
- **Heuristics** — Evaluation function approximates position quality without solving the game completely
- **Algorithm Optimisation** — Alpha-beta pruning, move ordering, and quiescence search reduce search space

---

*MIT License — see [LICENSE](./LICENSE)*
