# Audit Report Template

Lives at `<audits-path>/<date>-<scope>.md`. Uses the triangulation rule: 3+ independent sources
= hard blocker, 2 = probable, 1 = solo (lead-verified before it counts for anything).

A multi-role audit runs several subagents in parallel, each in its own role; the lead then
aggregates their findings into this single report.

---

# Audit: `<Scope>` — `<Date>`

**Date:** `<YYYY-MM-DD>`
**Scope:** `<what is being audited — an epic's fitch package, one module, the whole codebase>`,
`<size: lines of code / number of task files / etc>`.
**Scale:** `<Mini | Small | Medium | Large>` (drives which roles run — see "Scope sizing" below)
**Sources:** `<self-inspection (lead) + architect + engineer + skeptic + security + qa, as applicable>`

## Summary

1-2 honest paragraphs:
- What is genuinely good in the audited artifact.
- What blocks it from moving forward.
- The main systemic signal — something that repeats across several findings and points at a
  root cause, not just a list of unrelated bugs.

Example shape:
> The epic is well structured — N waves in a sound order, the coverage matrix formally complete,
> file isolation declared. But literal verification found two hard blockers (one wave describes
> work already done elsewhere; one referenced task file does not physically exist) plus a series
> of implementation blockers in the platform-facing tasks.
>
> Main systemic signal: the affected wave passed through the analyst/tech-lead/QA fitch phase
> citing "real code was read" — but that verification never actually ran for this file. That is
> a violation of the project's own "never guess a signature from memory" rule; the fix is a full
> grep-pass over every task file before the next wave starts.

---

## Triangulated findings (3+ sources) — hard blockers

Confirmed by three or more roles. **Must fix before merge.**

### FAIL B1 — `<one-line summary>`

**Where:**
- `<file:line>` — `<one-line context>`
- `<another file:line>` — `<context>`
- `<contract / fitch-plan reference>` — `<claim that contradicts reality>`

**Agree:** architect (`<finding ref>`), engineer (`<ref>`), skeptic (`<ref>`), self-inspection.

**Risk:**
1. `<concrete failure scenario 1>`
2. `<scenario 2>`

**Fix:** `<concrete, actionable resolution — a command or an edit description>`.

---

### FAIL B2 — `<...>` (same structure)

---

## Probable findings (2 sources, code-verified)

Confirmed by two roles plus the lead's own verification against the real file. **P1 priority —
fix if it doesn't block merge, otherwise in the first hotfix wave.**

### WARN P1 — `<summary>`

**Where:** `<refs>`. **Agree:** `<two role refs>`, verified by self-inspection.
**Risk:** `<scenario>`. **Fix:** `<resolution>`.

---

## Solo findings (lead-verified)

One source, but the lead confirmed it against real code. **P2 — backlog, or a quick fix if
cheap.**

### WARN S1 — `<summary>`

**Source:** `<single role>`. **Verified:** reading `<file>` confirms `<observation>`.
**Scenario / risk:** `<concrete>`. **Impact:** `<who is affected, blast radius>`.
**Fix:** `<resolution>`.

---

## REJECT — under dispute

Findings the skeptic or security role proposed to reject. Both positions plus the lead's ruling.

### `<finding name>`

**Skeptic / security:** `<position>`. **Counter-argument:** `<other position>`.
**Lead's ruling:** `<reject | accept | partial accept as a backlog item>`.

---

## Backlog / dropped

Single-source, non-critical, or needing code verification the lead did not perform this round —
documented as a possible risk, not fixed in this epic. Follow-up ownership sits with the owner.

- `<finding>` (`<source ref>`). `<one-line context>`. Cost: `<estimate>`. Pickup: `<epic id | next epic>`.

---

## Sources detail

### Architect summary (`<N>` findings)
Structural focus. Top: `<B-list>`, `<P-list>`.

### Engineer summary (`<N>` findings)
Implementation focus. Top: `<list>`. Verified against real code at `<file refs>`.

### Skeptic summary (`<N>` findings)
Composition and REJECT-filter. Top: `<list>`.

### Security summary (`<N>` findings, OWASP-mapped)
Threat-model focus. Top: `<list>`. Maps to OWASP `<A03 / A09 / A10>`.

### QA summary (`<N>` findings + matrix verification)
Coverage-gap focus. Top: `<list>`. Coverage matrix verification: `<hard gaps + weak assertions
count>`.

---

## Action list before W1

A concrete, ordered plan to close blockers before the epic starts:

1. `<Action 1>` — `<one-line>`.
2. `<Action 2>` — `<one-line>`.
3. `<Action N>` — `<one-line>`.

After these changes, W1 can start understanding that an empty "uncovered requirements" list is
still formal — real coverage is proven only once `<the relevant infrastructure>` is in place.

---

## Triangulation rule mechanics

- **1 source** = weak signal. Any single role can misdiagnose or defend its own first read. A
  solo finding is documented but never acted on without lead verification.
- **2 sources** = probable. P1 priority if it isn't a blocker.
- **3+ sources** = hard blocker. Must fix.

Composition bugs — individual findings that chain into a real attack path across modules — may
not triangulate inside any one module but visually compose into one real problem. Triangulation
does not catch these automatically; the lead runs a synthesis pass for them.

## Scope sizing

**Mini** — a single file/function. 1 role. 5-15 minutes.
**Small** — a single module, 5-10 files. 2 roles (architect + engineer). 30-60 minutes.
**Medium** — a feature/epic, 20-50 files. 3 roles (+ skeptic). 1-2 hours.
**Large** — multi-module / cross-cutting / a full fitch package. 5 roles (+ security + QA). 2-4h.

Wall-clock is parallel — subagents start together, the lead synthesizes after.
