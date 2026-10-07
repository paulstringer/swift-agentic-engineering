# Experiment 02 — two-pack — findings

Date: 2026-10-06 · Branch: exp-02-two-pack · Backend/model: claude (both roles), `--dangerously-skip-permissions`, dedicated config dir `CLAUDE_CONFIG_DIR=~/.claude-swarm` · Baseline commit: a8d5588 (`main`) + 55522a9 (teardown) · Result commit: 06e815f

**Status: COMPLETED with operator help. All six acceptance criteria met. The run is not clean: three agent starts were done by hand.**

## Setup
- `main` got a standing setup doc update: dedicated Claude config dir, liveness check and `experiments/teardown.sh` (after exp-01 showed that the operator's user-level `blockReadsOutsideWorkingDirectories: true` blocks agent commands and is not overridden by `--dangerously-skip-permissions` or by any project-level setting).
- Branch `exp-02-two-pack` cut from `main`, `get-swarm-forge two-pack`, both roles set to `claude` + `--dangerously-skip-permissions`.
- Swarm launched with `CLAUDE_CONFIG_DIR=$HOME/.claude-swarm ./swarm`.
- Prompt typed into the swarm: `Implement @brief-import-graph.md`. This differs from the plan, which was the brief text verbatim plus a line naming the tool. The file is at `experiments/brief-import-graph.md`, not the repo root, and the agents located it. The brief itself is unchanged.

## Outcome
Acceptance criteria, checked by the operator in the cleaner's worktree (and again after the merge onto `exp-02-two-pack`):
1. Sorted `X → Y` edges: pass. `Fixtures/Acyclic` prints `Domain → Data` and `UI → Domain`.
2. SwiftSyntax, not regex: pass. `ImportScanner` uses `SwiftParser` and `ImportDeclSyntax`, and a grep found no regex.
3. Fixture with `UI`, `Domain`, `Data` and a planted violation: pass. `Fixtures/Layered/Domain/Model.swift` imports `Data` and `UI`.
4. Expected edges on the fixture, as an automated test: pass.
5. Cycle detected, printed, non-zero exit: pass. `Layered` prints `Cycle: Domain → UI → Domain` and exits 1. `Acyclic` exits 0.
6. `swift test`: pass. 9 tests, 0 failures.

## Measurements
| Measure | Value |
|---|---|
| Agent iterations / handoffs | coder → cleaner: 1 (delivery notification failed, see below). cleaner → coder: 1 (completion, priority 00). Commits: 2 by coder, 1 by cleaner. |
| Human interventions (SwarmForge approval gates, clarification questions) | 0 observed |
| Operator actions (not interventions) | Coder started by hand twice; cleaner started by hand once; one approval of a permission prompt in exp-01 only (none recorded here). |
| Wall-clock time | 09:57 brief typed → 13:52 done, about 4 h. Productive time about 16 min: coder started 13:36, commits 13:39, cleaner commit 13:51, done 13:52. |
| Token / API cost | not measured |
| Test count / coverage | 9 tests, coverage not measured |

## What went wrong or surprised us
1. **Coder launch failed again (2 of 2 runs).** The coder pane's launch command was garbled when the "You have new handoff mail" notification was typed into it mid-command, leaving zsh at `quote>`. `claude` never started and the task sat unconsumed for about 3.5 h. This reproduced in a fresh run with the new config dir, so it is a launcher fault, not caused by our setup.
2. **The first manual coder restart omitted `CLAUDE_CONFIG_DIR`.** The agent hit the `blockReadsOutsideWorkingDirectories` prompt. The mistake was the operator's: the restart command left the variable out. The second restart set it and ran without prompts.
3. **The cleaner window disappeared.** The `swarmforge-cleaner` tmux session was gone by the time the coder finished. Cause unknown. The coder's `git_handoff` failed with `tmux send text failed` and was moved to `handoffs/failed/`, while the board card still moved to the cleaner lane.
4. **The handoff worked anyway.** After the cleaner was restarted by hand, its startup `ready_for_next.sh` picked up the work and it produced a commit. The failed notification did not lose the handoff.
5. **The board does not show liveness.** A card in a lane, open windows and running daemons looked like activity while no agent was running (see the Obsidian note "SwarmForge looks busy but isn't - hard to tell what it is doing"). Only `ps`, `tmux capture-pane` and the handoff folders showed the real state.
6. **The pack's quality tooling is Clojure-shaped.** The cleaner's startup notes list `crap4clj`, `dry4clj`, `clj-mutate`, `cloverage` and `speclj`, and it ran `swarm_tool.sh require crap4clj` before refactoring. I did not capture what that returned, and no Swift-equivalent gates ran.
7. **The cleaner's work is on a separate branch** (`swarmforge-cleaner`), and `teardown.sh` deletes `swarmforge-*` branches. The operator fast-forwarded `exp-02-two-pack` to `06e815f` before teardown so the result is not lost.

## Does this answer the question?
Only weakly. The cleaner made one structural refactor: it split `DepGraph.swift` into `ImportScanner`, `DependencyGraph`, `SourceTree` and `DepGraphCommand`, and the tests still passed. It ran no mutation, coverage, CRAP, duplication or architecture checks on the Swift code, so there was no executable quality feedback to react to. The result shows that a two-pack can produce a correct tool for this brief in about 16 productive minutes, and that its Swift quality loop was absent. It is not evidence for or against quality disciplines as agent feedback. A four- or six-pack run with Swift-capable gates is what the research question needs.

## Follow-ups
- Decide how to handle the coder launch race: always restart the coder by hand and record it, or find the cause in the SwarmForge launcher.
- Capture the cleaner transcript for what `swarm_tool.sh require crap4clj` returned, and whether it chose not to run any gates for that reason.
- Check whether the cleaner changed behaviour in its refactor. Only the existing 9 tests cover it.
- Decide the Swift gate set (architecture rules, duplication, complexity, mutation) before the four-pack.
