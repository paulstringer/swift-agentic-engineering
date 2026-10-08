# Launch failure: coder never starts, and the minimal-ZDOTDIR fix

Date: 2026-10-08 · Repo: `swift-agentic-quality-engineering` · SwarmForge: `unclebob/swarm-forge` `main` `f4f5fbc` (unchanged since 2026-09-04), scripts unmodified

**Status: reproduced without a swarm, fixed without changing SwarmForge. Verified on 2 of 2 real launches. n is small.**

## Problem
The coder's `claude` process did not start in exp-01 (2026-10-05), exp-02 (2026-10-06) or a test launch on 2026-10-08 (3 of 3). Each time the board looked busy while no agent ran. The cleaner started every time. Earlier notes (exp-01, exp-02) attributed this to the "You have new handoff mail" text being typed into the pane. That was wrong: in the 2026-10-08 launch no brief had been typed and no handoff existed, and the coder pane still showed its own launch line cut off at `…/swarmforge/scripts/swar`.

## Cause
1. SwarmForge's `launch-command` builds one shell line per role from absolute paths (about 1,070 characters for the coder before its tail on a 146-character root; the real coder line is about 1,500 with our 99-character path). The first role also gets a cleanup tail.
2. `launch-role!` types that line into the pane with `tmux send-keys … Enter` right after the session is created, then prints "started in session …" without checking anything.
3. The pane's zsh takes about 0.6 s to start (the operator's dotfiles). Until its line editor starts, typed-ahead input sits in the terminal's canonical line buffer, limited to 1,024 characters on macOS (`PC_MAX_CANON`). A longer line is lost or cut, and `claude` never runs.

## Evidence

Typed-ahead into a shell that was not yet reading (scratch tmux, `sleep 3; head -n 1`):

| Characters typed | Received |
|---|---|
| 900 | 900 |
| 1,020 | 1,020 |
| 1,100 | 0 |
| 1,300 | 0 |

Real zsh in a scratch tmux pane, 1,300-character line typed 0.1 s or 0.3 s after the session starts:

| Shell | 1,300-char line | 900-char line (control) |
|---|---|---|
| Operator's zsh (about 0.6 s start-up) | failed 6 of 6 | succeeded 3 of 3 |
| Minimal `ZDOTDIR` (empty `.zshrc`, about 0.03 s) | succeeded 6 of 6 | n/a |

Launch-line length from the launcher's own `--test-launch-command` (coder line without the tail):

| Root | Length |
|---|---|
| 146-character absolute path | 1,070 |
| `.` (relative root) | 771 (mixed: some paths stay absolute) |
| `/tmp/saqe` (symlink) | 522 |

## Fix
Launch with a minimal shell dir so that zsh is ready before the line arrives:

```
mkdir -p ~/.zdotdir-swarm && touch ~/.zdotdir-swarm/.zshrc     # one time
ZDOTDIR=$HOME/.zdotdir-swarm CLAUDE_CONFIG_DIR=$HOME/.claude-swarm ./swarm
```

Tmux panes inherit `ZDOTDIR` from the launching shell, so SwarmForge is untouched. `claude`, `bb`, `git`, `swift` and `tmux` still resolve in a minimal pane because `PATH` is inherited from the launching shell.

## Verification on real launches

| Launch | Branch | ZDOTDIR | Coder started by itself | Cleaner |
|---|---|---|---|---|
| 2026-10-08 10:13 | `exp-02-two-pack` | no | **no** (line cut off at `…/scripts/swar`, pane at `zsh`) | yes |
| 2026-10-08 10:49 | `exp-02-two-pack` | yes | yes (10:49:51) | yes |
| 2026-10-08 10:54 | `exp-03-two-pack` (fresh, from `main`) | yes | yes (10:54:11) | yes |

In the two ZDOTDIR launches both roles ran their start-up reads and `ready_for_next.sh` (`NO_TASK`) within about 20 s. No brief was typed, so this verifies launch only: no handoff, no code, no cost beyond start-up.

## Caveats
- This narrows a timing race rather than removing it. A shell that starts slowly again (for example a heavy `.zshrc` in `ZDOTDIR`) would bring the failure back. 2 of 2 real launches and 6 of 6 scratch trials are encouraging, not proof.
- The coder line is still about 1,500 characters. A short root path (symlink, `./swarm /tmp/saqe`) takes it to about 790 and removes the dependence on timing. Estimated, not yet run on a real swarm.
- The agents' shells no longer read the operator's dotfiles. This is a (small) change to the environment under test and applies to every run from now on.
- I did not check whether any alias or shell function from the dotfiles was relied on by the agents.

## Effect on earlier findings
- The coder-launch failure in [exp-01](exp-01-two-pack.md) and [exp-02](exp-02-two-pack.md) was a launcher artefact, not caused by the brief or the pack. Their conclusions about the pack are unchanged, but their cause attributions for the coder failure should be read with this note.
- The `blockReadsOutsideWorkingDirectories` finding is separate and still stands.
