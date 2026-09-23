> hint: one chain-brief per chain, written once by the lead before launch,
> folding in the owner-level brief that preceded it. Fill every
> `<placeholder>`; delete this hint line after the first fill.

# Chain brief — `<CHAIN-ID>`: `<one-line mission>`

## 0. Mission

`<What this chain delivers, in one paragraph. No product feature detail
beyond what a fresh, memory-less session needs to start.>`

## 1. Base

- **Base commit/branch:** `<sha-or-branch>` — do not rebase past this without
  a logged reason.
- **Worktree:** `<path>`
- **Roles:** orchestrator = `<model/provider>`; card executor = `<model/provider>`;
  exactly one tool drives the process at a time.
- **Owner decisions folded in (verbatim where it matters):** `<D-NN: ...>`

## 2. Invariants (verified on the base sha above)

Each invariant carries a `path:line` anchor checked against the base sha —
an invariant without an anchor is a guess, not a fact.

- `<invariant 1>` — `<path:line>`
- `<invariant 2>` — `<path:line>`

## 3. Scope

**In scope:** `<bullet list>`

**Out of scope — never picked up along the way:** `<bullet list>`. A scope
question outside this list is a stop condition (`AWAIT-LEAD:` below), not a
silent judgment call.

## 4. Ledger of links

| Link | Gate | Status |
|---|---|---|
| `<link-1>` | `<one command, expected exit>` | pending |
| `<link-2>` | `<one command, expected exit>` | pending |

## 5. Moderation checklist (named failures from the prior run)

Numbered, near-verbatim citations of prior incidents, reformulated as rules
this run must not repeat:

1. `<failure class 1> → <rule>`
2. `<failure class 2> → <rule>`

## 6. Stop menu

Exactly three ways to hand back control (full sentinel dictionary —
docs/12-external-contours-and-chains.md, section on sentinels and RESUME):

- `HALT: <reason>` — unsafe to work around, needs a decision before
  continuing.
- `AWAIT-LEAD: <what>` — a checkpoint the lead must review; work not gated
  by it continues.
- `CHAIN-DONE: <summary>` — nothing left to do.

**Canonical resume line** (fixed string — do not retype from memory):

> `RESUME: continue autonomously per chain-brief; recover state from
> progress.md + decisions.md; do not re-read code you already changed.`

## 7. Intervention budget

`<N interventions expected per link; what counts as one>`

## 8. Evidence rules

Every gate number ships with a committed transcript (UTF-8/LF; capture
recipe — kit/RULES.md §14(в) and docs/10-gates-and-evidence.md §4) — a
number without one does not exist.

## 9. Close conditions

- All ledger links `CHAIN-DONE` with a green gate transcript each.
- `<acceptance pack for the owner, if the chain ships user-facing behaviour — see docs/11-acceptance-and-final.md>`
- Self-acceptance pass complete (`kit/skills/pr-with-proofs/SKILL.md` §4c).
