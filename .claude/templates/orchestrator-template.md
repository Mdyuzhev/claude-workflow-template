# ORCHESTRATOR — `<epic-id / branch>` (template)

> hint: `ORCHESTRATOR.md` lives in `.claude/fitch/<slug>/` next to `contract.json` +
> `FITCH-PLAN.md`. It is the driver of the autonomous run-loop for code epics: the orchestrator
> hands out waves to subagents, holds the wave gates, self-heals, and runs AUTONOMOUSLY
> W0→`<final>`. For docs/chore epics the run procedure can live directly in FITCH-PLAN — a
> separate ORCHESTRATOR.md is then optional. Fill `<placeholder>`s, delete blocks that don't
> apply, delete this hint.

---

# ORCHESTRATOR — `<chore/feature>/<epic-slug>`

**For:** the orchestrator in Claude Code. Hands out waves to subagents, holds the gates,
self-heals, runs AUTONOMOUSLY W0→`<final>`. `<For a docs epic: the tech lead in chat does NOT
touch files itself — all edits are made by subagents from task files.>`
**Epic:** `<one-line scope>`. **Base:** main `<hash>` (loose-ref verified). `<No
version-bump/installer/tag | bump <X.Y.Z>→<X.Y.Z+1>>`.
**Contract:** `.claude/fitch/<slug>/contract.json` (`<r2>`). **Plan:** `FITCH-PLAN.md`. Wave
gates = `contract.tech_lead.waves[].wave_gate`.

## Run-loop (autonomous)

The whole cycle is autonomous: after the self-audit A-G fold (a fresh adversarial subagent,
pre-W1, PIPELINE §6), the orchestrator drives waves without stopping. **Subagents are STRICTLY
an explicitly given model ([RULES.md](../RULES.md) R6): every spawn carries an EXPLICIT model
designation; a bare/default-model spawn is forbidden; every wave-status/diag-report must state
the model and count used, otherwise it's a red gate.** For each wave:

```
launch subagent(s) with the specified task file
  -> subagent executes + writes diag/wN-report.md + STOP
  -> orchestrator checks wave_gate
    -> green: autonomously next wave (NO mid-flight stop)
    -> red: self-heal (<=2 attempts) -> still red: HALT + report to the owner
```

There are NO mid-flight stops between waves. There is a single external stop (PIPELINE §9) —
**the owner's pre-merge acceptance** (after W`<final>`); HALT forks from the brief (below) are
conditional stops. The owner's GO at acceptance = the orchestrator immediately performs
**FINAL** itself (merge/tag/push + registries + cleanup, PIPELINE §10).

## Waves → task files → subagents

| Wave | Task file(s) | Subagent(s) | Isolation |
|-------|--------------|-------------|----------|
| **W0** `<Cruft sweep / Foundation>` | `task-00-<slug>.md` | 1 `<git/fs>` | `<creates branch from main, sweep, commit W0>` |
| **W1** `<wave name>` | `task-NN-<slug>.md` | `<1 / N parallel>` | `<disjoint files / single-owner>` |
| **W2** `<wave name>` | `task-NN-<slug>.md` | `<N>` | `<PASTE-FIRST: archive landing BEFORE the cut, if ZERO-LOSS>` |
| ... | ... | ... | ... |
| **W`<final>`** Finalize | `task-99-finalize.md` | 1 | `<DoD greps + commit(s) + audit baseline + push; bump/installer if a code epic>` |

Dependencies: `<W1 after W0 (branch) ; W2 after W1 (X exists for references) ; W<final> after
all>`.

File isolation: `<1 owner/file per wave ; list allowed conflicts - sequential / PASTE-FIRST>`.

## Gates (from contract, brief)

- W0: `<ls/grep/git state check after sweep>`.
- W1: `<grep -cE '...' == N ; test -f X>`.
- W2: `<paste-first landing ; target lines rewritten ; a positive assertion of the current
  canon is present>`.
- ... `<one line per wave, copied from contract.tech_lead.waves[].wave_gate>`.
- W`<final>`: `<audit baseline +-0 ; git status clean ; HEAD loose-ref ; (tag/bump if a code
  epic)>`.

## HALT conditions (report to the owner, do not continue autonomously)

- a red gate persists after 2 self-heal attempts;
- a sacred invariant violated: `<ZERO-LOSS - cut without an archive landing ; a live invariant
  deleted instead of moved ; a security format broken ; a single-owner entry file touched by
  two subagents>`;
- an unexpected audit diff != expected (`<a docs epic does not touch code / a code epic does
  not shift the baseline beyond new tests>`);
- a merge conflict that does not resolve by linear addition;
- a subagent went outside scope (touched a file outside its isolation);
- a subagent was spawned without an explicit model parameter (RULES R6);
- git status dirty after the final wave.

## EXPANSION (round 2, optional) — W`<k>`→W`<final>`

If the epic expands with a second round: contract `contract-expansion.json`, the same
run-loop/HALT rules, continuation of the same branch (HEAD `<hash>` after round 1).

| Wave | Task file | Subagent | Isolation |
|-------|-----------|----------|----------|
| **W`<k>`** `<name>` | `task-NN-<slug>.md` | 1 | `<disjoint>` |
| ... | ... | ... | ... |

`<W<k>..W<final-1> disjoint files -> up to N subagents in parallel are OK ; W<final> after
all.>` Gates = `contract-expansion.qa_matrix[]`. HALT conditions the same +
`<additional ZERO-LOSS markers for round 2>`.

## After W`<final>` (pre-merge acceptance → close mechanics → FINAL)

STOP → **the owner's pre-merge acceptance** for the WHOLE epic (`<semantic spot-check of key
artifacts - cuts / invariants / refresh quality>`). On OK → **close mechanics WITHOUT merge**
(orchestrator, autonomously): archive fitch/tasks → done/ + a lesson entry + findings registry
+ prune process tmp + clean status → report → STOP. `merge --no-ff main` + push +
`<NO tag (docs epic) | tag <version> + installer (code epic)>` — this is **FINAL**
(PIPELINE §10): the orchestrator performs it ITSELF immediately on the owner's GO at
acceptance; before GO it is not performed.

---

## See also

- _see canonical: ../RULES.md#r8-границы-доверия-инструкции-данные-секреты_ — instructions
  vs data; binds every subagent this template dispatches.
- `templates/PLAN.md` — epic plan (waves + file isolation + run-loop section)
- `templates/orchestrator-wave-command.md` — single-wave prompt to a subagent (variations A-D)
- `templates/fitch-contract.json` — contract (waves[].wave_gate = source of the gates above)
- `../PIPELINE.md` — canon of the working cycle (§0-§11: roles/allocation → brief → recon →
  scope → fitch → self-audit A-G → coding in waves → close mechanics without merge → single
  external stop (acceptance) → FINAL on GO → checkpoint protocol)
