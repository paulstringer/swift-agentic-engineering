# Experiment 01 — two-pack — findings

Date: 2026-10-05 · Branch: exp-01-two-pack · Backend/model: claude (both roles), `--dangerously-skip-permissions` · Baseline commit: cd861bf

**Status: FAILED RUN, aborted by the operator. No product code was produced.**

## Setup
- Branch `exp-01-two-pack` from `main`, two-pack installed (`coder`, `cleaner`), both roles on `claude` + `--dangerously-skip-permissions`, per `experiments/setup.md`.
- Swarm launched with `./swarm` at ~15:29.
- Brief typed into the swarm as a prompt: the text of `brief-import-graph.md` verbatim, plus one opening line naming the tool `depgraph`. SwarmForge saved it as `tasks/Import Graph.md` and queued a handoff to `coder` at 15:30.

## Outcome
- Brief acceptance criteria 1–6: **none met** (no `Package.swift`, no sources, no tests, no commits).
- `swift test` passes: n/a.

## Measurements
| Measure | Value |
|---|---|
| Agent iterations / handoffs | 0 completed (1 queued, never consumed until the manual restart) |
| Human interventions (approvals, clarifications) | 1 approval, given by the operator on a permission prompt after the restart (see below). Not a SwarmForge gate. |
| Operator actions (not interventions) | Diagnosis ~16:10; coder pane cleared and `claude` relaunched by hand; swarm torn down from the UI |
| Wall-clock time | ~40 min with the coder not running (15:30–~16:10), then a short restarted attempt, then abort |
| Token / API cost | $0.34 across both agent sessions, no output (see [benchmarks](benchmarks-first-runs.md)) |
| Test count / coverage | n/a |

## What went wrong or surprised us
1. **The coder never started.** The coder pane's launch command was garbled when the "You have new handoff mail" notification was typed into the pane mid-command. (Superseded: the likely cause is the 1,024-character typed-ahead limit hit by the coder's long launch line while zsh was still starting. See [launch-race-fix](launch-race-fix.md).) The shell was left at `quote>` and `claude` never launched. The queued handoff sat unconsumed.
2. **The forge looked busy but wasn't.** The board showed a card in the coder lane, windows were open and the daemons were up. Nothing signalled that no agent was running. Only `ps`, `tmux capture-pane` and the handoff folders revealed it. See the Obsidian note "SwarmForge looks busy but isn't - hard to tell what it is doing".
3. **Permission prompts still appeared despite `--dangerously-skip-permissions`.** After the manual restart the coder hit prompts from the `permissions.blockReadsOutsideWorkingDirectories` setting on commands the shell parser could not analyse (a `cd` to a computed path, a brace with a quote character). This breaks the premise in `setup.md` that the flag makes runs prompt-free and comparable.
4. The operator restart was a manual intervention in the run, so the run is not a clean sample of the two-pack condition.

## Does this answer the question?
No. The quality feedback loop was never exercised: the cleaner received no work and no code was written. The only evidence is about the harness: launch fragility, no liveness signal, and permission prompts that survive the skip flag.

## Follow-ups before the next run
- Add a liveness check to the run procedure: a `claude` process per role, and tool activity or a first commit from the coder within a few minutes of the brief.
- Find out why the managed `blockReadsOutsideWorkingDirectories` setting overrides the flag, and whether to disable it for experiment runs.
- Re-run as exp-02-two-pack with the same unchanged brief and prompt.
