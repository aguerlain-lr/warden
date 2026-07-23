---
name: setup-jail
description: Use from the HOST (unjailed) to provision a nono-jailed Claude Code session's config dir — installs the jail's command-policy config (git gating + interpreter auto-jail), a bash-read allowlist (safe because nono gates the filesystem), and the audit-permission-requests log hook. Targets the jail's config dir via $JAIL_DIR (e.g. ~/.claude-jail); the jailed agent never configures itself. Leans on the housekeep command-policy skill/engine. Idempotent.
---

# /setup-jail

Provisions the **jailed** Claude Code config dir, run from the **HOST** (an
unjailed session). The host writes into the jail's config tree so the jailed
agent never needs write access to its own config — tighter least-access.

The target is `$JAIL_DIR` (default `~/.claude-jail`), NOT the host's
`$CLAUDE_CONFIG_DIR`. Set `JAIL_DIR` explicitly if the jail lives elsewhere.

Prereq: the **housekeep** plugin is installed and its `command-policy` engine
binary is deployed into the jail dir (`$JAIL_DIR/command-policy/command-policy`).
If command-policy is not yet set up for the jail, run `/command-policy setup`
against the jail dir first, then this skill layers the jail-specific config on top.

## Step 0 — Refuse if jailed

Provisioning is a host act. If this session is itself inside a nono jail, bail —
a jailed session generally cannot write another config dir, and self-config is
exactly what this design avoids:

```bash
if [ -n "$NONO_CAP_FILE" ]; then
  echo "setup-jail: NONO_CAP_FILE is set — you appear to be INSIDE the jail."
  echo "Run this from an unjailed (host) Claude session."; exit 1
fi
```

Resolve + guard the target dir:

```bash
JAIL_DIR="${JAIL_DIR:-$HOME/.claude-jail}"
HOST_CFG="${CLAUDE_CONFIG_DIR:-$HOME/.claude}"
if [ "$JAIL_DIR" = "$HOST_CFG" ]; then
  echo "setup-jail: JAIL_DIR ($JAIL_DIR) equals the host's own config dir."
  echo "That would clobber the host with jail-only interpreter configs. Aborting."; exit 1
fi
echo "Provisioning jail config at: $JAIL_DIR"
```

The `jail-*` interpreter wrappers only exist under the nono broker; installing the
interpreter auto-jail configs into the host's config would break every interpreter.
The guard above enforces that they only land in the jail dir.

## Step 1 — Deploy the jail command-policy config

Copy this skill's payload into the jail's command-policy config dir. `<skill-root>`
is `${CLAUDE_PLUGIN_ROOT}/skills/setup-jail` when installed as a plugin.

```bash
JAIL_DIR="${JAIL_DIR:-$HOME/.claude-jail}"
mkdir -p "$JAIL_DIR/command-policy/config/examples"
cp <skill-root>/payload/config/git.json "$JAIL_DIR/command-policy/config/git.json"
cp <skill-root>/payload/config/examples/*.json "$JAIL_DIR/command-policy/config/examples/"
```

Then activate the interpreter configs (they ship as examples; the jail is the one
place they belong). For each of python, python3, node, perl, awk, copy the example
into the live config dir so the engine loads it:

```bash
JAIL_DIR="${JAIL_DIR:-$HOME/.claude-jail}"
for t in python python3 node perl awk; do
  cp "$JAIL_DIR/command-policy/config/examples/$t.json" "$JAIL_DIR/command-policy/config/$t.json"
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
JAIL_DIR="${JAIL_DIR:-$HOME/.claude-jail}"
TS=$(date +%Y%m%d-%H%M%S)
[ -f "$JAIL_DIR/settings.json" ] || echo '{}' > "$JAIL_DIR/settings.json"
cp "$JAIL_DIR/settings.json" "$JAIL_DIR/settings.json.bak.$TS"
jq '
  .permissions //= {} | .permissions.allow //= [] |
  .permissions.allow = ((.permissions.allow + [
    "Bash(cat:*)","Bash(find:*)","Bash(grep:*)","Bash(head:*)","Bash(ls:*)",
    "Bash(rg:*)","Bash(sed:*)","Bash(tail:*)","Bash(wc:*)"
  ]) | unique)
' "$JAIL_DIR/settings.json" > /tmp/jail-settings.new && mv /tmp/jail-settings.new "$JAIL_DIR/settings.json"
```

## Step 3 — Wire the audit-permission-requests log hook

(Own copy of housekeep's step-7b logic, config-dir-aware. The stored command uses a
literal `${CLAUDE_CONFIG_DIR:-$HOME/.claude}` so it expands **at fire time inside the
jailed session**, where `$CLAUDE_CONFIG_DIR` resolves to the jail dir — correct even
though the host wrote it.)

```bash
JAIL_DIR="${JAIL_DIR:-$HOME/.claude-jail}"
TS=$(date +%Y%m%d-%H%M%S)
cp "$JAIL_DIR/settings.json" "$JAIL_DIR/settings.json.bak.$TS"
LOG_CMD='jq --argjson noop 0 '"'"'. + {logged_at: (now | todate)}'"'"' >> "${CLAUDE_CONFIG_DIR:-$HOME/.claude}/permission-requests.log"'
jq --arg cmd "$LOG_CMD" '
  .hooks //= {} | .hooks.PermissionRequest //= [] |
  (.hooks.PermissionRequest |= map(select(((.hooks // []) | any(.command | test("permission-requests\\.log"))) | not))) |
  .hooks.PermissionRequest += [{ "hooks": [{"type":"command","command":$cmd,"async":true}] }]
' "$JAIL_DIR/settings.json" > /tmp/jail-settings.new && mv /tmp/jail-settings.new "$JAIL_DIR/settings.json"
```

## Step 3b — Ensure the slim MCP set (drop redundant servers)

nono already provides read-only, path-allowlisted filesystem access, so the
`local-search` MCP is redundant in the jail. Remove it from the jail's MCP
registration if present (idempotent — no-op if already absent):

```bash
JAIL_DIR="${JAIL_DIR:-$HOME/.claude-jail}"
CJ="$JAIL_DIR/.claude.json"
if [ -f "$CJ" ] && jq -e '.mcpServers["local-search"]' "$CJ" >/dev/null 2>&1; then
  TS=$(date +%Y%m%d-%H%M%S); cp "$CJ" "$CJ.bak.$TS"
  jq 'del(.mcpServers["local-search"])' "$CJ" > /tmp/cj.new && mv /tmp/cj.new "$CJ"
  echo "removed local-search from jail MCP set"
else
  echo "local-search not registered — slim MCP set OK"
fi
```

Do NOT run housekeep's `setup-housekeep` against the jail — it is the local-search
installer and would re-add the redundant server.

## Step 4 — Register the command-policy PreToolUse hook (if not already)

If `/command-policy setup` has not run against this jail dir, register the engine hook
(the engine binary is deployed by the command-policy skill). The stored command is a
literal `${CLAUDE_CONFIG_DIR:-$HOME/.claude}` — it expands in the jailed session at
fire time, so it points at the jail's engine regardless of the host writing it:

```bash
JAIL_DIR="${JAIL_DIR:-$HOME/.claude-jail}"
HOOK='${CLAUDE_CONFIG_DIR:-$HOME/.claude}/command-policy/command-policy'
jq --arg hook "$HOOK" '
  .hooks //= {} | .hooks.PreToolUse //= [] |
  (.hooks.PreToolUse |= map(select(((.hooks // []) | any(.command | test("command-policy/command-policy"))) | not))) |
  .hooks.PreToolUse += [{ "matcher":"Bash", "hooks":[{"type":"command","command":$hook}] }]
' "$JAIL_DIR/settings.json" > /tmp/jail-settings.new && mv /tmp/jail-settings.new "$JAIL_DIR/settings.json"
```

## Step 5 — Verify + tell the user

```bash
JAIL_DIR="${JAIL_DIR:-$HOME/.claude-jail}"
ls "$JAIL_DIR/command-policy/config"/*.json
jq '[.permissions.allow[] | select(startswith("Bash(") and test("cat|grep|rg"))] | length' "$JAIL_DIR/settings.json"
jq '[.hooks.PermissionRequest[] | select((.hooks//[])|any(.command|test("permission-requests\\.log")))] | length' "$JAIL_DIR/settings.json"
```
Expected: config lists git + interpreters; allow count ≥ 3; PermissionRequest count = 1. Tell the user to restart the jailed session and confirm the nono-side `jail-*` wrappers exist.

## Files this skill creates / modifies (all under the jail dir)
| Path | Action |
|---|---|
| `$JAIL_DIR/command-policy/config/*.json` | Jail command-policy configs (git + interpreters) |
| `$JAIL_DIR/settings.json` | bash-read allowlist + audit hook + command-policy hook |
| `$JAIL_DIR/.claude.json` | drops redundant `local-search` MCP (slim set) |
| `$JAIL_DIR/{settings.json,.claude.json}.bak.<TS>` | backups |
