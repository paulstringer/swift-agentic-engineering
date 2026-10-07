# Metrics spec: deterministic quality metrics

**Status:** draft, 2026-10-07. Nothing here is implemented. The tools are built by SwarmForge against briefs, never by hand.

## Purpose

Give every experiment run the same set of numbers, so quality conditions (two-, four-, six-pack) can be compared on evidence. These are the numbers the quality tools emit, plus the per-run measurements around them (see [benchmarks-first-runs](../findings/benchmarks-first-runs.md)).

## Rules for a metric

1. **Deterministic.** Same source tree and same test suite give the same value on every run, on any machine with the same toolchain. No LLM judgement, no unseeded sampling, no wall-clock dependence.
2. **Machine-readable.** Every tool can print JSON (shape below) as well as human text, and exits non-zero only on a gate failure or a tool error, never on a plain measurement.
3. **Honest about absence.** A metric that was not computed is `null`, never `0`.
4. **Versioned.** Every report carries `schema` and `tool` versions.
5. **Not novel.** These are established software-engineering metrics. We claim no novelty in them (Project.md §475).

## Common report envelope

```json
{
  "schema": "1",
  "tool": { "name": "dependency-checker-swift", "version": "0.1.0" },
  "target": { "path": "Sources", "commit": "06e815f", "swift": "6.x" },
  "metrics": { },
  "findings": [ ],
  "gate": { "passed": true, "thresholds": { } }
}
```

- `metrics`: the named values below.
- `findings`: the detail behind a metric, one entry per item (cycle, clone group, survivor, over-threshold function), each with `file`, `line` where relevant, and a stable `id`.
- `gate`: only present when thresholds were supplied. Thresholds are inputs, so they appear in the output.
- Ordering: all lists sorted (by file, then line, then id) so output is byte-identical between runs.

## Metrics by tool

### dependency-checker-swift (first, in progress)

| Metric | Unit | Definition |
|---|---|---|
| `components` | count | components found (top-level folder with a Swift source file) |
| `edges` | count | distinct component → module-component edges |
| `cycles` | count | strongly connected components with more than one member |
| `cycle_members` | list | members of each cycle, sorted |
| `ca` | per component | afferent coupling: components that depend on this one |
| `ce` | per component | efferent coupling: components this one depends on |
| `instability` | per component, 0–1 | `ce / (ca + ce)`; `null` if both are 0 |
| `max_depth` | count | longest path after collapsing cycles |
| `rule_violations` | count | edges breaking an allowed/forbidden rule (needs a rules file; later) |

Only import-level dependencies count until type-reference analysis exists. Imports of modules that are not components (Foundation, SwiftUI) are excluded from the graph and counted as `external_imports`.

### dry4swift (duplication)

| Metric | Unit | Definition |
|---|---|---|
| `clone_groups` | count | groups of structurally identical code above `min_tokens` |
| `duplicated_lines` | count | lines in any clone instance, counted once |
| `duplication_ratio` | 0–1 | `duplicated_lines / total_lines` |
| `largest_clone` | lines | size of the biggest group, with its instance count |

Structural match on SwiftSyntax token streams with identifiers and literals normalised. `min_tokens` is a parameter and part of the output.

### crap4swift (complexity and coverage)

| Metric | Unit | Definition |
|---|---|---|
| `complexity` | per function | cyclomatic: 1 + decision points counted from the syntax tree |
| `coverage` | per function, 0–1 | line coverage from `swift test --enable-code-coverage`; `null` if not run |
| `crap` | per function | `complexity² × (1 − coverage)³ + complexity`; `null` without coverage |
| `max_crap` | number | worst function |
| `over_threshold` | count | functions above the threshold (default 10, per the cleaner role prompt) |
| `function_length`, `nesting_depth` | per function | cheap syntax-tree add-ons |

### mutate4swift (mutation)

| Metric | Unit | Definition |
|---|---|---|
| `mutants` | count | mutants generated |
| `killed`, `survived`, `timed_out`, `excluded` | count | outcome of each mutant |
| `mutation_score` | 0–1 | `killed / (mutants − excluded − timed_out)`, with `timed_out` reported separately |
| `survivors` | list | file, line, operator, for each survivor |

Determinism conditions: fixed mutation operator order, no random sampling (or a fixed seed recorded in the report), a fixed per-mutant timeout. Timeouts are the main source of drift, which is why they are excluded from the score and listed on their own. Existing Swift mutation tools are evaluated first (Project.md §177).

### Code and tests (no new tool)

`source_lines`, `test_lines`, `test_count`, `assertion_count`, `build_warnings`, `files`. From `git ls-files`, `wc` and `swift test`.

## Per-run metrics

Computed by running the tools at each commit the swarm makes, so the output shows what each role changed.

| Metric | Definition |
|---|---|
| `delta_per_handoff` | each tool metric at the coder commit vs the following cleaner commit |
| `gate_history` | per commit, which gates passed |
| `iterations_to_green` | handoffs between a gate failing and next passing |
| `regressions` | any metric that got worse between consecutive commits |
| `gate_coverage` | which of the four disciplines ran at all in the run (exp-02: 0 of 4) |
| final-state scores | every metric above on the result commit |

Alongside these, the session benchmarks (cost, tokens, model, agent-active time) from `experiments/setup.md`. Those are measurements, not deterministic metrics, and are reported with n.

## Not metrics
- LLM review scores.
- Agent time, tokens or cost as a quality claim.
- Anything from a test suite with timing-dependent tests.

## Build order
1. Dependency: cycles, Ca/Ce/instability, depth. Almost free from the existing graph. Brief: `experiments/brief-depgraph-metrics.md` in `swift-agentic-quality-engineering`.
2. CRAP.
3. Duplication.
4. Mutation.

## Open questions
- Which Swift coverage source: `llvm-cov` JSON export or the SwiftPM coverage output?
- Per-function CRAP needs a stable function identity across commits (file + qualified name?) for the delta metrics.
- Whether to adopt an existing mutation tool or build `mutate4swift`.
- JSON schema as a checked-in file, so agents and tests can validate against it.
