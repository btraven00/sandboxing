# sandboxing

Notes and scripts for running agents (Claude Code, etc.) in a box.

## agent-box

`agent-box` — bubblewrap wrapper. Lives here; install with `ln -s "$PWD/agent-box" ~/.local/bin/`.

```sh
agent-box claude            # every namespace in the config
agent-box -n lab bash       # only [lab] (plus the top lines)
alias claude='agent-box -n phd claude'
```

Read-write dirs come from `~/.clankerjail` (or `$CLANKERJAIL`). Lines before any `[section]` are
always bound; `[name]` sections are namespaces, picked with `-n name`. Suffix `:ro` for read-only; a child overrides its parent
(`~/phd` + `~/phd/data:ro` = writable phd with a read-only `data`), in any line order.

```
~/.claude
~/.claude.json

[phd]
~/phd

[lab]
~/lab
~/data:ro
```

What the box sees:

| path | access |
|------|--------|
| `/` | read-only |
| `/tmp`, `~` | empty tmpfs (wiped on exit) |
| paths from `~/.clankerjail` (global + chosen namespace) | read-write (or `:ro`) |
| each `$PATH` dir under `~` + targets of symlinks in it | read-only |
| toolchains: `~/.rustup`, `~/.juliaup`, `~/.julia`, go's GOROOT | read-only |
| `agent-box` itself, `~/.claude/{settings.json,hooks,plugins}` | read-only (no editing the box or host-side hooks) |

Own PID namespace + fresh `/proc` (can't see or signal host processes), dies with parent.
`AGENT_BOX=1` is set inside.

Caveats:

- A tool that breaks usually needs its data dir: add it to the toolchain list in the script.
- `julia` via juliaup fails (wants to write a lockfile in `~/.juliaup`).
- Claude can't auto-update inside, so `agent-box claude` runs `claude update` on the host before entering the box.
- Network is not isolated (no `--unshare-net`).
- Tools that want to write caches under `~` (cargo, pixi, go) write to the tmpfs and lose them on exit.
