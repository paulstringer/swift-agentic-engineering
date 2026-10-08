# Experiment 03 — two-pack — findings

Date: 2026-10-08 · Branch: exp-03-two-pack · Backend/model: claude (both roles, `claude-sonnet-5-5`), `--dangerously-skip-permissions`, `CLAUDE_CONFIG_DIR=~/.claude-swarm`, `ZDOTDIR=~/.zdotdir-swarm` · Baseline commit: `72c1160` (`main`) + pack commit `f7f6aa7` · Result commit: `138877c` (tag `baseline/depgraph-exp03`)

**Status: COMPLETED, clean run. All six acceptance criteria met. No operator actions during the run.**

## Setup
- `main` got the launch fix first (see [launch-race-fix](launch-race-fix.md)): launch with a minimal `ZDOTDIR` so the coder's long launch line is not lost.
- Branch `exp-03-two-pack` cut from `main`, `get-swarm-forge two-pack` (SwarmForge `main` `f4f5fbc`, unchanged since 2026-09-04), both roles set to `claude` + `--dangerously-skip-permissions`. Committed as `f7f6aa7`.
- Swarm launched at 10:54 with `ZDOTDIR=$HOME/.zdotdir-swarm CLAUDE_CONFIG_DIR=$HOME/.claude-swarm ./swarm`. Both agents started by themselves and ran their start-up reads and `ready_for_next.sh` (`NO_TASK`) within about 20 s.
- Prompt typed into the swarm at 11:02: `See @brief-import-graph.md` (saved by SwarmForge as `tasks/import-graph.md`). exp-02's prompt was `Implement @brief-import-graph.md`. The brief itself (`experiments/brief-import-graph.md`) is unchanged.

## Outcome
Acceptance criteria, checked by the operator on the result commit:
1. Sorted `X → Y` edges: pass. `Fixtures/Layered` prints `Domain → Data`, `Domain → UI`, `UI → Domain`.
2. SwiftSyntax, not regex: pass. `ImportScanner` uses `SwiftParser` and `ImportDeclSyntax`; a grep for `NSRegularExpression`, `Regex` and `range(of` found nothing.
3. Fixture with `UI`, `Domain`, `Data` and a planted violation: pass. `Fixtures/Layered/Domain/Violation.swift` imports `UI`.
4. Expected edges on the fixture, as an automated test: pass (`CLITests` runs on `Fixtures/Layered`; assertions not read line by line).
5. Cycle detected, printed, non-zero exit: pass. `Layered` prints `cycle: Domain → UI → Domain` and exits 1.
6. `swift test`: pass. 14 tests, 0 failures, 0.014 s.

There is no acyclic fixture in this result (exp-02 had one). The brief does not require it.

## Measurements
| Measure | Value |
|---|---|
| Agent iterations / handoffs | task → coder → cleaner → coder (completion). Commits: coder 2, cleaner 1. 3 handoff files in `inbox/completed` (one task handoff, two cleaner → coder). None failed. |
| Human interventions (SwarmForge approval gates, clarification questions) | 0 |
| Operator actions (not interventions) | 0 during the run (typed the prompt only) |
| Wall-clock time | 11:02:43 prompt → 11:05:41 board `done`: 2 min 58 s. Coder 11:02:44 – 11:04:24 (build), cleaner 11:04:31 – 11:05:45. |
| Token / API cost | **$0.955** total: coder $0.610, cleaner $0.345 (Claude Code's own estimate from `cost-state`) |
| Tokens (in / out / thinking / cache read / cache write) | coder 7,649 / 9,298 / 966 / 1,677,639 / 41,641; cleaner 1,042 / 2,432 / 464 / 920,735 / 33,479 |
| API time / tool time | coder 84 s / 59 s; cleaner 35 s / 62 s |
| Test count / coverage | 14 tests; coverage not measured |
| Code | 153 source lines in 7 files, 125 test lines in 4 files, 19 fixture lines, `Package.swift` 21 |

## Compared with exp-02 (same brief, same pack)
| | exp-02 | exp-03 |
|---|---|---|
| Launch | coder hand-started twice, cleaner once, one restart without the config dir | coder and cleaner started by themselves (`ZDOTDIR`) |
| Brief → done | about 4 h (3.5 h waiting on a dead coder) | 2 min 58 s |
| Cost, sessions that produced the result | $0.67 | $0.95 |
| Cost, all sessions | $1.03 | $0.95 |
| Tests / source lines | 9 / 107 | 14 / 153 |
| Cleaner outcome | split one file into four, behaviour unchanged | restored dropped `.gitignore` entries, no refactor |

n = 1 per condition. The cost difference is within what run-to-run variation could produce, and exp-03 produced more code and tests. Not evidence of a trend.

## What went wrong or surprised us
1. **The launch race did not appear.** First launch of the coder without a manual restart in any run. See [launch-race-fix](launch-race-fix.md).
2. **Case-insensitive filesystem collision.** The coder first made `Sources/DepGraph` (library) and `Sources/depgraph` (executable), which collide on macOS. It renamed the library to `DepGraphCore`, then fixed the executable directory's case with `git mv` in a follow-up commit (`6260ce7`). Swift-specific friction that a Swift-aware gate (or a convention) could have prevented.
3. **The coder's commit changed `.gitignore` and dropped lines.** `.swarmforge/` and `.worktrees/` were removed (the file was rewritten to `.build/`, `.swiftpm/`, `tmp/`, `.DS_Store`). Left alone, the next `git add` on the branch would have picked up SwarmForge run state.
4. **The audit gate caught it.** `swarm_handoff.sh` refused the cleaner's first handoff with `AUDIT_REQUIRED`. The cleaner re-read the work against the brief, found the `.gitignore` regression and fixed it in `138877c`. This is the first time in three runs that a gate led to a change in the product tree. It is a generic SwarmForge audit, not a Swift quality measure.
5. **No Swift quality tooling ran.** The cleaner ran `swarm_tool.sh require dry4clj` and got `MISSING: dry4clj`, and did not install it or run any coverage, CRAP, duplication or mutation check. Same outcome as exp-02 (`crap4clj`). The pack's tooling is Clojure-shaped.
6. **The cleaner chained the commands in the wrong order.** It ran `swarm_handoff.sh … && done_with_current.sh`, then a second `done_with_current.sh` returned `NO_CURRENT_BATCH`. Harmless.
7. **Duplicate handoff files again.** The cleaner → coder handoff produced two files (`…000002…`, `…000003…`) for one task, as in exp-02 (2 of 2). Cause unknown.
8. **Cost record timing.** `cost-state` is written only when a session exits. A cost estimate computed from per-message usage during the run came out at $0.85 against the recorded $0.955, because per-message `usage` undercounts input and cache tokens. Use `cost-state`.

## Does this answer the question?
Weakly, and in the same way as exp-02. The only quality feedback that influenced the code was the generic audit step, which caught a repository hygiene defect (3). No executable Swift quality feedback existed: no coverage, CRAP, duplication, mutation or architecture check ran. What this run does establish is a **clean two-pack baseline**: a correct tool for this brief in under 3 minutes of wall-clock time and about $0.95, with zero operator actions, now that the launch fault is avoided. A four- or six-pack run with Swift-capable gates would have something to be compared with.

## Follow-ups
- Run the metrics brief (`brief-depgraph-metrics.md`) from tag `baseline/depgraph-exp03` as the "feature on existing code" experiment.
- Decide the Swift gate set (architecture rules, duplication, complexity and coverage, mutation) before the four-pack.
- Find out why the cleaner → coder handoff is queued twice.
- Check whether the `.gitignore` regression is typical for the coder, or particular to this run.
