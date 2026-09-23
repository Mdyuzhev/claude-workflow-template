# /init — Session Init for `<PROJECT>`

Print: **Starting `<PROJECT>` session**

---

## Step 1 — Restore context from the checkpoint

The checkpoint channel may be down or undefined in this repository (PIPELINE §11). Source
of truth for state between sessions is the checkpoint contract:

```
Read .claude/<handoff>.json
```

Print `next_session_first_steps` from it. If the file does not exist — this is the epic's
first session; take context from `.claude/CLAUDE.md` (current state, epic registry, next
epic) instead.

## Step 2 — Infra health check (optional)

If an infra MCP is configured for this project, call its status/health tool and list any
service reporting down. Skip this step entirely if no infra MCP is configured — it is not a
hard dependency of session start.

## Step 3 — MCP liveness

Check the stdio MCP servers this project depends on (see `.claude/CLAUDE.md` § «MCP-реестр»):
banner/version of each, any `stale`/outdated-source flag, any `NEEDS-RESTART:` marker left by
the previous session's report. Do not assume liveness from memory — call the cheap liveness
gate/command named in the registry.

## Step 4 — Confirm epic state

Cross-check against `.claude/CLAUDE.md` (epic registry + next epic in queue). Confirm: is an
epic active, and if so, which phase of the autonomous cycle it is in — recon / scope /
fitch+self-audit / coding waves / close / acceptance / FINAL (PIPELINE §1).

---

Print: **`<PROJECT>` session open. Infra: [status or "not checked"]. Epic: [state]. Ready.**
