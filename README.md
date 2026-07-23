# warden

Skills for the nono-sandboxed ("jailed") Claude Code.

- **setup-jail** — HOST-ONLY. Installs the jail's command-policy config, a
  bash-read allowlist, and the audit-log hook into the jail's config dir
  (`$JAIL_DIR`, default `~/.claude-jail`). The host provisions the jail; the
  jailed agent never configures itself. Leans on the housekeep
  `command-policy` skill.
- **tune-nono** — HOST-ONLY. Correlates `nono` kernel denials
  (`~/.local/state/nono`, unreadable from inside the jail) with the jail's
  `permission-requests.log`, decides which layer each fix belongs in, and
  drafts nono profile edits to `~/.config/nono/profile-drafts/`.

Install:
```
/plugin marketplace add aguerlain-lr/warden
/plugin install warden@warden
```

Config-dir inversion (jail → `~/.claude`, host → `~/.claude-warden`) is a
separate effort; these skills are `CLAUDE_CONFIG_DIR`-aware and work under
either layout.
