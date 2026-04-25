# Phase 2: Attacker Agent — Weakness Detection

## Goal
Build the attacker subsystem: a programmatic test harness plus a Claude API agent that interprets failures and produces structured `AttackReport` JSON.

---

## Directory Additions

```
adversarial/
├── attacker.py
├── attacker_agent.py
├── engine_tester.py
├── test_positions.py
└── position_analyzer.py
```

---

## Files to Create

### `engine_tester.py` — Subprocess Harness
Spawns an engine process, sends UCI commands, parses output.

```python
class EngineTester:
    __init__(self, engine_path: str)
    run_position(self, fen: str, depth: int = 5, movetime_ms: int = None) -> str
        # returns bestmove in UCI format
    run_batch(self, positions: list[dict]) -> list[TestResult]
        # positions: [{"name": str, "fen": str, "expected": str, "depth": int}]
    check_uci_compliance(self) -> bool
        # sends uci/isready, checks responses
    close(self)

@dataclass
class TestResult:
    name: str
    fen: str
    expected_move: str
    engine_move: str
    passed: bool
    depth_searched: int
    time_ms: float
```

**Protocol detail:** after each `go`, read lines until `bestmove` line; parse first token after `bestmove`.

### `test_positions.py` — Curated Test Suite
Hard-coded list of positions that expose common engine weaknesses.

```python
TEST_POSITIONS: list[dict] = [
    # Mate in 1
    {"name": "mate1_back_rank", "fen": "6k1/5ppp/8/8/8/8/8/R5K1 w - - 0 1", "expected": "a1a8", "depth": 1},
    # Mate in 2
    {"name": "mate2_smothered", "fen": "...", "expected": "...", "depth": 4},
    # Hanging piece — queen blunder
    {"name": "hanging_queen", "fen": "...", "expected": "...", "depth": 2},
    # Fork — knight fork
    {"name": "fork_knight", "fen": "...", "expected": "...", "depth": 3},
    # Pin — bishop pin
    {"name": "pin_bishop", "fen": "...", "expected": "...", "depth": 3},
    # Skewer
    {"name": "skewer_rook", "fen": "...", "expected": "...", "depth": 3},
    # Promotion
    {"name": "promotion_queen", "fen": "...", "expected": "...", "depth": 2},
    # Endgame — opposition
    {"name": "kp_opposition", "fen": "...", "expected": "...", "depth": 6},
    # Stalemate avoidance
    {"name": "stalemate_avoid", "fen": "...", "expected": "...", "depth": 3},
    # Discovered check
    {"name": "discovered_check", "fen": "...", "expected": "...", "depth": 4},
    # ... (20+ total positions)
]
```

Each FEN is chosen so that any engine weaker than ~1200 ELO will likely fail it.

### `position_analyzer.py` — Dynamic Position Generator
Generates additional test positions beyond the curated set.

```python
def random_endgame_positions(n: int) -> list[str]
    # returns n FEN strings: K+pieces vs K+pieces, no pawns
def pawn_endgame_positions(n: int) -> list[str]
    # returns KPK, KPPK-type positions
def tactical_noise_positions(base_fen: str, n: int) -> list[str]
    # perturbs base_fen by making 2-4 random legal moves, returns variants
```

Uses `python-chess` to verify legality and filter out invalid/terminal positions.

### `attacker.py` — Test Orchestrator
Runs the full attack sequence and compiles raw results.

```python
class Attacker:
    __init__(self, engine_path: str)
    run(self, iteration: int) -> RawAttackData
        # 1. Check UCI compliance
        # 2. Run TEST_POSITIONS suite
        # 3. Run dynamic positions from position_analyzer
        # 4. Return all TestResults + metadata

@dataclass
class RawAttackData:
    iteration: int
    engine_path: str
    results: list[TestResult]
    compliance_ok: bool
    elapsed_s: float
```

### `attacker_agent.py` — Claude API Agent
Calls the Anthropic API to interpret raw test results and return a structured `AttackReport`.

```python
class AttackerAgent:
    __init__(self, api_key: str, model: str = "claude-opus-4-6")
    analyze(self, raw: RawAttackData, engine_source: dict[str, str]) -> AttackReport
        # Builds prompt with: test results + engine source snippets
        # Calls Claude with system prompt (see below)
        # Parses JSON from response
        # Returns AttackReport
```

**System prompt (attacker_agent.py constant):**
```
You are an adversarial chess engine tester. You receive:
1. A list of test results showing positions where the engine succeeded or failed.
2. The engine's source code.

Identify the root causes of failures. For each failure, classify the weakness type
("tactical", "positional", "eval_error", "time"), describe what went wrong in the
engine's logic, and rate your confidence (0.0–1.0).

Respond ONLY with a JSON object matching this schema:
{
  "weaknesses": [
    {
      "position_fen": "...",
      "weakness_type": "...",
      "description": "...",
      "engine_move": "...",
      "best_move": "...",
      "depth_needed": 0,
      "confidence": 0.0
    }
  ],
  "summary": "...",
  "overall_confidence": 0.0
}
```

**User prompt construction:**
```
Failed tests ({N} of {M}):
<list each failed TestResult as: name | FEN | expected | got | depth>

Engine source files:
<evaluation.py>
{source}
</evaluation.py>
<search.py>
{source}
</search.py>
```

---

## Data Flow

```
Attacker.run()
  └─ EngineTester.run_batch(TEST_POSITIONS + dynamic_positions)
       └─ [list[TestResult]]
            └─ AttackerAgent.analyze(raw, engine_source)
                 └─ Claude API → JSON → AttackReport
```

---

## Verification
1. `python3 -c "from engine_tester import EngineTester; t = EngineTester('engine.py'); print(t.check_uci_compliance())"` → `True`
2. Running the curated suite against the baseline engine fails at least 5 of 20 positions (confirms attacker is finding weaknesses)
3. `AttackerAgent.analyze()` returns valid `AttackReport` with `overall_confidence > 0`
4. JSON from `reporter.report_to_json(report)` parses back without error
