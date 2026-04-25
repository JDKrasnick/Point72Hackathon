# Phase 1: Foundation — Baseline Engine & Communication Protocol

## Goal
Establish the adversarial directory with a fully UCI-compliant baseline chess engine and the data structures agents use to communicate.

---

## Directory Structure

```
adversarial/
├── engine.py
├── search.py
├── evaluation.py
├── transposition_table.py
├── time_manager.py
├── communication.py
└── reporter.py
```

---

## Files to Create

### `engine.py` — UCI Protocol Handler
Entry point; reads UCI commands from stdin, writes responses to stdout (flushed after every line), logs to stderr.

```
class Engine:
    __init__(self)          # init board, search, time_manager
    run(self)               # main loop: readline → dispatch
    _send(self, msg)        # print + flush stdout
    _log(self, msg)         # print to stderr
    _handle_uci(self)       # → "id name ...\nid author ...\nuciok"
    _handle_isready(self)   # → "readyok"
    _handle_ucinewgame(self)
    _handle_position(self, tokens)
    _handle_go(self, tokens) # → "bestmove <move>"
```

**Baseline behavior:** depth-3 alpha-beta, material-only eval.

### `search.py` — Alpha-Beta Search
```
minimax(board, depth, alpha, beta, tt) -> (score, move)
iterative_deepening(board, max_depth, time_limit_ms) -> move
```
- Uses transposition table for cutoffs (TT_EXACT, TT_LOWER, TT_UPPER)
- Move ordering: captures first, then quiet moves

### `evaluation.py` — Static Evaluator
```
evaluate(board) -> int   # centipawns, positive = white advantage
```
Material values: P=100, N=320, B=330, R=500, Q=900, K=20000

### `transposition_table.py` — TT Cache
```
TT_EXACT = 0
TT_LOWER = 1
TT_UPPER = 2

class TranspositionTable:
    store(key, depth, score, flag, move)
    lookup(key, depth, alpha, beta) -> (score, move) | None
    clear()
```

### `time_manager.py` — Time Budget
```
class TimeManager:
    allocate(wtime, btime, winc, binc, movestogo, movetime, depth) -> (target_ms, max_ms)
```
- If `depth` given: ignore clock, return (None, None) and search to depth
- If `movetime` given: use that directly
- If clock given: use 1/30th of remaining time + increment

### `communication.py` — Agent Data Structures
```python
@dataclass
class Weakness:
    position_fen: str
    weakness_type: str      # "tactical" | "positional" | "time" | "eval_error"
    description: str
    engine_move: str        # UCI
    best_move: str          # UCI, empty if unknown
    depth_needed: int
    confidence: float       # 0.0–1.0

@dataclass
class AttackReport:
    iteration: int
    timestamp: str
    engine_version: str
    total_tests: int
    pass_count: int
    fail_count: int
    weaknesses: list[Weakness]
    summary: str
    overall_confidence: float

@dataclass
class BuilderPatch:
    iteration: int
    timestamp: str
    files_modified: list[str]
    code_changes: dict[str, str]    # filename -> full new source
    changes_summary: str
    reason: str
    confidence: float
```

### `reporter.py` — Serialization
```python
def report_to_json(report: AttackReport) -> str
def report_from_json(s: str) -> AttackReport
def patch_to_json(patch: BuilderPatch) -> str
def patch_from_json(s: str) -> BuilderPatch
```
Uses `dataclasses.asdict()` + `json.dumps/loads`.

---

## Verification
1. `echo -e "uci\nquit" | python3 adversarial/engine.py` → prints `id name ...`, `uciok`
2. `echo -e "isready\nquit" | python3 adversarial/engine.py` → prints `readyok`
3. `echo -e "position startpos\ngo depth 1\nquit" | python3 adversarial/engine.py` → prints `bestmove <legal UCI move>`
4. Round-trip `AttackReport` through `reporter.py` without data loss
