# Benchmarks

Status: **empty by design.** Nothing is added until a corpus is chosen and the existing Swift tools for that discipline have been evaluated.

## What belongs here

A reference set for the quality tools, so a tool we build can be compared with accepted ones on code we did not write.

| Thing | What it is | Where it lives |
|---|---|---|
| Fixture | Tiny hand-planted project (a known cycle, a known clone). The expected answer is known because we planted it. | With the tool code, e.g. `Fixtures/` in `swift-agentic-quality-engineering` |
| Corpus | Larger, realistic Swift code, pinned by URL and commit | Described in `benchmarks/<discipline>/corpus.md`, not copied in |
| Reference results | Output of an established tool on the corpus | `benchmarks/<discipline>/reference/` |
| Expected results | Output of our tool on the same corpus, once it exists | `benchmarks/<discipline>/expected/` |

Experiment run results (cost, time, tokens, per-run metrics) do **not** go here. They go in `research/findings/`. Metric definitions are in `research/experiments/metrics-spec.md`.

## Layout

```
benchmarks/
  dependency/
  duplication/
  complexity/
  mutation/
  agent-workflows/
```

Each discipline folder has `corpus.md`, `reference/` and `expected/`. `agent-workflows/` is for fixed tasks and expected outcomes used to compare agent set-ups, and is defined later.

## Rules

1. **Pin everything.** Every reference file records the tool and its version, the Swift toolchain, the corpus URL and commit, and the exact command. Without all four, the file is not a reference.
2. **Do not vendor other people's code.** Pin by URL and commit, and check the licence in `corpus.md`.
3. **A reference is an oracle, not the truth.** Where our tool and the reference disagree, record which one is right and why, in `expected/`.
4. **Deterministic only.** Re-running the same command on the same pins gives the same output, or the file says what varies.

## Candidate reference tools

From memory, not yet checked. Verify maintenance and Swift version support before using any of them.

- Duplication: `jscpd`
- Complexity: SwiftLint `cyclomatic_complexity`
- Coverage: `llvm-cov` via `swift test --enable-code-coverage`
- Mutation: Muter
- Dependency: no close match known. A hand-checked graph on a small corpus may be the oracle.

## Before adding anything

1. Evaluate the existing tools for the discipline (Project.md, "Existing Swift ecosystem tools should be evaluated before duplicating functionality").
2. Choose and record the corpus in `corpus.md`.
3. Run the reference tools and commit the pinned output.
