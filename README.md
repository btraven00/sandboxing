# sandboxing

Notes and scripts for running agents (Claude Code, etc.) in a box.

## agent-box

`agent-box` — bubblewrap wrapper. Lives here; install with `ln -s "$PWD/agent-box" ~/.local/bin/`.

```sh
agent-box claude            # every namespace in the config
agent-box -n lab bash       # only [lab] (plus the top lines)
```

### Config: `~/.clankerjail`

Your home directory is empty inside the box. The config lists which folders the box gets to see.
It must exist, or `agent-box` refuses to start. Use `CLANKERJAIL=/some/other/file` to point at a
different one.

```
# shared with every box
~/.claude
~/.claude.json

[phd]
~/phd
~/phd/data:ro

[lab]
~/lab
~/datasets:ro
```

- **One path per line.** `~` is expanded. Blank lines and lines starting with `#` are ignored.
- **Lines before the first `[section]`** are shared with every box. Keep `~/.claude` and
  `~/.claude.json` there, or Claude Code has no login or settings inside.
- **`[name]` sections are namespaces.** `agent-box -n phd claude` shares the top lines plus
  `[phd]` and nothing else. Without `-n`, every section is shared. An unknown name is an error.
- **Paths are read-write.** Add `:ro` to the end of a path to make it read-only.
- **A subfolder overrides its parent.** With `~/phd` and `~/phd/data:ro`, the agent can write
  anywhere in `~/phd` except `data`. Line order doesn't matter.
- **Every path must exist.** A missing one (often a typo) stops the box from starting.

Typical setup: alias your default namespace, and name another one when the work needs it.

```sh
alias claude='agent-box -n phd claude'
agent-box -n lab claude
```

### Profiles

A profile is a whole separate config: `agent-box -p NAME` reads `~/.clankerjail.NAME` instead of
`~/.clankerjail` (same format, its own top lines and sections). Combine with `-n` in any order:
`agent-box -p personal -n blog claude`. `~/.clankerjail` stays the default profile.

### What the box sees

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

### Caveats

- A tool that breaks usually needs its data dir: add it to the toolchain list in the script.
- `julia` via juliaup fails (wants to write a lockfile in `~/.juliaup`).
- Claude can't auto-update inside, so `agent-box claude` runs `claude update` on the host before entering the box.
- Network is not isolated (no `--unshare-net`).
- Tools that want to write caches under `~` (cargo, pixi, go) write to the tmpfs and lose them on exit.
