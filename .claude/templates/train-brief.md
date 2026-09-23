# Merge Train Brief Template

A merge-train brief is a short, versioned planning document the lead writes once, before a batch
of independently-developed chains/PRs is folded into one release. Each chain's own
self-contained chain-brief inherits from this frame. Lives at `<train-artifact-path>/train-brief.md`.

Frame items are numbered `T-1`...`T-N` as generic preconditions and rules — fill every one in,
delete none silently (an item that does not apply gets `n/a — <reason>`, not a blank).

---

# Train `<version/slug>` brief

## Preconditions (machine-checkable, gate before anything starts)

- **T-1** Previous release fully shipped and tagged, both locally and on the shared remote.
  Check: `<git command>`. If false — STOP here, do not proceed or work around it.
- **T-2** Working tree clean except the always-present untracked set: `<list, or "none">`.
- **T-3** Long-lived tool processes (agent-driver / test-runner / knowledge-base MCP servers,
  background runners) restarted recently enough to see the previous train's changes. Check:
  `<liveness command>`.

## Contour and model allocation

- **Orchestrator:** `<role, model>`.
- **Delegate(s):** `<role, model, per chain>` — exactly one thing drives the process at a time;
  state which safety switch is on by default (e.g. auto-resume off unless explicitly armed).
- **Subagent model split:** `<e.g. "the orchestrator's own subagents: an explicitly named model
  per RULES R6, never the tool default">`.

## Moderator role

Reformulated ownership boundary: only the lead performs mechanically risky steps — regression
sweeps, environment deploys, repository-history operations (merge, tag, push, branch delete). A
delegate never performs these directly, regardless of how ready it believes its own work is.

## Numbering / ordering reservations

`<Exactly which identifiers — feature/task numbers, spec/test numbers, test-environment keys,
similar namespaces — the upcoming chains may claim, checked by a fresh search at brief time, not
assumed from memory. Documented rollback rule: if a slot is already taken, the next free one is
used and the choice is recorded as a line here.>`

## Cumulative lessons checklist

`<Each item — a concrete failure from a previous run, restated as an imperative rule, copied
near-verbatim into the next chain-brief so it never needs rediscovering.>`

- `<lesson 1>`
- `<lesson 2>`

## Known tooling defects

`<Anything broken in the tooling itself right now, and how the moderator compensates for it
until it is fixed.>`

## Acceptance package required from each delegate

- Every gate rerun independently by the lead, with a transcript the lead regenerated.
- Every claimed artifact independently verified to exist at the delivered head.
- A concrete measurement/benchmark table wherever the chain claims a performance number.
- An impact check on downstream consumers for any change with external callers.

## FINAL — two parts

- **Part 1 (no push):** merge, tag, internal knowledge-base update, registries — everything the
  lead can do without push-time risk.
- **Part 2 (push, only after operator restart + green health):** the operator restarts their own
  client/tools; every live health check must come back clean; only then is the branch actually
  pushed. A red health check after part 1 stops the process before push — nothing is pushed by
  half.

## Explicit commands

`<The literal phrases the human operator will type to launch each stage, so the lead never
guesses the boundary of its own authority.>`

- Start: `"<phrase>"`
- Accept chain N: `"<phrase>"`
- FINAL part 1: `"<phrase>"`
- FINAL part 2 (push): `"<phrase>"`

---

## §A — Epic A: `<name>`

`<One paragraph: scope, chain-brief pointer, owning contour.>`

## §B — Epic B: `<name>`

`<One paragraph: scope, chain-brief pointer, owning contour.>`

## §C — Commands to launch

```bash
<command to start epic A's chain>
<command to start epic B's chain>
```
