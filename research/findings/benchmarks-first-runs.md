# Benchmarks — first two-pack runs (exp-01, exp-02)

Captured 2026-10-07 from run artefacts, after both runs. This fills the "not measured" cells in [exp-01](exp-01-two-pack.md) and [exp-02](exp-02-two-pack.md). Same brief, same pack (two-pack: `coder`, `cleaner`), same model (`claude-sonnet-5-5`) in both runs.

## Sources
- Per-session cost, tokens and durations: the `cost-state` record at the end of each Claude Code transcript (`~/.claude-swarm/projects/…` for exp-02, `~/.claude/projects/…` for exp-01 and the first exp-02 coder restart). `costUSD` is Claude Code's own estimate, not an invoice.
- Agent activity windows: first and last assistant-message timestamps in the transcript (UTC; BST = UTC+1).
- Code size and tests: `git ls-files` and `swift test` on `exp-02-two-pack` at `06e815f`.
- Not captured: SwarmForge's own per-handoff timings. It does not record them.

## Headline numbers

| | exp-01 | exp-02 |
|---|---|---|
| Result | no code, aborted | all 6 criteria met, 9 tests pass |
| Model | claude-sonnet-5-5 | claude-sonnet-5-5 |
| Total API cost, all agent sessions | **$0.34** | **$1.03** |
| Cost of the sessions that produced the result | n/a | **$0.67** (coder $0.36 + cleaner $0.31) |
| Cost of sessions that produced nothing | $0.34 (all of it) | $0.36 (restart without config dir $0.21, launch-time cleaner $0.15) |
| Agent-active time that produced the result | 0 | **≈ 4 min** (coder 2 m 19 s, cleaner 1 m 51 s) |
| Brief typed → result committed | n/a | ≈ 4 h (3.5 h of it waiting on a coder that never launched) |

"Agent-active" is first to last assistant message in the session, which includes tool time. It is not the same as the "≈ 16 min productive" in the exp-02 note, which ran from coder launch (13:36) to done (13:52) and so includes the cleaner's start-up wait. Both are true; they measure different things.

## Per-session detail

Tokens are from `cost-state`. "Cache read" is context re-read from the prompt cache on each turn, which is why it dwarfs everything else.

### exp-02 (2026-10-06)

| Session | Role | Active (BST) | Output tok | Thinking tok | Cache read | Cache write | API time | Cost |
|---|---|---|---|---|---|---|---|---|
| c5330dd6 | cleaner, first launch | 10:55:42 – 10:55:49 | 429 | 0 | 205,866 | 24,844 | 15 s | $0.146 |
| 54d6e41f | coder, 1st manual restart, **no config dir** | from 13:33 | 3,633 | 229 | 314,090 | 27,854 | n/a | $0.211 |
| 5cb8a35d | coder, 2nd manual restart, config dir set | 13:37:22 – 13:39:41 | 6,747 | 873 | 727,813 | 36,413 | 54 s | $0.359 |
| 04bcce99 | cleaner, restarted by hand | 13:50:21 – 13:52:12 | 3,305 | 636 | 679,537 | 36,431 | 39 s | $0.315 |
| | **Total** | | 14,114 | 1,738 | 1,927,306 | 125,542 | | **$1.030** |

- Coder: 13 messages, 11 Bash calls and 2 Reads. It wrote the whole tool with shell heredocs (no Write/Edit calls). Commits at 13:39:03 and 13:39:32.
- Cleaner: 12 messages, 10 Bash calls and 2 Reads. One commit at 13:51:53, handoff queued 13:52:07.
- The first cleaner launch (c5330dd6) lasted 5 s of agent activity. Its transcript ends after the start-up reads. This is probably the cleaner whose window later disappeared (exp-02 finding 3), but the transcripts do not confirm that.
- `swift test` time is negligible: 9 tests in 0.014 s. A cold `swift test` including build was 5.7 s on the operator's machine with a warm `.build`.

### exp-01 (2026-10-05)

| Session | Role | Started (BST) | Output tok | Thinking tok | Cache read | Cache write | Cost |
|---|---|---|---|---|---|---|---|
| 3a2566d3 | cleaner | 16:07 | 418 | 0 | 210,767 | 26,186 | $0.152 |
| bca71ebb | coder, manual restart | 16:12 | 3,144 | 396 | 253,200 | 26,229 | $0.187 |
| | **Total** | | 3,562 | 396 | 463,967 | 52,415 | **$0.339** |

The coder produced no commits. Its transcript shows 3,144 output tokens and no code. Not included: five operator probe sessions testing `cd "$(pwd)/experiments" && ls` against the permission setting, about $0.27 in total. They are setup diagnosis, not run cost.

## Code produced (exp-02, at `06e815f`)

| Measure | Value |
|---|---|
| Swift source (`Sources/`) | 107 lines in 5 files (`ImportScanner` 10, `DependencyGraph` 46, `SourceTree` 18, `DepGraphCommand` 18, `main` 15) |
| Tests | 83 lines, 9 tests, 0 failures |
| Fixtures | 2 projects, 6 files, 13 lines |
| Total vs `main` | 12 files, +203 lines |
| Commits | coder 2, cleaner 1 |
| Coverage, CRAP, duplication, mutation | not run (see below) |

## Quality-gate behaviour (answers an open exp-02 follow-up)

The cleaner's role prompt says to run coverage, a CRAP tool (≤ 10) and a mutation tool. What it did, from the transcript:

1. `swarm_tool.sh require crap4clj` returned `MISSING: crap4clj — Run: swarm_tool.sh ensure crap4clj`.
2. It ran `swift test` (9 pass) and read the tests.
3. Its stated reason for stopping there: "The Clojure tools (CRAP, DRY, mutation) don't apply to Swift. I'll do the structural cleanup and keep behavior the same." It did not run `ensure`, and ran no coverage tool.
4. It split `DepGraph.swift` into four files. Its first commit-and-handoff command failed on a BSD `sed -i` usage error but the later steps ran. 9 tests passed after the split, which was the only check on the refactor.
5. `swarm_handoff.sh` refused the first handoff with `AUDIT_REQUIRED / HANDOFF_NOT_QUEUED`. The cleaner re-read the diff against the brief, found nothing to fix, and retried. This is the only SwarmForge gate that fired in either run. It cost one extra round of about 20 s.
6. The retry queued **two** handoff files (`…000002…` and `…000003…`) for the same task. Cause unknown.

So the pack's built-in quality loop did nothing for Swift in this run: the only gate that bit was a generic audit step, not a code-quality measure.

## Baseline for the next runs

For a four- or six-pack run on the same brief to be comparable, these are the numbers to beat or explain:

| Measure | exp-02 two-pack |
|---|---|
| Cost of result-producing sessions | $0.67 |
| Cost including operator-caused restarts | $1.03 |
| Agent-active time | ≈ 4 min |
| Output tokens (result-producing sessions) | 10,052 (+ 1,509 thinking, see note) |
| Tests / source lines | 9 / 107 |
| Quality gates that ran on Swift code | 0 (audit only) |

Note on the output-token row: coder 6,747 + cleaner 3,305 = 10,052. `cost-state` reports thinking tokens separately (873 + 636 = 1,509); it is not confirmed whether `outputTokens` already includes them, so treat the total as 10.1k–11.6k.

## Caveats
- n = 1 per condition and exp-01 produced nothing, so there is no variance estimate. One more two-pack run would be needed before reading anything into $0.67 vs a four-pack figure.
- The two runs are not equally clean. Costs in the "Total" rows include operator-caused restarts, and the result-producing rows exclude them. Use the right row for the comparison being made.
- `costUSD` depends on Claude Code's built-in price table at the time of the run.
- Exp-02's prompt (`Implement @brief-import-graph.md`) differed from exp-01's (brief text verbatim), as noted in the exp-02 finding. Only the brief file is held fixed.
