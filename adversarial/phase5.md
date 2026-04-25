# Phase 5: Integration & Entry Point

## Goal
Wire everything into a runnable `main.py`, add integration tests, and define the benchmark positions used for validation throughout the loop.

---

## Directory Additions

```
adversarial/
├── main.py
├── test_adversarial.py
└── benchmark_positions.py
```

---

## Files to Create

### `main.py` — Entry Point
```python
#!/usr/bin/env python3
"""
Usage:
  python3 main.py [--iterations N] [--threshold F] [--verbose]

Options:
  --iterations N    Max loop iterations (default: 10)
  --threshold F     Min improvement fraction to count as progress (default: 0.02)
  --verbose         Print iteration details to stdout
"""
import argparse
from orchestrator import AdversarialOrchestrator
import os

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--iterations", type=int, default=10)
    parser.add_argument("--threshold", type=float, default=0.02)
    parser.add_argument("--verbose", action="store_true")
    args = parser.parse_args()

    api_key = os.environ["ANTHROPIC_API_KEY"]
    engine_dir = os.path.dirname(__file__)

    orchestrator = AdversarialOrchestrator(engine_dir, api_key)
    orchestrator.run_loop(max_iterations=args.iterations)

if __name__ == "__main__":
    main()
```

**Env var:** `ANTHROPIC_API_KEY` must be set. Exit with error message if missing.

---

### `benchmark_positions.py` — Validation Suite
30 positions used by `Orchestrator._benchmark_score()` to measure improvement across iterations.

```python
BENCHMARK_POSITIONS: list[dict] = [
    # --- Tactical (10 positions) ---
    {"name": "mate1_01",       "fen": "...", "expected": "...", "depth": 2, "category": "tactical"},
    {"name": "mate2_01",       "fen": "...", "expected": "...", "depth": 5, "category": "tactical"},
    {"name": "hanging_01",     "fen": "...", "expected": "...", "depth": 2, "category": "tactical"},
    {"name": "fork_01",        "fen": "...", "expected": "...", "depth": 3, "category": "tactical"},
    {"name": "pin_01",         "fen": "...", "expected": "...", "depth": 3, "category": "tactical"},
    {"name": "skewer_01",      "fen": "...", "expected": "...", "depth": 3, "category": "tactical"},
    {"name": "discovered_01",  "fen": "...", "expected": "...", "depth": 4, "category": "tactical"},
    {"name": "promo_01",       "fen": "...", "expected": "...", "depth": 2, "category": "tactical"},
    {"name": "deflect_01",     "fen": "...", "expected": "...", "depth": 4, "category": "tactical"},
    {"name": "decoy_01",       "fen": "...", "expected": "...", "depth": 4, "category": "tactical"},

    # --- Positional (10 positions) ---
    {"name": "rook_open_01",   "fen": "...", "expected": "...", "depth": 4, "category": "positional"},
    {"name": "bishop_pair_01", "fen": "...", "expected": "...", "depth": 4, "category": "positional"},
    {"name": "outpost_01",     "fen": "...", "expected": "...", "depth": 4, "category": "positional"},
    {"name": "pawn_chain_01",  "fen": "...", "expected": "...", "depth": 4, "category": "positional"},
    # ... (6 more)

    # --- Endgame (10 positions) ---
    {"name": "kpk_01",         "fen": "...", "expected": "...", "depth": 8, "category": "endgame"},
    {"name": "rook_end_01",    "fen": "...", "expected": "...", "depth": 6, "category": "endgame"},
    {"name": "opposition_01",  "fen": "...", "expected": "...", "depth": 6, "category": "endgame"},
    # ... (7 more)
]
```

Each position has a single correct move (verified by external engine like Stockfish offline). `_benchmark_score()` returns `correct / 30`.

---

### `test_adversarial.py` — Integration Tests
Run with `python3 -m pytest adversarial/test_adversarial.py -v`.

```python
def test_engine_uci_compliant():
    # spawn engine.py, send uci+isready, assert uciok and readyok in output

def test_engine_returns_legal_move():
    # send "position startpos\ngo depth 1", assert bestmove is in board.legal_moves

def test_attacker_finds_weaknesses():
    # run Attacker.run(iteration=0) against baseline engine
    # assert len(raw.results) > 0
    # assert sum(1 for r in raw.results if not r.passed) >= 3

def test_builder_produces_valid_patch():
    # create mock AttackReport with 2 weaknesses
    # call BuilderAgent.patch() (requires ANTHROPIC_API_KEY)
    # assert patch.code_changes keys are subset of PATCHABLE_FILES
    # assert all patched files pass syntax_validator

def test_orchestrator_single_iteration():
    # run orchestrator.run_iteration(0)
    # assert iterations.json exists and has 1 record
    # assert engine still responds to uci after iteration

def test_loop_terminates_on_max():
    # run orchestrator.run_loop(max_iterations=2)
    # assert len(tracker.load()) == 2

def test_rollback_on_worse_engine():
    # manually apply a bad patch (evaluation always returns 0)
    # verify orchestrator detects regression and rolls back
    # verify engine still works after rollback
```

---

## End-to-End Run

```bash
# Set API key
export ANTHROPIC_API_KEY=sk-...

# Run 3 iterations
cd adversarial
python3 main.py --iterations 3 --verbose
```

**Expected stdout:**
```
[Iteration 1] Attack: 8/20 tests failed. Weaknesses: 5 (confidence 0.82)
[Iteration 1] Build: patching evaluation.py, search.py (confidence 0.71)
[Iteration 1] Benchmark: 0.40 → 0.55 (+0.15) — IMPROVED
[Iteration 2] Attack: 5/20 tests failed. Weaknesses: 3 (confidence 0.78)
[Iteration 2] Build: patching evaluation.py (confidence 0.66)
[Iteration 2] Benchmark: 0.55 → 0.60 (+0.05) — IMPROVED
[Iteration 3] Attack: 4/20 tests failed. Weaknesses: 2 (confidence 0.70)
[Iteration 3] Build: patching search.py (confidence 0.60)
[Iteration 3] Benchmark: 0.60 → 0.58 (-0.02) — ROLLED BACK
Loop complete. Best benchmark score: 0.60 (iteration 2)
```

---

## Full Verification Checklist

- [ ] `echo -e "uci\nisready\nquit" | python3 engine.py` → `uciok` + `readyok`
- [ ] `echo -e "position startpos\ngo depth 3\nquit" | python3 engine.py` → legal `bestmove`
- [ ] `python3 main.py --iterations 1` completes without exception
- [ ] `iterations.json` has 1 entry after `--iterations 1`
- [ ] `python3 main.py --iterations 3` shows benchmark score ≥ baseline after iteration 2
- [ ] `python3 -m pytest test_adversarial.py -v` — all tests pass
- [ ] Final engine still UCI-compliant after 3 iterations (re-run UCI compliance test)
