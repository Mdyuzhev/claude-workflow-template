# Acceptance Report Template

The lead's acceptance of externally produced work — a PR, a chain, a submitted link — against a
tested head. Lives at `<epic-or-chain-artifact-path>/acceptance-r<N>-lead.md`.

---

# Acceptance round r`<N>` — `<EPIC-ID / chain slug>`

**Verdict:** `<ACCEPTED | NOT ACCEPTED as-is | ACCEPTED with residue>`

**Tested head:** `<hash>` (branch `<name>`) — verified by `<the actual git command run, e.g.
"git rev-parse HEAD" / "git log -1 --format=%H">`, never by reading the contributor's own claim.

## Independent rerun

Every gate below was rerun by the lead ON the tested head above, from a transcript the lead
regenerated itself — never a re-statement of the contributor's own numbers.

| Gate | Command | Result | Transcript |
|---|---|---|---|
| `<type-check>` | `<command>` | `<N passed / M failed>` | `<path>` |
| `<unit tests>` | `<command>` | `<N passed / M failed / K skipped>` | `<path>` |
| `<full sweep>` | `<command>` | `<N passed / M failed / K skipped>` | `<path>` |

## Walked the user path

`<For every new user-facing capability: which UI entry points were physically exercised — not
only the path the contributor's own spec drives. Switcher / creation button / menu / wherever a
real user reaches it. State each entry point and its result.>`

## Findings

### Blockers (break core function or an invariant — fix before this work is looked at again)

- `<finding>` — `<file:line / evidence>`.

### Required before merge (must close, but "not everything is broken")

- `<finding>`.

### Residue (recorded, not blocking)

- `<finding>` — `<fixed now at the lead's discretion | left for an explicit owner decision>`.

## Report-layer vs. code-layer verdict

State separately whether the CODE is accepted and whether the CONTRIBUTOR'S OWN REPORT about
that code is accurate. These are independent axes — code can be correct while its report
contains a fabricated number or a narrative that does not match the artifact it cites (precedent
class: a committed transcript's own hand-written header claimed a fully green run while the same
file's body recorded a failure).

- **Code:** `<accepted | not accepted>` — `<why>`.
- **Report layer:** `<accurate | contains N discrepancies, listed above>`.

## Session-level acceptance (only if the work came from an unattended external contour)

- Intervention budget: `<used/allowed>`.
- Forbidden commands / silent gaps: `<none found | list>`.
- Verdict: `<within threshold | over threshold>`.

## Decisions requested from the owner

`<a decision, if any; otherwise "none — the lead ruled within its own authority">`.

## What happens next

`<a fix-wave by the same external contour | a fix-wave performed by the lead directly, by owner
directive | merge into the train | close>`.
