# Experiment: dependency graph (import-graph checker)

**Status:** two-pack runs completed (exp-02, exp-03; exp-03 is the clean baseline). Four- and six-pack runs not started.

## Purpose

This is the first workload for the central research question in [`docs/thesis.md`](../../docs/thesis.md): can established quality disciplines become executable feedback for coding agents? The tool is only a workload. The evidence is what the swarm did under each quality condition.

A dependency checker is the smallest useful test of the "architecture" feedback in the loop: a tool that reads Swift source and reports component dependencies is both a product the agents must build and the first verification primitive such a loop would use.

## Design

- **Workload (fixed):** the brief below, run unchanged under every condition. It is never edited to suit a run.
- **Condition:** the SwarmForge pack (two-, four-, six-pack) and its quality gates.
- **Environment:** SwarmForge ([unclebob/swarm-forge](https://github.com/unclebob/swarm-forge)), every role on the `claude` backend with `--dangerously-skip-permissions`, launched with a dedicated Claude config dir so the operator's global settings do not change agent behaviour.
- **Output:** a findings note per run. A working tool is a bonus, and a recorded failure is a valid result.
- **Operator rules:** the operator never writes product code. Interventions are counted separately: "human interventions" means SwarmForge approval gates and clarification questions only. Operator actions (restarts and the like) are listed on their own.
- **Reproducing a run:** branch from the baseline in `swift-agentic-quality-engineering` (`main`), follow `experiments/setup.md`, install the pack, and type a prompt that points at the brief. Run branches (`exp-NN-<pack>`) are local and never pushed, so the code a run produced is not kept. The findings notes are the record.

## Brief

> Build a small command-line tool in Swift that reads a directory of Swift source and prints the dependencies between its components.
>
> - **Component:** the top-level folder (relative to the given path) that contains a source file.
> - **Dependency:** a component imports a module, e.g. `import Domain` in a file under `UI/`.
>
> Acceptance criteria:
> 1. `depgraph <path>` prints one line per distinct edge, sorted, in the form `UI → Domain`.
> 2. Only `import` declarations are considered. Parsing uses SwiftSyntax, not regular expressions.
> 3. A fixture project exists with three components (`UI`, `Domain`, `Data`) layered `UI → Domain → Data`, plus one planted violation where `Domain` imports `UI`.
> 4. Running the tool on the fixture prints exactly the expected edges, including the violation. This is an automated test.
> 5. `depgraph` exits non-zero when it finds a cycle between components, and prints the cycle. The fixture's planted violation must trigger this.
> 6. All tests pass with `swift test`.
>
> Out of scope: type-reference analysis, configuration files, allowed/forbidden rules, CI integration.

The canonical copy is `experiments/brief-import-graph.md` in `swift-agentic-quality-engineering`.

## Runs

| Run | Pack | Date | Outcome | Findings |
|---|---|---|---|---|
| exp-01 | two-pack | 2026-10-05 | Failed: the coder agent never started, no code produced | [exp-01-two-pack](../findings/exp-01-two-pack.md) |
| exp-02 | two-pack | 2026-10-06 | Completed: all six criteria met, 9 tests, with three manual agent starts | [exp-02-two-pack](../findings/exp-02-two-pack.md) |
| exp-03 | two-pack | 2026-10-08 | Completed: all six criteria met, 14 tests, no operator actions, brief to done in 2 min 58 s, $0.95 | [exp-03-two-pack](../findings/exp-03-two-pack.md) |

## What we have learned so far

- **The workload is tractable.** In exp-02 the coder and cleaner produced a correct tool in about 16 productive minutes: SwiftSyntax parsing, a layered fixture with a planted violation, cycle detection with a non-zero exit, and 9 passing tests.
- **The two-pack has no Swift quality loop.** The cleaner did one structural refactor and ran no mutation, coverage, CRAP, duplication or architecture checks. The pack's tooling is Clojure-shaped (`crap4clj`, `clj-mutate`, `speclj`). So this run is a baseline for what a bare two-role swarm produces, not evidence about quality feedback.
- **The harness is fragile in ways that distort measurement.**
  - The coder launch failed in exp-01, exp-02 and a test launch (the coder's launch line exceeds the terminal's 1,024-character typed-ahead limit while zsh is still starting), losing about 3.5 hours in exp-02. Fixed without changing SwarmForge by launching with a minimal `ZDOTDIR`; exp-03 launched cleanly. See [launch-race-fix](../findings/launch-race-fix.md).
  - The board shows a card as queued or moved even when no agent is running.
  - A user-level permission setting blocks agent commands even with the skip flag, and no project-level setting overrides it. A dedicated config dir is needed.
  - Measures such as wall-clock time and interventions need a liveness check before they mean anything.
- **Observability of agent state is itself a missing control.** Whether an agent is alive, what it is doing and what it is waiting for are not visible in the forge UI. That fits the wider theme of verification and controls for organisations adopting coding agents.

## Next

1. Choose the Swift quality gates to test, one discipline at a time: architecture rules (the checker this experiment builds is the first candidate), duplication, complexity and coverage, mutation.
2. Run the same brief under the four-pack and six-pack, with Swift-capable gates wired in.
3. Compare across conditions on the same measures: criteria met, handoffs, interventions, productive time, and whether gate feedback changed what the agents did.
4. ~~Resolve the launcher fault~~ Done: minimal `ZDOTDIR` launch (see above). Re-check on every run with the liveness check.
5. Run the feature-on-existing-code brief (`brief-depgraph-metrics.md`) from tag `baseline/depgraph-exp03`.
