# The Workflow Gambit: A Controlled Evaluation of AI Development Strategies
**Cubist Hackathon 2026**

---

## The Finding

**Adversarial critic-generator loops produced the strongest engine. Single-prompt generation produced the best value. More tokens, time, and effort did not produce better results.**

---

## The Question

AI coding tools are everywhere. Nobody agrees on how to use them.

Not which model is best — that conversation is everywhere. The real debate is the *workflow*: how you structure your interaction with the model, how much context you provide, how much autonomy you give it, how you verify its output.

Everyone has opinions. Nobody has data.

So we ran an experiment.

---

## Experiment Design

One problem. One time budget. One shared interface. The only variable was the workflow used to build each engine.

Standardization was enforced three ways: a shared base `claude.md` that every builder started from, a fixed UCI (Universal Chess Interface) move contract every engine had to expose, and a tournament harness that neither knew nor cared how each engine was built — it only called the move interface.

One team member (Ishan) built the evaluation harness while the other four built engines in parallel. No engine builder touched the judge.

---

## The 9 Workflows

### Basic Prompting
*Direct human-steered prompting — the spectrum from least to most structured.*

| Engine | Owner | Hypothesis |
|---|---|---|
| Single Shot | Ethan | Minimal prompting produces a functional engine with no scaffolding needed |
| Standard Prompting | Sachin | Iterative human steering with no chess context provided upfront replicates a typical developer workflow |
| Phased Engine | JD | Breaking the build into gated milestones improves output quality and maintainability |

### Agent Harnesses
*Workflows where the model operates with significant autonomy or through multi-agent architectures.*

| Engine | Owner | Hypothesis |
|---|---|---|
| Adversarial | Ethan | A critic agent attacking the builder's output forces systematic improvement over single-pass generation |
| AutoResearch | JD | A hands-off write→test→measure→improve loop run overnight can match or exceed human-steered output |

### Indirect / Creative
*Workflows that approach the problem by simplifying it first or exploring unconventional design spaces.*

| Engine | Owner | Hypothesis |
|---|---|---|
| Build Up (TicTacToe) | Hyun | Solving a simpler game first produces a more coherent architecture when complexity is added incrementally |
| Random Features | Ethan | Generating a large set of outlandish ideas and letting the model self-rank them sidesteps standard assumptions |

### Data-Driven
*Workflows that front-load the model with existing knowledge rather than building from first principles.*

| Engine | Owner | Hypothesis |
|---|---|---|
| Stockfish Replica | Hyun-Dai | Studying Stockfish's architecture and explaining it in plain language produces a stronger engine than generic prompting |
| Large Dataset | Hyun | Priming the model with grandmaster games, openings, and puzzles produces an engine whose strength comes from pattern exposure |

---

## The Tournament Harness

The harness was the shared integration artifact that made the comparison fair. It knows nothing about how any engine was built.

**Design constraints:**
- All engines expose an identical UCI move interface
- Time controls are fixed and enforced externally (30s base + 0.5s increment)
- Illegal moves are auto-adjudicated as a loss, no manual review
- Results log automatically

**Tournament format:**
- **Round-robin:** all 9 engines play each other, 5 games per pair, alternating colors
- **Double-elimination bracket:** seeded from round-robin standings
- **Stockfish benchmark:** all engines play Stockfish at Skill 0, 5, 10, and 20

---

## Metrics

### Output Quality

| Metric | Type | Notes |
|---|---|---|
| Round-robin record | Objective | W/L out of 40 games per engine |
| vs. Stockfish baseline | Objective | Win % at Skill 0, 5, 10, 20 |
| Termination type | Objective | Checkmate vs. timeout vs. crash |
| Illegal move rate | Objective | Nonzero = disqualification |

### Workflow Cost

| Metric | Type | Notes |
|---|---|---|
| Total tokens | Objective | Input + output combined |
| Estimated API cost | Objective | USD at Sonnet pricing ($3/MTok in, $15/MTok out) |
| Human active time | Objective | Minutes at keyboard, timer-measured |
| Prompt count | Objective | Total prompts sent to the model |
| Human corrections | Objective | Times builder had to redirect the model |

### Prompting Quality (Self-Reported, 1–5)

| Metric | Scale |
|---|---|
| Prior knowledge required | 1 = none, 5 = deep domain + SWE expertise |
| Frustration score | 1 = completely smooth, 5 = very painful |
| Multitaskability | 1 = requires full attention, 5 = fully async |
| Parallelization | 1 = sequential only, 5 = massively parallelizable |

---

## Results

### Round-Robin Standings (40 games per engine, 9-engine field)

| Rank | Engine | Cluster | Score | Timeouts | Crashes |
|---|---|---|---|---|---|
| 1 | Adversarial | Agent Harness | **40/40** | 0 | 0 |
| 2 | Build Up | Indirect | 35/40 | 0 | 0 |
| 3 | Single Shot | Basic | 27/40 | 13 | 0 |
| 4 | Random Features | Indirect | 21/40 | 19 | 0 |
| 5 | Phased Engine | Basic | 20/40 | 20 | 0 |
| 6 | Stockfish Replica | Data-driven | 14/40 | 0 | 26 |
| 7 | AutoResearch | Agent Harness | 13/40 | 27 | 0 |
| 8 | Large Dataset | Data-driven | 6/40 | 34 | 0 |
| 9 | Standard Prompt | Basic | 4/40 | 36 | 0 |

### Double-Elimination Bracket (Winners Round 1)

| Match | Winner | Score | Termination Note |
|---|---|---|---|
| Adversarial vs. Build Up | **Adversarial** | 2–0 | Both games ended in checkmate (139 plies, 110 plies) |
| Single Shot vs. Random Features | **Single Shot** | 2–0 | Random Features timed out both games |
| Phased Engine vs. Stockfish Replica | **Stockfish Replica** | 1–1 (tiebreak) | Phased timed out g0; Stockfish crashed g1 |
| AutoResearch vs. Large Dataset | **AutoResearch** | 1–1 (tiebreak) | Both timed out in regular play; Large Dataset timed out tiebreak |

### Stockfish Benchmark (Win % by Skill Level)

| Engine | Skill 0 | Skill 5 | Skill 10 | Skill 20 | Overall Avg |
|---|---|---|---|---|---|
| Single Shot | 100% | 100% | 0% | 0% | **50%** |
| Build Up | 0% | 100% | 100% | 0% | **50%** |
| Phased Engine | 50% | 50% | 50% | 0% | **37.5%** |
| Standard Prompt | 50% | 0% | 50% | 0% | **25%** |
| Adversarial | — | — | — | — | Not benchmarked |
| Random Features | — | — | — | — | Not benchmarked |
| Stockfish Replica | — | — | — | — | Not benchmarked |
| AutoResearch | — | — | — | — | Not benchmarked |
| Large Dataset | — | — | — | — | Not benchmarked |

### Workflow Cost Summary

| Engine | Tokens (k) | API Cost | Human Time | Prompts | Corrections |
|---|---|---|---|---|---|
| Single Shot | 45 | $0.33 | 15 min | 1 | 0 |
| Adversarial | 80 | $0.58 | 40 min | 8 | 4 |
| Standard Prompt | 115 | $0.83 | 25 min | 7 | 3 |
| Build Up | 128 | $0.92 | 30 min | 1 | 1 |
| Stockfish Replica | 150 | $1.08 | 90 min | 1 | 1 |
| Large Dataset | 155 | $1.12 | 40 min | 2 | 3 |
| Phased Engine | 168 | $1.21 | 17 min | 5 | 2 |
| Random Features | 259 | $1.87 | 538 min | ~15 | 8 |
| AutoResearch | 400 | $2.88 | 150 min | ~20 | 2 |

### Prompting Quality Scores (Self-Reported, 1–5)

| Engine | Prior Knowledge | Frustration | Multitaskability | Parallelization |
|---|---|---|---|---|
| Single Shot | 1 | 1 | 5 | 5 |
| Standard Prompt | 2 | 2 | 2 | 2 |
| Phased Engine | 3 | 2 | 3 | 2 |
| Build Up | 2 | 2 | 3 | 2 |
| Adversarial | 3 | 3 | 3 | 2 |
| AutoResearch | 2 | 2 | 5 | 4 |
| Stockfish Replica | 5 | 4 | 1 | 1 |
| Large Dataset | 4 | 3 | 3 | 3 |
| Random Features | 3 | 4 | 4 | 3 |

---

## Analysis by Dimension

### Power
**Winner: Adversarial.** The only engine with zero losses across 40 round-robin games. Its double-elim win over BuildUp via actual checkmates — not timeouts — is the clearest signal of genuine chess quality in the dataset. The critic-generator loop is structurally analogous to test-driven development: every attack the critic makes is a failing test case the generator must fix. Iterative adversarial pressure forced the engine to address failure modes that single-pass generation cannot anticipate.

**Runner-up: Build Up.** Finishing 35/40 with a 50% average Stockfish win rate (including 100% at Skill 5 and 10) is the second-strongest result. Incremental complexity scaffolding — solving a simpler problem first and inheriting the working structure — produced the most coherent implementation after Adversarial.

### Cost
**Winner: Single Shot.** $0.33, 45k tokens, 15 minutes, 1 prompt. It is not close. Single Shot is 8.7× cheaper than the median ($0.92) and 4.4× cheaper than the field average ($1.48). It finished 3rd in the round robin. The marginal cost of any additional prompting approach must be justified against this baseline — and in most cases the data shows it cannot be.

### Human Involvement
**Winner: Single Shot.** One prompt, zero corrections, 15 minutes of active time. AutoResearch is the philosophical alternative — only 2 corrections and fully async once running — but required 20 prompts and 150 minutes to initialize the loop. Single Shot requires no chess knowledge, no iteration, and no monitoring. It is the only approach that is genuinely zero-attention after the first prompt.

### Parallelization
**Winner: Single Shot.** Scored 5/5 on both multitaskability and parallelization. Because Single Shot is stateless (one prompt → one response), it is trivially parallelizable: run 20 instances concurrently, hold a mini-tournament among them, ship the winner. AutoResearch (4/5) has internal parallelization but is a single opaque process. Every other approach has sequential dependencies that prevent parallel execution.

### Code Complexity / Maintainability
**Least complex: Single Shot.** Bounded by a single LLM context window — it cannot sprawl. No refactoring cycles, no accumulated patches, no dead code from iterative revision.

**Most maintainable: Phased Engine.** Five milestone-gated phases with accompanying documentation (phase1.md–phase5.md) enforce explicit interface contracts between modules. Each phase is independently reviewable. This is the closest any approach came to production-quality software architecture — analogous to service boundary design in a real codebase.

**Highest complexity risk: AutoResearch and Random Features.** AutoResearch accumulated 400k tokens of iterative patches over 2.5 hours; Random Features explored for 9 hours with 8 human corrections. Both are high-risk for spaghetti output that passes functional tests but is unreadable.

---

## Verdict

### Best Engine: Adversarial

Perfect round-robin record (40/40). Won the double-elim bracket via real checkmates. Cost $0.58 and 40 minutes — second cheapest in the field. The ROI on the adversarial approach is the most striking number in the experiment: it produced the best output at near-minimal cost. Structured adversarial pressure is the most efficient quality multiplier we found.

### Best Value: Single Shot

$0.33, 1 prompt, 3rd place. If the question is "what is the cheapest way to get a functional chess engine from an LLM," the answer is one prompt. The floor of what a single unguided LLM call produces is higher than most engineers expect.

### Rankings

| Rank | Engine | Strongest Dimension |
|---|---|---|
| 1 | Adversarial | Power — 40/40 round robin, won by checkmate |
| 2 | Build Up | Power #2 — best Stockfish benchmark in field |
| 3 | Single Shot | Cost, Human Involvement, Parallelization |
| 4 | Phased Engine | Maintainability — best code structure |
| 5 | Random Features | Exploration — widest design space covered |
| 6 | Stockfish Replica | Domain depth — most architecture-aware prompt |
| 7 | AutoResearch | Autonomy — lowest active human corrections |
| 8 | Large Dataset | Novel approach — RAG-inspired design |
| 9 | Standard Prompt | Accessibility — no chess knowledge required |

### Tradeoff Matrix

| If you're optimizing for... | Use this workflow |
|---|---|
| Strongest output | Adversarial |
| Minimum cost | Single Shot |
| Minimum human time | Single Shot |
| Fully hands-off / async | AutoResearch |
| Clean, maintainable code | Phased Engine |
| No domain knowledge required | Single Shot or Standard Prompting |

---

## What We Learned

**More effort did not produce better results.** AutoResearch spent $2.88 and 150 minutes; it finished 7th. Random Features spent 9 hours; it finished 4th. Adversarial spent $0.58 and 40 minutes; it finished 1st. The data strongly suggests that workflow *structure* — specifically, adversarial feedback loops — matters more than raw compute or human time invested.

**The quality floor is higher than expected.** Single Shot finished 3rd with a single unguided prompt. Any engineer evaluating whether to invest in a more elaborate AI workflow should treat Single Shot as the baseline, not an afterthought.

**Reliability is the hidden variable.** Several engines that might have performed well on pure chess logic (Stockfish Replica, Standard Prompt, AutoResearch) were disqualified from competitiveness by timeouts and crashes. A chess engine that times out loses. Reliability is not a secondary metric — it is a prerequisite. Adversarial's zero-timeout, zero-crash record was as important to its 40/40 score as its move quality.

**Human knowledge transfer has diminishing returns.** Stockfish Replica required the highest prior knowledge score (5/5) and the highest frustration score (4/5), hit the usage limit at 90 minutes, and still finished 6th — crashing 26 out of 40 games. The idea that human expertise fed to the model improves output was not supported by the data, at least not within the constraints of a hackathon.

---

## The Team

| Member | Engine(s) | Role |
|---|---|---|
| Ethan | Adversarial, Random Features, Single Shot | Engine builder |
| Sachin | Standard Prompting | Engine builder |
| JD | Phased Engine, AutoResearch | Engine builder |
| Hyun-Dai | Stockfish Replica | Engine builder |
| Hyun | Build Up (TicTacToe), Large Dataset | Engine builder |
| Ishan | Evaluation Framework | Harness design & tournament execution |
