# The Workflow Gambit
**Cubist Hackathon 2026 — 5-Minute Presentation**

---

## 00:00 – 00:30 | Hook *(Sachin)*

We couldn't agree on how to use AI to build this.

> *"Everyone uses AI — nobody agrees on how."*

We asked engineers. The answers ranged across the board:
- Some don't trust it. They're holding its hand the entire time.
- Some treat it like a senior dev. They just describe the problem and walk away.
- Some don't even read the output before shipping it. *(pause for laughter)*

So instead of arguing about it, we turned it into an experiment.

---

## 00:30 – 01:00 | Experimental Design *(Sachin)*

The controlled variable: a chess engine.

- **Why chess?** Objective win condition. Zero ambiguity in output quality.
- **Independent variable:** the AI workflow — how each of us chose to interact with Claude.
- **Standardization:** every engine started from the same base `claude.md`, the same UCI contract, and plugged into the same evaluation harness.
- One person built the harness while four others built engines in parallel. Nobody on the engine side touched the judge.

---

## 01:00 – 02:15 | The 9 Workflows *(JD)*

We grouped them into four philosophies:

### Basic Prompting
*Direct human-steered prompting — spectrum from least to most structured*

| Engine | The Idea |
|---|---|
| **Single Shot** | One prompt, one response. No iteration, no correction. |
| **Standard Prompting** | Iterative human steering, step by step, no chess context upfront. |
| **Phased Engine** | Discrete milestone phases — move gen, eval, search — each gated before the next begins. |

### Agent Harnesses
*Model operates with significant autonomy or through multi-agent architectures*

| Engine | The Idea |
|---|---|
| **Adversarial** | Two agents: one builds, one attacks. Output of each becomes input to the other. |
| **AutoResearch** | Set up a write → test → measure → improve loop, walk away, collect the result. |

### Indirect / Creative Approaches
*Sidestep the problem or simplify it first*

| Engine | The Idea |
|---|---|
| **Build Up (TicTacToe)** | Solve TicTacToe first, extend to checkers, then extend to chess. |
| **Random Features** | Generate a large set of outlandish ideas, let the model rank them, build the winner. |

### Data-Driven
*Front-load the model with knowledge rather than building from first principles*

| Engine | The Idea |
|---|---|
| **Stockfish Replica** | Study Stockfish's architecture yourself, explain it in plain language, let the model reimplement. |
| **Large Dataset** | Feed the model grandmaster games, openings, puzzles. Let pattern exposure drive the design. |

---

## 02:15 – 02:45 | Evaluation Metrics *(Ishan)*

Every engine is a black box. The harness only calls the UCI move interface.

**Chess Strength**
- Round-robin tournament — all 9 engines, 5 games per pair, both colors (40 games per engine)
- Double-elimination bracket — seeded from round-robin
- Stockfish benchmark — vs. Skill 0, 5, 10, 20

**Reliability**
- Timeout rate, crash rate, illegal move rate

**Workflow Cost**

```
Cost Comparison — Estimated API Spend vs. Round-Robin Rank
(■ = ~$0.15 | sorted cheapest → most expensive)
──────────────────────────────────────────────────────────
Engine             Cost Bar               $      Tokens   Time    Rank
──────────────────────────────────────────────────────────
Single Shot        ██░░░░░░░░░░░░░░░░░░  $0.33   45k     15 min   #3
Adversarial        ████░░░░░░░░░░░░░░░░  $0.58   80k     40 min   #1
Standard Prompt    ██████░░░░░░░░░░░░░░  $0.83  115k     25 min   #9
Build Up           ██████░░░░░░░░░░░░░░  $0.92  128k     30 min   #2
Stockfish Replica  ███████░░░░░░░░░░░░░  $1.08  150k     90 min   #6
Large Dataset      ████████░░░░░░░░░░░░  $1.12  155k     40 min   #8
Phased Engine      ████████░░░░░░░░░░░░  $1.21  168k     17 min   #5
Random Features    █████████████░░░░░░░  $1.87  259k    538 min   #4
AutoResearch       ████████████████████  $2.88  400k    150 min   #7
──────────────────────────────────────────────────────────
```

> **Key tension:** more spend ≠ better rank.
> Adversarial spent 8× less than AutoResearch and finished 6 places higher.

---

## 02:45 – 03:30 | Results *(Kim)*

### Power — Winner: Adversarial
Adversarial went **40/40** in the round robin — the only engine with zero losses. Its double-elim match against the #2 engine (BuildUp) ended in real checkmates at 139 and 110 plies, not timeouts. The critic-generator feedback loop is just structured code review: systematic discovery of failure modes that no single-pass approach ever finds.

*Honorable mention:* **BuildUp** ranked #2 at 35/40, and was the only engine to beat Stockfish at both Skill 5 and Skill 10 at 100%.

### Cost — Winner: Single Shot
$0.33, 45k tokens, 15 minutes, 1 prompt — 7–8× cheaper than the median engine, still placed 3rd.

### Human Involvement — Winner: Single Shot
1 prompt, 0 corrections, 15 minutes. Fire and forget — the software equivalent of submitting a job to a queue and walking away.

### Parallelization — Winner: Single Shot
Scored 5/5. You could run 100 of these concurrently, tournament them, pick the best. Stateless, no shared state, no sequencing dependencies — embarrassingly parallel.

### Code Complexity — Winner (simplest): Single Shot | Winner (most maintainable): Phased Engine
Single Shot is bounded by one LLM response — it physically cannot sprawl. Phased Engine's five milestone-gated phases enforce interface contracts the same way microservice boundaries do: explicit, reviewable, extensible.

---

## 03:30 – 04:00 | Verdict *(Hyun)*

**Best engine: Adversarial.**

Perfect round-robin record. Won the bracket via real checkmates. Cost only $0.58 and 40 minutes — second cheapest in the experiment. Making two agents adversarial is just structured code review. One builds, one attacks. Every weakness the critic finds forces the generator to produce something stronger. Every dollar spent on the critic paid off disproportionately.

**Best value: Single Shot.**

$0.33, 1 prompt, 3rd place overall. The floor of what a single unguided LLM call produces is higher than most engineers expect. Before building anything more elaborate, start here.

**The broader takeaway:**

More tokens, more time, and more prompts did not produce better results. Workflow structure mattered more than effort spent.

---

## 04:00 – 04:45 | Live Demo *(JD)*

*[Run the Adversarial engine vs. the Single Shot engine — live UCI match.]*

*[Show move output, time-per-move, and final result on screen.]*

---

## 04:45 – 05:00 | Close *(Sachin)*

We went in thinking this was a chess problem.

It turned out to be a workflow problem.

The chess engine was just the test harness.

**Thank you.**
