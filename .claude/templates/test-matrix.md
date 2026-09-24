# Test Matrix Template

Built BEFORE tests are written (PIPELINE §7.1) — tests come FROM the matrix, not the other way
around. Once filled in, it is an epic artifact, e.g. `<epic-artifact-path>/test-matrix.md`.

**Rule.** Every contract requirement gets all three columns filled — a test id, or an explicit
`n/a — <reason>`. Never a blank cell: a blank cell is indistinguishable from "forgot to check".

| Requirement | Positive | Negative | Edge | Covered by |
|---|---|---|---|---|
| R-01 `<one-line requirement>` | `<test id>` | `<test id>` | `<test id>` | `<spec/suite file>` |
| R-02 `<...>` | `<test id>` | n/a — `<reason, e.g. "no error path exists for this input">` | `<test id>` | `<file>` |

- **Positive** — the requirement holds under a normal, expected input.
- **Negative** — the requirement's failure mode: an invalid input, a missing precondition, a
  denied permission. A requirement with no plausible negative case still gets an explicit
  `n/a — <reason>`, not a silent gap.
- **Edge** — a boundary the requirement's own wording implies (empty, maximum, zero, the
  smallest/largest value the type allows, a race between two callers).
- **Covered by** — the actual spec/suite file the id above lives in, so a reviewer jumps to the
  assertion instead of re-deriving it from the requirement text.

## Combination-matrix note

A matrix built requirement-by-requirement misses interaction bugs: two features that are each
individually correct can break only when combined with a third mode. Precedent class: a matrix
crossing two independent feature toggles against two independent delivery modes found a real
parity bug that no single-requirement row would have caught. When an epic ships more than one
orthogonal toggle, add a small pairwise combination table (`feature × mode`) beside the
requirement rows — it is cheap and it has already paid for itself once.

| | Mode A | Mode B |
|---|---|---|
| Feature 1 | `<test id / n/a>` | `<test id / n/a>` |
| Feature 2 | `<test id / n/a>` | `<test id / n/a>` |

## Rules

1. The matrix is built and reviewed BEFORE any test is written — tests are derived from its
   cells, not retrofitted to justify them.
2. Every requirement row has all three classes covered or explicitly `n/a` with a reason; a
   coding stage is not done while a row has an empty cell.
3. The matrix is an epic artifact — it ships alongside the code it verifies, and self-audit and
   acceptance both read it as evidence, not as a planning scratchpad that gets discarded.
