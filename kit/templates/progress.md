> hint: one append-only log per chain (or per contributor sharing a chain).
> Never rewrite an existing entry — only append. Fill every `<placeholder>`;
> delete this hint line after the first fill.

# `<CHAIN-ID>` — progress

Append-only. Stamps are event times (UTC, `YYYY-MM-DDTHH:MM:SSZ`). A wave/link
report is an entry, never the end of the turn
(docs/12-external-contours-and-chains.md § «Модель цепочки: звено, ветка, PR»).

## Entry 0 — `<UTC-stamp>` — start

**Commit:** pre-code. This entry is the start marker; the first commit lands
with the next entry.

**Start notification RESULT-line (if the project has one; verbatim, last
stdout line):**

```
<{"relayed":..., "event":"start", ...} — or "mcp: unavailable(<reason>)">
```

**Environment record:**

- `<runtime/tooling versions>`
- orchestrator: `<agent/provider/model/variant>`
- card executor: `<agent/provider/model/variant>`
- shell: `<shell + version>`
- hook: `<armed|disarmed>` (if the project uses an auto-resume hook)
- `mcp:` `<server> alive (<ping> OK, N <units>)` — or the honest
  unavailable-reason form (`kit/skills/setup-env/SKILL.md` §6)

**State in words:** `<worktree, branch, base sha, sources read, open
questions>`

**Next step:** `<first unit of work>`

## Entry `<N>` — `<UTC-stamp>` — `<short state label>`

**Commit:** `<sha>`

**Done:** `<what, which gates passed>`

**Next step:** `<...>`

---

**Sentinel line examples** — end a message, or the final entry, with
exactly one of these three (docs/12-external-contours-and-chains.md, section
on sentinels and RESUME):

```
HALT: <reason>
AWAIT-LEAD: <what>
CHAIN-DONE: <summary>
```
