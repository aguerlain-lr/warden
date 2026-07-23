---
name: tune-nono
description: Use from the HOST (unjailed) to tune the nono jail profile — correlates nono kernel denials with the jailed Claude's permission-requests log, decides which layer each recurring failure belongs to (nono profile vs Claude settings), and drafts nono profile edits. MUST run host-side; nono's audit trail (~/.local/state/nono) is denied to the jailed agent by design. Delegates Claude-layer allowlist tuning to the housekeep audit-permission-requests skill.
---

# /tune-nono

HOST-ONLY. Reads the two gating layers and proposes fixes.

## Step 0 — Refuse if jailed

nono's state dir is unreadable inside the jail by design. Detect and bail:

```bash
if ! ls ~/.local/state/nono/audit >/dev/null 2>&1; then
  echo "tune-nono: cannot read ~/.local/state/nono — you appear to be INSIDE the jail."
  echo "Run this from an unjailed (host) Claude session."; exit 1
fi
```

## Step 1 — Enumerate the jail's recent nono sessions + denials

```bash
nono audit list --today
```
Pick the jailed-claude sessions (command contains `claude` or the `jail-*`
wrappers; cwd under `~/dev`). For each session id of interest, pull denial events:

```bash
nono logs <session-id> --json | jq -c 'select(.decision=="deny" or .type=="denial" or .event=="deny")'
```
(Field names vary by nono version — first run `nono logs <id> --json | jq -s '.[0]'`
to see the event shape, then filter on the denial marker you observe. Do NOT assume;
inspect.)

Aggregate the denied paths/operations across sessions into a frequency list.

## Step 2 — Read the jail's Claude-layer prompts

```bash
JAIL_CFG="$HOME/.claude-jail"   # today's jail config dir (pre-inversion)
jq -r '.tool_name + " | " + (.tool_input.command // .tool_input.file_path // "n/a")' \
  "$JAIL_CFG/permission-requests.log" 2>/dev/null | sort | uniq -c | sort -rn | head -20
```
If the log is absent, note it and continue with kernel denials only.

## Step 3 — Layer-routing (the core judgment)

For each recurring failure, decide the owner:

| Symptom | Layer | Fix |
|---|---|---|
| Kernel denial (EPERM/`nono why` = no capability) for a path the jail SHOULD reach | **nono** | Draft a `filesystem.allow`/`read` addition (Step 4) |
| Kernel denial for a network egress the jail needs | **nono** | Draft an `open_urls`/network rule |
| Claude prompt (in the log) for a safe read-only bash tool | **Claude** | Hand to `audit-permission-requests` — add to `permissions.allow` in the jail's settings |
| Claude prompt for a dual-nature tool (gh/kubectl/…) | **Claude** | Hand to `audit-permission-requests` → `/command-policy add <tool>` |
| BOTH a kernel denial and a Claude prompt for the same op | decide by **root cause** | If the kernel blocks it, no allowlist helps — fix nono first |

Never propose widening nono for something the Claude layer already refuses for good
reason (e.g. destructive ops). Least-access first: only draft the narrowest rule that
clears the observed denial.

## Step 4 — Draft nono profile edits (never promote)

The active profile dir is read-only; write drafts to `~/.config/nono/profile-drafts/`.
Follow the nono-sandbox skill's draft protocol: if a user profile
`~/.config/nono/profiles/claude-code-hardened.json` exists, read it, compute the
SHA-256 of the exact bytes, base the edit on it, and write that hash to
`~/.config/nono/profile-drafts/claude-code-hardened.base`. Run `nono profile guide`
for the schema. Write the full edited profile JSON to
`~/.config/nono/profile-drafts/claude-code-hardened.json`.

Tell the user:
> Drafted claude-code-hardened. Run `nono profile promote claude-code-hardened` to
> review and apply, then restart the jailed session.

## Step 5 — Delegate the Claude layer

For every row routed to "Claude" in Step 3, invoke `audit-permission-requests`
against the jail's `permission-requests.log` (pass `$HOME/.claude-jail/permission-requests.log`)
and apply its recommendations to the jail's `settings.json`. Do NOT reimplement
allowlist logic here.

## Output

A summary: (a) top kernel denials + the drafted nono rule for each, (b) Claude-layer
items handed to audit-permission-requests, (c) anything intentionally NOT widened and
why.
