# PLAN-`<epic-slug>`.md (template)

> hint: lives at `.claude/tasks/<epic-id>/` next to the task files. The orchestrator reads this
> file in full at start and autonomously drives waves through it (see "Run-loop" below). For
> code epics there is a parallel `ORCHESTRATOR.md` (`templates/orchestrator-template.md`) that
> drives the same run-loop from `.claude/fitch/<slug>/`. Delete this hint after the first fill.

---

# PLAN `<PRJ-NNN>` — `<Epic name>`

**Epic:** `<PRJ-NNN>`
**Branch:** `feature/<epic-slug>`
**Working dir for all tasks:** `<PROJECT_PATH>`
**Installer slot:** `<slot prefix> <version>`
**Duration:** ~`<X>` substantive waves + W`<X+1>` finalization
**Audit applied:** `<.claude/fitch/<slug>/self-audit-A-G.md>` (if there was an audit-fix before
W1, list the findings that were applied)

## Pre-flight

1. Self-audit A-G by a fresh adversarial subagent (pre-W1, PIPELINE §6) passed, findings folded
   into the contract/task files. The orchestrator chooses the wave composition itself, per the
   scope rule from the brief — owner confirmation of composition is NOT required. After the
   self-audit fold, coding proceeds autonomously (PIPELINE §7).
2. `.claude/fitch/<slug>/contract.json` — contract with `<N>` active requirements +
   `<M>` verification requirements.
3. `git status` clean, `git branch --no-merged main` empty except self.
4. Create the branch: `git checkout -b feature/<epic-slug> main`.

## Wave-by-wave execution (orchestrator → subagents)

### W1 — `<Wave name>`

**Goal:** `<one-line goal>`.
**Tasks:** `task-01..task-<NN>` (`<count>` files).
**Parallel groups within the wave:**
- Group 1 (sequential): `<task-XX>` (`<one-line>`) — must go first because `<reason>`.
- Group 2 (parallel): `<task-YY>`, `<task-ZZ>` (different files).
- Group 3 (sequential on the same file): `<task-AA>` → `<task-BB>` (one subagent).

**Wave gate:** `<test command>` green, baseline ≥ `<N>`. Green gate → orchestrator
autonomously starts W2 (no mid-flight confirmation). Red gate → self-heal (≤2 attempts) →
still red → HALT + report.

### W2 — `<Wave name>`

**Goal:** `<goal>`. **Tasks:** `<task-NN>..<task-MM>`. **Sequential / parallel:** ...
**Wave gate:** ...

(... repeat for each wave ...)

### W`<final>` — Finalization

**Goal:** bump version + installer + close mechanics WITHOUT merge (PIPELINE §8).
**Tasks:** `task-<final>-finalization.md`.

See `templates/orchestrator-wave-command.md` Variation C for the close-wave command template.

## File isolation audit

Before the epic starts, the tech lead verifies via physical grep that in each wave, at most
one subagent edits any given file.

**Files touched by multiple tasks across the epic:**
- `<file A>` — task-NN (W2), task-MM (W4). **Sequential** — no problem.
- `<file B>` — task-AA (W3), task-BB (W3). **CONFLICT** — resolution: sequential within one
  wave (one subagent), or move one of the tasks to W4.

**Files with multiple exposes (e.g. test-hook blocks under a build-time flag):**
- `<app entry file>` — centralize ALL exposes in one task in W1 as a single block. All later
  waves then **only use** the exposes, without touching the entry file. One task, one file,
  no race.

## Run-loop (autonomous coding cycle, PIPELINE §7)

The whole cycle is autonomous: after the self-audit A-G fold (PIPELINE §6), the orchestrator
drives waves without stopping after each one. Gating happens only on HALT conditions and at
pre-merge acceptance, not per-wave-confirm. For each wave:

```
launch the wave's subagent(s) -> subagent executes + writes diag/wN-report.md + STOP
  -> orchestrator checks wave_gate
    -> green: autonomously next wave (NO mid-flight stop)
    -> red: self-heal (<=2 attempts) -> still red: HALT + report to the owner
```

After each wave, the orchestrator records a short status (for traceability, not for blocking):

```
W<N> closed
- closed: <task-list>
- gate: <smoke metrics>, failure IDs diff = <empty | listed>
- commits: <hash-list> + comment
- next: W<N+1> per PLAN (autonomous)
```

The cycle has a single external stop (PIPELINE §9) — **the owner's pre-merge acceptance** (a
semantic spot-check after the last substantive wave); HALT forks from the brief (below) are
conditional stops. Close mechanics run WITHOUT merge — they record readiness and stop at
acceptance; **the owner's GO = the orchestrator immediately performs FINAL itself** (merge
--no-ff + annotated tag + push + registries + disk cleanup, PIPELINE §10), with no separate
command. Mid-flight HALT happens only on red-gate-after-self-heal / sacred-invariant violation
/ unexpected audit diff (see HALT conditions below). Diag-then-fix micro-wave escalation is
done by the orchestrator autonomously when the failure diff is non-empty.

## HALT conditions (report to the owner, do not continue autonomously)

The orchestrator stops and reports to the owner only when:

- a red gate persists after 2 self-heal attempts;
- a sacred-invariant violation (`<list for the epic>` — e.g. ZERO-LOSS: content cut without an
  archive landing; a live invariant deleted instead of moved; a security invariant broken);
- an unexpected audit baseline diff (`<a docs epic must not touch code / a code epic must not
  shift the baseline beyond the expected new tests>`);
- a merge conflict that does not resolve by linear addition;
- a subagent went outside the task's scope (touched a file outside its isolation);
- a dirty git status after the final wave.

In all other cases (green gate, expected metric growth from new tests) the orchestrator
continues autonomously.

## Expected metrics

| Wave | Baseline before | Expected after | Notes |
|---|---|---|---|
| W1 | `<X passing / Y failing>` | `<X passing / Y failing>` (baseline-identical, or +N from new tests) | `<context>` |
| W2 | post-W1 | `<expected>` | ... |
| ... | | | |

If post-WN differs from expected — a diag-then-fix micro-wave, or escalation.

## Epic exit (Definition of Done)

- [ ] All W1..W`<final-1>` closed, gates green.
- [ ] Finalization W`<final>` closed: installer exists in `distr/`, CHANGELOG updated, close
      mechanics done (WITHOUT merge).
- [ ] FINAL (merge/tag/push — performed by the orchestrator itself on the owner's GO at
      acceptance) done; the audit baseline diff vs pre-epic is empty or known-acceptable.
- [ ] Task files moved to `.claude/tasks/done/<epic-id>/`.
- [ ] ROADMAP.md updated: epic added to the closed-epics section, changelog entry, metrics.
- [ ] Long-term agent memory updated, if the project uses one and the epic produced a new
      lesson or changed a baseline metric.

## Skeptic counter-arguments accepted during planning

> If there is audit / skeptic feedback on what NOT to do in this epic — record it here
> explicitly. This guards against scope creep in the final waves.

- Do NOT decompose `<File X>` further — `<reason>`.
- Do NOT unify `<Y>` — `<reason>`.
- Do NOT migrate `<Z>` — `<reason>`.
- Backlog: `<list>` — deferred to `<PRJ-NNN.x pickup>`.
