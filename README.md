# warden

Skills for running Claude Code inside a `nono` security
sandbox — a hardened, "jailed" agent that can't wander outside a tight
filesystem + network allow-list, with a second gating layer on dangerous tool
commands. Both skills are **host-run**: an unjailed session provisions and tunes
the jail; the jailed agent never configures itself.

## The setup this supports

Two independent gating layers protect the jailed session:

1. **nono (kernel layer).** A nono profile allow-lists the paths and network the
   jailed process may touch; everything else is denied by Landlock/Seatbelt at
   the kernel. There is no escalation from inside the jail.
2. **command-policy + Claude permissions (agent layer).** Even for operations
   nono permits, the housekeep `command-policy` `PreToolUse` hook gates
   dangerous tool subcommands (git, gh, kubectl, …) and rewrites reflex
   interpreter calls (`python`, `node`, …) to nono-brokered `jail-*` wrappers.
   Claude's own `permissions.allow` silences safe read-only bash so the jail
   isn't noisy.

The moving pieces:

| Piece | Where | Role |
|---|---|---|
| **Host Claude** | `~/.claude` (unjailed) | Runs warden's `setup-jail` + `tune-nono`. |
| **Jailed Claude** | `CLAUDE_CONFIG_DIR=~/.claude-jail`, launched under nono | The sandboxed agent. Provisioned by the host. |
| **housekeep `command-policy`** | deployed into the jail dir | The agent-layer gating engine + `PreToolUse` hook. |
| **nono profile** | `~/.config/nono/profiles/<profile>.json` | The kernel-layer allow-list. |

## Skills

- **setup-jail** — HOST-ONLY. Installs the jail's command-policy config, a
  bash-read allowlist, and the audit-log hook into the jail's config dir
  (`$JAIL_DIR`, default `~/.claude-jail`). The host provisions the jail; the
  jailed agent never configures itself. Leans on the housekeep
  `command-policy` skill.
- **tune-nono** — HOST-ONLY. Correlates `nono` kernel denials
  (`~/.local/state/nono`, unreadable from inside the jail) with the jail's
  `permission-requests.log`, decides which layer each fix belongs in, and
  drafts nono profile edits to `~/.config/nono/profile-drafts/`.

## Install

```
/plugin marketplace add aguerlain-lr/warden
/plugin install warden@warden
```

Prereq: the [housekeep](https://github.com/aguerlain-lr/housekeep) plugin, whose
`command-policy` engine warden leans on.

## First-time setup

1. **Install nono** and create a hardened profile for the jail (see nono's own
   docs / `nono profile guide`). This is the kernel allow-list — warden does not
   manage it beyond drafting edits via `tune-nono`.
2. **Install housekeep** on the host and deploy its `command-policy` engine into
   the jail dir (`/command-policy setup` targeting `~/.claude-jail`).
3. **Install warden** on the host (above).
4. **Provision the jail** from an unjailed host session:
   ```
   /setup-jail
   ```
   (Set `JAIL_DIR` first if your jail dir isn't `~/.claude-jail`.)
5. **Add a launch alias** (below) and start the jailed session.
6. **Iterate.** When the jail hits a wall, run `/tune-nono` from the host to see
   which layer denied it and draft the narrowest fix.

## Recommended `.zshrc` alias

Launch the jailed agent with its own config dir, under the nono profile:

```zsh
# Jailed Claude Code — hardened nono sandbox + command-policy agent gating
alias claude-jail='CLAUDE_CONFIG_DIR="$HOME/.claude-jail" nono run --profile <profile> -- claude'
```

Replace `<profile>` with your hardened nono profile name and adjust the jail dir
if you use a different one. The jailed session then reads `~/.claude-jail`
(provisioned by `setup-jail`) while your host `claude` keeps using `~/.claude`.

## Config-dir layout

Config-dir inversion (jail → `~/.claude`, host → `~/.claude-warden`) is a
separate effort; these skills are `CLAUDE_CONFIG_DIR`-aware and work under either
layout.

## License

MIT
