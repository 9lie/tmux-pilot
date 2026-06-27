# tmux-pilot

A lightweight tmux wrapper that **auto-saves your layout and Claude Code sessions**, and lets you **pick or restore a session on launch** — all from a single shell script, no TPM, no plugins.

## Why

I run tmux on a remote dev box with a lot of Claude Code sessions open at once.
A few times tmux froze and took the whole layout — every window, pane, and
running Claude conversation — down with it. tmux-pilot snapshots the layout
**and** each pane's Claude `sessionId` on a timer, so a restore brings the actual
chats back, not just empty panes. Kept deliberately small: one Bash file, no TPM,
no plugins.

## Requirements

- `bash` 4+ (the script; your interactive shell can be anything)
- `tmux` ≥ 3.0
- `claude` CLI (optional, for Claude session capture/restore)
- `jq` (optional, faster JSON parsing; falls back to awk)

Single file, no fzf, no python, no plugins.

## Install

```bash
git clone https://github.com/<you>/tmux-pilot
ln -s "$PWD/tmux-pilot/bin/tmux-pilot" ~/.local/bin/tmux-pilot
# ensure ~/.local/bin is on your PATH
```

## Usage

```bash
tmux-pilot            # smart launch: pick / restore / create + start monitor
tmux-pilot save       # snapshot all sessions right now
tmux-pilot restore X  # rebuild session X from its snapshot (detached)
tmux-pilot panes      # show which Claude is in which pane (+ sessionId)
tmux-pilot status     # monitor state + snapshot list
tmux-pilot stop       # stop background monitor
tmux-pilot help
```

### `tmux-pilot panes` — which Claude in which pane

```
TMUX PANE        WINDOW     SESSION    CWD
----------------------------------------------------------------------
demo:1.1         server     f3e738b3   ~/code/app
demo:1.0         server     55df8e97   ~/code/app
```

Maps every running Claude to its tmux pane by reading `claude agents --json`
(pid + sessionId) and walking each pid's parent process chain until it hits a
pane's `pane_pid`. Tells two Claudes apart even in the same directory.

### Launch logic

```
tmux-pilot
├── tmux server frozen? (probe times out)
│   └── prompt to force-kill it, then fall through to restore
├── live tmux sessions?
│   ├── exactly 1  → attach it
│   └── 2+         → numbered picker → attach
├── no live session, but snapshots exist?
│   ├── exactly 1  → restore it → attach
│   └── 2+         → numbered picker → restore → attach
└── nothing        → new session "main" → attach
(in all cases: background monitor is (re)started)
```

### Frozen tmux detection

A wedged tmux server makes `tmux info` hang forever, which is exactly when you
most need to recover your layout. On launch tmux-pilot probes the server with a
timeout (`PILOT_PROBE_TIMEOUT`, default 3s); if it doesn't answer it offers to
force-kill the server. Once killed, the normal flow restores your last snapshot —
layout and Claude sessions included.

## How it works

### Snapshot format

One file per tmux session: `~/.local/share/tmux-pilot/snapshots/<session>.snap` (TSV):

```
win   <session> <win_idx> <win_name> <layout> <active>
pane  <session> <win_idx> <pane_idx> <cwd> <cmd> <active> <claude_session_id>
```

`<claude_session_id>` is `-` when the pane isn't running Claude.

### Version history

Every time a snapshot's content actually changes, an immutable timestamped copy
is archived under `snapshots/history/<session>/<YYYYMMDD-HHMMSS>.snap`, so the
original state is always recoverable. Unchanged saves are not re-archived (no
noise). Inspect and roll back:

```bash
tmux-pilot history              # sessions that have history + version counts
tmux-pilot history <name>       # list all timestamped versions of a session
tmux-pilot history <name> <ts>  # promote a past version to the current snapshot
                                # (the current state is archived first, never lost)
```

Set `PILOT_HISTORY_ENABLED=0` to disable, `PILOT_HISTORY_KEEP=N` to cap retained
copies per session (default 50; 0 = unlimited).

### Claude session capture

`claude agents --json` returns `{pid, cwd, sessionId, status}` for every running
Claude. tmux-pilot walks each pane's process tree to map `pane_pid → claude pid →
sessionId`, so even **two Claudes in the same directory** are told apart precisely.

### Restore

Recreates windows/panes, `cd`s each pane to its saved cwd, re-applies the exact
tmux layout string, and:

- pane had a Claude session → `claude --resume <id>`
- pane ran nvim/vim/less/top/… → relaunches the program
- otherwise → just restores the working directory (won't blindly re-run unknown commands)

## Configuration

Environment variables:

| var | default | meaning |
|-----|---------|---------|
| `PILOT_DATA_DIR` | `~/.local/share/tmux-pilot` | data + snapshots + logs |
| `PILOT_SAVE_INTERVAL` | `60` | monitor save interval (seconds) |
| `PILOT_PROBE_TIMEOUT` | `3` | seconds to wait for tmux to respond before treating it as frozen |
| `PILOT_CLAUDE_CMD` | `claude` | CLI used for session discovery (`<cmd> agents --json`) and restore fallback. Point it at any Claude-compatible CLI. |
| `PILOT_CLAUDE_DATA_DIR` | `~/.claude` | data dir holding `projects/<cwd>/<sid>.jsonl`, used to tell empty sessions apart |
| `PILOT_KNOWN_CMDS` | `nvim vim vi nano less man top htop btop` | non-claude commands that get re-launched on restore (others: cd only) |
| `PILOT_HISTORY_ENABLED` | `1` | archive a timestamped copy on every content change (0 = off) |
| `PILOT_HISTORY_KEEP` | `50` | max archived copies per session (0 = unlimited) |

All config lives in one block at the top of `bin/tmux-pilot`. The launch command
(executable name plus flags like `--dangerously-skip-permissions`) is captured
from the running process at save time and replayed verbatim on restore, so
whatever CLI name and flags you used survive automatically — no executable name
is hardcoded.

## License

MIT

---

Powered by GLM-5.2
