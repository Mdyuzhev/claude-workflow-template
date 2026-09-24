# Self-audit A-G prompt — template

> hint: prompt for the **self-audit A-G by a fresh adversarial subagent** (PIPELINE §6). Run
> **BEFORE W1** (before coding starts), inside the CC orchestrator. Fill the `<…>` placeholders,
> delete this hint. The subagent is strictly Sonnet ([RULES.md](../RULES.md) R6), an explicit
> model parameter on spawn; "fresh" means no memory of the fitch/brief, so it does not inherit
> the contract author's blind spots.

## Mandate

**Audit the package at `<path to the fitch package>` along 7 directions.** The package =
`contract.json` + `FITCH-PLAN.md`/`PLAN.md` + task files. Check signatures and paths against
**real code on disk** (`<base HEAD loose-ref>`), not against the contract's own description.
Goal — catch defects BEFORE they become a red gate during coding.

## 7 directions

> Tag every finding with its direction letter for traceability.

- **A Coverage** — every requirement (`R<NN>`) is covered by a task AND a QA check; no orphaned
  requirements, no tasks without a requirement.
- **B Feasibility** — function/file signatures are checked against real code; every
  `file:line` anchor and path EXISTS; DoD commands are actually runnable.
- **C File isolation** — one owner per file; no two tasks writing the same file in the same
  wave; cut→paste order (ZERO-LOSS) is correct.
- **D Waves** — the wave breakdown is correct; task dependencies are not violated; parallel
  groups are genuinely parallel (disjoint files).
- **E DoD greps** — gates catch the REAL thing. **False-green hunt:** a grep that passes even
  though the feature doesn't exist, or targets a nonexistent/rephrased string; positive
  assertions instead of "absence of something".
- **F Invariants** — the project's sacred invariants are not violated by the package's tasks.
- **G Missing edges** — missed cases, boundary conditions, plan-introduced defects (defects the
  plan itself introduces).

## Verdict rules

- **HIGH = W1 blocker** — with `file:line` + a **concrete fix** (exactly what to change in the
  contract/task file). Coding does not start until HIGH fixes are folded.
- **MED** — worth fixing before W1, not a blocker.
- Special focus — **false-green hunt** (direction E): a falsely-green gate is worse than a
  missing one.
- Flag **plan-introduced defects** separately (direction G).

## Deliverable

- **Path:** `<.claude/fitch/<slug>/self-audit-A-G.md>`.
- Format: a findings list `[direction] [HIGH/MED] file:line — problem → fix`.
- Header: summary `N HIGH + M MED`, overall verdict ready-for-W1 / blocked.

## Handoff

The CC orchestrator folds the result (self-audit fold step, PIPELINE §6): HIGH/MED findings go
into `contract.json` / task files via a **full rewrite**, BEFORE W1 starts.
