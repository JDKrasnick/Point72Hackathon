# Phase 4: Orchestrator — Adversarial Loop

## Goal
Wire the attacker and builder together into a persistent improvement loop with iteration tracking, rollback, and convergence detection.

---

## Directory Additions

```
adversarial/
├── orchestrator.py
├── iteration_tracker.py
└── config.py
```

Runtime-generated:
```
adversarial/
├── iterations.json     # created on first run
└── snapshots/          # engine code snapshots per iteration
    ├── v0/
    ├── v1/
    └── ...
```

---

## Files to Create

### `config.py` — Constants
```python
MAX_ITERATIONS = 10
IMPROVEMENT_THRESHOLD = 0.02   # min fraction of new weaknesses fixed to count as improvement
MIN_ATTACKER_CONFIDENCE = 0.6  # discard AttackReport if overall_confidence < this
MIN_BUILDER_CONFIDENCE = 0.5   # don't apply patch if BuilderPatch.confidence < this
STALL_ITERATIONS = 2           # stop if no improvement for this many consecutive iterations
VALIDATION_DEPTH = 5           # depth used for validation games
VALIDATION_POSITIONS = 10      # number of benchmark positions used for ELO estimate
ENGINE_TIMEOUT_S = 10.0        # max seconds per move during validation
PATCHABLE_FILES = {"evaluation.py", "search.py", "transposition_table.py", "time_manager.py"}
```

### `iteration_tracker.py` — Persistence
Reads/writes `iterations.json`. Each entry captures one full iteration.

```python
@dataclass
class IterationRecord:
    iteration: int
    timestamp: str
    attack_report: dict        # serialized AttackReport
    builder_patch: dict        # serialized BuilderPatch
    score_before: float        # fraction of benchmark positions solved before patch
    score_after: float         # fraction solved after patch
    improvement: float         # score_after - score_before
    status: str                # "improved" | "stalled" | "rolled_back" | "error"
    reason: str

class IterationTracker:
    __init__(self, path: str = "iterations.json")
    load(self) -> list[IterationRecord]
    append(self, record: IterationRecord) -> None
    last_n_improvements(self, n: int) -> list[float]
        # returns improvement values from last n records
    is_stalled(self) -> bool
        # True if last STALL_ITERATIONS records all have improvement <= 0
```

### `orchestrator.py` — Main Loop Controller

```python
class AdversarialOrchestrator:
    __init__(self, engine_dir: str, api_key: str)
        # init Attacker, AttackerAgent, BuilderAgent, IterationTracker

    run_loop(self, max_iterations: int = MAX_ITERATIONS) -> None
        # while iteration < max_iterations and not is_stalled():
        #     run_iteration(iteration)
        #     iteration += 1

    run_iteration(self, iteration: int) -> IterationRecord
        # 1. snapshot = snapshot_engine_code(engine_dir)
        # 2. save snapshot to snapshots/v{iteration}/
        # 3. raw = Attacker.run(iteration)
        # 4. report = AttackerAgent.analyze(raw, snapshot)
        # 5. if report.overall_confidence < MIN_ATTACKER_CONFIDENCE: skip iteration
        # 6. score_before = _benchmark_score()
        # 7. patch = BuilderAgent.patch(report, snapshot, iteration)
        # 8. if patch.confidence < MIN_BUILDER_CONFIDENCE: skip application
        # 9. apply_patches(engine_dir, patch.code_changes)
        # 10. if not _engine_still_uci_compliant(): restore_snapshot(); status = "rolled_back"
        # 11. score_after = _benchmark_score()
        # 12. improvement = score_after - score_before
        # 13. if improvement < 0: restore_snapshot(); status = "rolled_back"
        # 14. record = IterationRecord(...); tracker.append(record)
        # 15. return record

    _benchmark_score(self) -> float
        # run VALIDATION_POSITIONS positions from benchmark_positions
        # return fraction solved correctly (0.0–1.0)

    _engine_still_uci_compliant(self) -> bool
        # spawn engine; send uci + isready; check responses
```

---

## Loop State Machine

```
START
  │
  ▼
run_iteration(i)
  ├─ Attacker runs → AttackReport
  │    └─ confidence < MIN? → SKIP (log, continue)
  ├─ Builder runs → BuilderPatch
  │    └─ confidence < MIN? → SKIP (restore snapshot, continue)
  ├─ Apply patch
  │    └─ UCI compliance fails? → ROLLBACK → status="rolled_back"
  ├─ Benchmark
  │    └─ score worse? → ROLLBACK → status="rolled_back"
  │    └─ score same or better? → status="improved" | "stalled"
  ├─ Save IterationRecord
  └─ Check termination:
       ├─ iteration >= max_iterations → STOP
       └─ tracker.is_stalled() → STOP
```

---

## Rollback Mechanism
Snapshots stored at `adversarial/snapshots/v{iteration}/` before each patch application. `restore_snapshot()` copies files back from there.

---

## Convergence / Termination Conditions

| Condition | Check |
|---|---|
| Max iterations | `iteration >= MAX_ITERATIONS` |
| ELO plateau | `tracker.is_stalled()`: last `STALL_ITERATIONS` records all `improvement <= 0` |
| Builder stalls | `patch.confidence < MIN_BUILDER_CONFIDENCE` for 2 consecutive iterations |
| Attacker stalls | `report.overall_confidence < MIN_ATTACKER_CONFIDENCE` for 2 consecutive iterations |
| User interrupt | KeyboardInterrupt → log state, exit cleanly |

---

## `iterations.json` Schema
```json
[
  {
    "iteration": 1,
    "timestamp": "2026-04-25T14:00:00Z",
    "attack_report": { "...": "..." },
    "builder_patch": { "...": "..." },
    "score_before": 0.40,
    "score_after": 0.55,
    "improvement": 0.15,
    "status": "improved",
    "reason": "Added PST bonus for knights; now solves fork positions"
  }
]
```

---

## Verification
1. `AdversarialOrchestrator.run_iteration(0)` completes end-to-end (real or mocked Claude calls)
2. `iterations.json` exists and contains 1 record after one iteration
3. Manually corrupt `evaluation.py` (syntax error) → orchestrator rolls back and records `"rolled_back"`
4. Run 2 iterations where improvement=0 twice → `tracker.is_stalled()` returns True → loop stops
