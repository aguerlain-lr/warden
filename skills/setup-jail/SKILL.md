---
name: setup-jail
description: Use when provisioning a nono-jailed Claude Code session's config dir — installs the jail's command-policy config (git gating + interpreter auto-jail), a bash-read allowlist (safe because nono gates the filesystem), and the audit-permission-requests log hook. Installs into the JAIL's config dir ($CLAUDE_CONFIG_DIR, e.g. ~/.claude-jail). Leans on the housekeep command-policy skill/engine. Idempotent.
---

# /setup-jail

Provisions the **jailed** Claude Code config dir. Run this from INSIDE the jail
(or with `CLAUDE_CONFIG_DIR` pointed at the jail's dir), so `$CLAUDE_CONFIG_DIR`
resolves to the jail's config tree.

Prereq: the **housekeep** plugin is installed (this skill leans on its
`command-policy` engine + skill). If command-policy is not yet set up, run
`/command-policy setup` first, then this skill layers the jail-specific config on top.

Resolve the config dir once:

```bash
CONFIG_DIR="${CLAUDE_CONFIG_DIR:-$HOME/.claude}"
echo "Provisioning jail config at: $CONFIG_DIR"
```

If `$CONFIG_DIR` is the host default (`$HOME/.claude`) AND you are not mid-inversion,
STOP and warn the user: the interpreter auto-jail configs must NOT land in the host's
default config (the `jail-*` wrappers only exist under the nono broker; installing
them unjailed breaks every interpreter). Only proceed if `$CONFIG_DIR` is the jail's
dir (e.g. `~/.claude-jail`).

## Step 1 — Deploy the jail command-policy config

Copy this skill's payload into the jail's command-policy config dir. `<skill-root>`
is `${CLAUDE_PLUGIN_ROOT}/skills/setup-jail` when installed as a plugin.

```bash
CONFIG_DIR="${CLAUDE_CONFIG_DIR:-$HOME/.claude}"
mkdir -p "$CONFIG_DIR/command-policy/config/examples"
cp <skill-root>/payload/config/git.json "$CONFIG_DIR/command-policy/config/git.json"
cp <skill-root>/payload/config/examples/*.json "$CONFIG_DIR/command-policy/config/examples/"
```

Then activate the interpreter configs (they ship as examples; the jail is the one
place they belong). For each of python, python3, node, perl, awk, copy the example
into the live config dir so the engine loads it:

```bash
CONFIG_DIR="${CLAUDE_CONFIG_DIR:-$HOME/.claude}"
for t in python python3 node perl awk; do
  cp "$CONFIG_DIR/command-policy/config/examples/$t.json" "$CONFIG_DIR/command-policy/config/$t.json"
done
```

These rewrite reflex interpreter calls to `jail-<lang>` wrappers brokered by nono
(no network, repo read-only, scratch read-write). The nono side (wrappers +
`command_policies`) is provisioned in the nono profile, out of band — see your
nono interpreter-jail design (the `command_policies` broker + `jail-*` wrappers).
This skill only does the hook side.

## Step 2 — Bash-read allowlist

Safe read-only shell tools should not prompt in the jail — nono already gates the
filesystem, so their blast radius is bounded. Add them to the jail's settings allow:

```bash
CONFIG_DIR="${CLAUDE_CONFIG_DIR:-$HOME/.claude}"
TS=$(date +%Y%m%d-%H%M%S)
[ -f "$CONFIG_DIR/settings.json" ] || echo '{}' > "$CONFIG_DIR/settings.json"
cp "$CONFIG_DIR/settings.json" "$CONFIG_DIR/settings.json.bak.$TS"
jq '
  .permissions //= {} | .permissions.allow //= [] |
  .permissions.allow = ((.permissions.allow + [
    "Bash(cat:*)","Bash(find:*)","Bash(grep:*)","Bash(head:*)","Bash(ls:*)",
    "Bash(rg:*)","Bash(sed:*)","Bash(tail:*)","Bash(wc:*)"
  ]) | unique)
' "$CONFIG_DIR/settings.json" > /tmp/jail-settings.new && mv /tmp/jail-settings.new "$CONFIG_DIR/settings.json"
```

## Step 3 — Wire the audit-permission-requests log hook

(Own copy of housekeep's step-7b logic, config-dir-aware. The stored command uses a
literal `${CLAUDE_CONFIG_DIR:-$HOME/.claude}` so it expands per-env at fire time.)

```bash
CONFIG_DIR="${CLAUDE_CONFIG_DIR:-$HOME/.claude}"
TS=$(date +%Y%m%d-%H%M%S)
cp "$CONFIG_DIR/settings.json" "$CONFIG_DIR/settings.json.bak.$TS"
LOG_CMD='jq --argjson noop 0 '"'"'. + {logged_at: (now | todate)}'"'"' >> "${CLAUDE_CONFIG_DIR:-$HOME/.claude}/permission-requests.log"'
jq --arg cmd "$LOG_CMD" '
  .hooks //= {} | .hooks.PermissionRequest //= [] |
  (.hooks.PermissionRequest |= map(select(((.hooks // []) | any(.command | test("permission-requests\\.log"))) | not))) |
  .hooks.PermissionRequest += [{ "hooks": [{"type":"command","command":$cmd,"async":true}] }]
' "$CONFIG_DIR/settings.json" > /tmp/jail-settings.new && mv /tmp/jail-settings.new "$CONFIG_DIR/settings.json"
```

## Step 3b — Ensure the slim MCP set (drop redundant servers)

nono already provides read-only, path-allowlisted filesystem access, so the
`local-search` MCP is redundant in the jail. Remove it from the jail's MCP
registration if present (idempotent — no-op if already absent):

```bash
CONFIG_DIR="${CLAUDE_CONFIG_DIR:-$HOME/.claude}"
CJ="$CONFIG_DIR/.claude.json"
if [ -f "$CJ" ] && jq -e '.mcpServers["local-search"]' "$CJ" >/dev/null 2>&1; then
  TS=$(date +%Y%m%d-%H%M%S); cp "$CJ" "$CJ.bak.$TS"
  jq 'del(.mcpServers["local-search"])' "$CJ" > /tmp/cj.new && mv /tmp/cj.new "$CJ"
  echo "removed local-search from jail MCP set"
else
  echo "local-search not registered — slim MCP set OK"
fi
```

Do NOT run housekeep's `setup-housekeep` in the jail — it is the local-search
installer and would re-add the redundant server.

## Step 4 — Register the command-policy PreToolUse hook (if not already)

If `/command-policy setup` has not run in this config dir, register the engine hook
(the engine binary is deployed by the command-policy skill):

```bash
CONFIG_DIR="${CLAUDE_CONFIG_DIR:-$HOME/.claude}"
HOOK='${CLAUDE_CONFIG_DIR:-$HOME/.claude}/command-policy/command-policy'
jq --arg hook "$HOOK" '
  .hooks //= {} | .hooks.PreToolUse //= [] |
  (.hooks.PreToolUse |= map(select(((.hooks // []) | any(.command | test("command-policy/command-policy"))) | not))) |
  .hooks.PreToolUse += [{ "matcher":"Bash", "hooks":[{"type":"command","command":$hook}] }]
' "$CONFIG_DIR/settings.json" > /tmp/jail-settings.new && mv /tmp/jail-settings.new "$CONFIG_DIR/settings.json"
```

## Step 5 — Verify + tell the user

```bash
CONFIG_DIR="${CLAUDE_CONFIG_DIR:-$HOME/.claude}"
ls "$CONFIG_DIR/command-policy/config"/*.json
jq '[.permissions.allow[] | select(startswith("Bash(") and test("cat|grep|rg"))] | length' "$CONFIG_DIR/settings.json"
jq '[.hooks.PermissionRequest[] | select((.hooks//[])|any(.command|test("permission-requests\\.log")))] | length' "$CONFIG_DIR/settings.json"
```
Expected: config lists git + interpreters; allow count ≥ 3; PermissionRequest count = 1. Tell the user to restart the jailed session and confirm the nono-side `jail-*` wrappers exist.

## Files this skill creates / modifies
| Path | Action |
|---|---|
| `$CONFIG_DIR/command-policy/config/*.json` | Jail command-policy configs (git + interpreters) |
| `$CONFIG_DIR/settings.json` | bash-read allowlist + audit hook + command-policy hook |
| `$CONFIG_DIR/.claude.json` | drops redundant `local-search` MCP (slim set) |
| `$CONFIG_DIR/{settings.json,.claude.json}.bak.<TS>` | backups |
