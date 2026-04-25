# Phase 3: Builder Agent — Code Patching

## Goal
Build the builder subsystem: a Claude API agent that reads an `AttackReport` plus the engine source, produces patched Python files, and a code patcher that safely applies those patches.

---

## Directory Additions

```
adversarial/
├── builder_agent.py
├── code_patcher.py
├── syntax_validator.py
└── builder_prompt.txt
```

---

## Files to Create

### `builder_agent.py` — Claude API Agent
Calls the Anthropic API to generate engine patches.

```python
class BuilderAgent:
    __init__(self, api_key: str, model: str = "claude-opus-4-6")
    patch(self, report: AttackReport, engine_source: dict[str, str], iteration: int) -> BuilderPatch
        # 1. Build prompt from report + source
        # 2. Call Claude
        # 3. Parse JSON response → BuilderPatch
        # 4. Run syntax_validator on each patched file
        # 5. Return BuilderPatch (or raise if validation fails)
```

**System prompt** (loaded from `builder_prompt.txt`):
```
You are a chess engine developer improving a Python UCI chess engine.
You receive:
1. An attack report listing weaknesses the engine exhibits.
2. The current source code of the engine modules.

Your job: produce improved source code that addresses the weaknesses.

Rules:
- Only modify files: evaluation.py, search.py, transposition_table.py, time_manager.py
- Do NOT modify engine.py (UCI protocol layer)
- All imports must be stdlib or python-chess only
- Output must be valid Python 3.11+
- Do not add dependencies beyond what is already imported

Respond ONLY with JSON matching this schema:
{
  "files_modified": ["evaluation.py"],
  "code_changes": {
    "evaluation.py": "...full file source..."
  },
  "changes_summary": "one-line description of changes",
  "reason": "which weaknesses this addresses",
  "confidence": 0.0
}
```

**User prompt construction:**
```
Attack report (iteration {N}):
Summary: {report.summary}
Overall confidence: {report.overall_confidence}

Weaknesses:
{for each weakness: type | description | FEN | engine_move | best_move | confidence}

Current engine source:
<evaluation.py>
{source}
</evaluation.py>
<search.py>
{source}
</search.py>
...
```

### `code_patcher.py` — File I/O
```python
def snapshot_engine_code(engine_dir: str) -> dict[str, str]:
    # reads evaluation.py, search.py, transposition_table.py, time_manager.py
    # returns {filename: source_code}

def apply_patches(engine_dir: str, patches: dict[str, str]) -> None:
    # for each filename in patches:
    #   validate filename is in allowed set
    #   write patches[filename] to engine_dir/filename
    # raises ValueError if filename not in PATCHABLE_FILES

PATCHABLE_FILES = {"evaluation.py", "search.py", "transposition_table.py", "time_manager.py"}

def restore_snapshot(engine_dir: str, snapshot: dict[str, str]) -> None:
    # write snapshot back (used for rollback)
    for filename, source in snapshot.items():
        write file
```

### `syntax_validator.py` — Static Analysis
```python
def validate_python(code: str) -> tuple[bool, str]:
    # uses ast.parse(); returns (True, "") or (False, error_message)

def validate_no_forbidden_imports(code: str) -> tuple[bool, str]:
    # walks AST, checks no import references LLMPlayer or PhasedEngine
    # returns (True, "") or (False, "imports forbidden module: ...")

def validate_patch(filename: str, code: str) -> tuple[bool, str]:
    # runs both validators above in sequence
    # returns first failure or (True, "")
```

### `builder_prompt.txt`
Stores the system prompt verbatim so it can be updated without changing Python code. Loaded by `BuilderAgent.__init__`.

---

## Patch Application Flow

```
BuilderAgent.patch(report, engine_source, iteration)
  ├─ build_prompt(report, engine_source)
  ├─ anthropic_client.messages.create(...)
  ├─ parse JSON from response.content[0].text
  ├─ for each file in code_changes:
  │    syntax_validator.validate_patch(filename, code)  # raises if invalid
  └─ return BuilderPatch(code_changes=..., ...)

Orchestrator calls:
  snapshot = code_patcher.snapshot_engine_code(engine_dir)
  patch = builder_agent.patch(report, snapshot, iteration)
  code_patcher.apply_patches(engine_dir, patch.code_changes)
  # ... validate; if worse: code_patcher.restore_snapshot(engine_dir, snapshot)
```

---

## Expected Improvements Per Weakness Type

| Weakness Type | Likely Builder Fix |
|---|---|
| `tactical` | Increase search depth; add null-move pruning; improve move ordering |
| `eval_error` | Add piece-square tables; mobility bonus; king safety |
| `positional` | Add passed-pawn bonus; rook on open file bonus; bishop pair |
| `time` | Fix time allocation logic in `time_manager.py` |

---

## Verification
1. Call `BuilderAgent.patch()` with a mock `AttackReport` (1–2 weaknesses); confirm `BuilderPatch.code_changes` contains valid Python
2. `syntax_validator.validate_python(patched_code)` returns `(True, "")` for all patched files
3. `code_patcher.apply_patches()` writes files; engine still responds to `uci`/`isready`
4. `code_patcher.restore_snapshot()` restores original files correctly
