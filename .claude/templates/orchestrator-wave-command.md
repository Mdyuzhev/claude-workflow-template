# Orchestrator Wave Command Template

> hint: a self-contained template for one wave-launch prompt inside the orchestrator's own
> run-loop, and for the per-subagent prompt it hands out. Style — imperative, self-contained (a
> subagent must work from its prompt alone). Fill `<placeholder>`s, delete unused blocks/hint.

The whole cycle is autonomous (PIPELINE §7): after the self-audit A-G fold (a fresh adversarial
subagent, pre-W1, PIPELINE §6), the orchestrator drives waves per PLAN/ORCHESTRATOR without
stopping after each one. This template describes one run-loop iteration. A green wave-gate →
the orchestrator autonomously starts the next wave. HALT (report to the owner) only on
red-gate-after-self-heal / a sacred-invariant violation / a HALT fork from the brief (PIPELINE
§9.1). The owner's pre-merge acceptance is a separate gate after the last substantive wave, not
per-wave — mid-flight per-wave confirmation is NOT required. For large code epics the run-loop
lives in `ORCHESTRATOR.md`; this template is for a one-off/manual issuance of a single wave.

---

## When to use

Any wave of any epic that launches 1+ subagents: parallel waves (N disjoint-file subagents),
sequential single-task waves, sequential after-dependency waves, diag-then-fix phases. NOT for
one-off tasks outside an epic (housekeeping, a single-file hotfix) — a direct command without
ceremony is enough for those.

---

## Placeholders

| Placeholder | What to substitute | Example |
|-------------|----------------|--------|
| `<EPIC_ID>` | Epic identifier | `PRJ-045` |
| `<WAVE_LABEL>` | Wave label | `W1`, `W2`, `W3` |
| `<WAVE_NAME>` | Short wave name | `Foundation`, `Config + transport`, `Race-condition fix` |
| `<BRANCH>` | Feature branch | `feature/prj-045-w1-security` |
| `<SUBAGENT_COUNT>` | How many subagents | `3 parallel`, `1 sequential` |
| `<TASK_LIST>` | List of task files | see block below |
| `<FILE_ISOLATION>` | Notes on file isolation | "All three isolated" / "B1 <-> C2 on the same manager file - sequential" |
| `<WAVE_SPECIFIC_VERIFY>` | Extra greps from the tasks' own DoD | see examples below |
| `<NEXT_WAVE_DESC>` | What comes after this wave | `W2 (3 parallel: B1 / C1 / C2)` |
| `<SACRED_INVARIANTS>` | Wave-specific HALT invariants | a security-format rule, ZERO-LOSS cut->paste, a single-owner entry file |
| `<WAVE_SPECIFIC_RULES>` | Wave-specific prohibitions | `Option A only, no fallback allowed` |
| `<COMMITS_EXPECTED>` | Commits expected after the wave | "3: one per task" |
| `<UX_REPORT_NEEDED>` | Include a "user-facing changes" section | `true` / `false` |

---

## Template (copy as-is, replace placeholders)

```
<WAVE_LABEL> START - <WAVE_NAME> (<SUBAGENT_COUNT>)

Working dir: <PROJECT_PATH>
Branch: <BRANCH>

===============================================================
PRE-FLIGHT (do this yourself)
===============================================================

1. Verify branch and state:
   git status                              # clean
   git log --oneline -<N>                  # tip shows <COMMITS_EXPECTED_FROM_PRIOR_WAVES>
   git branch --show-current               # <BRANCH>

   If anything is off - STOP, report.

<OPTIONAL - wave-specific pre-flight checks, e.g.:>
2. Verify <dependency> (this wave's task requires <X>):
   <grep / build command>
   # expected: <result>

   If it doesn't match - STOP, report.

===============================================================
<WAVE_LABEL> - <SUBAGENT_COUNT>
===============================================================

Launch <one / N> subagent(s), each with an EXPLICIT model parameter (RULES R6).
<If parallel:> A separate prompt for each, do NOT let them see each other's task files.
<If sequential:> Launch one at a time, wait for the commit before the next one.

<TASK_LIST>
  For example:
  Subagent #1 - <short task name>:
    task: .claude/tasks/<epic-slug>/task-01-<slug>.md
  Subagent #2 - <short task name>:
    task: .claude/tasks/<epic-slug>/task-02-<slug>.md

Prompt for EACH subagent (substitute your task path):

  --------------------------------------------------------------
  You are a subagent for <WAVE_LABEL> of epic <EPIC_ID>. Execute exactly one task file.

  Working dir: <PROJECT_PATH>
  Working branch: <BRANCH> (already checked out)
  Task file: <PATH TO TASK FILE>

  Execution rules:
  1. Read the task file IN FULL before any editing.
  2. Before every edit - locate the anchor text, make sure the match is unique. If line
     numbers have shifted (prior waves merged) - grep a unique string from the anchor and
     adapt.
  3. After every significant step - run the verify command from the task. If it fails - STOP,
     do not move on, report the command output.
  4. The task's DoD block is checked BEFORE commit. EVERY DoD item must pass. If any item
     fails - do NOT commit, report.
  5. Commit with a message strictly per the template from the task.
  6. After the commit - report in the format:
       commit: <hash>
       DoD: <count_passed>/<count_total>
       blockers: <none | description>
       <IF UX_REPORT_NEEDED=true:>
       user-facing changes: <what the end user will see>

  FORBIDDEN:
  - Editing or creating files outside the task's scope.
  - "While I'm at it" fixes - scope discipline; anything extra goes to the backlog.
  - Calling the base toolchain directly (e.g. a raw compiler/build binary) - only via the
    project's own wrapper scripts.
  - Running the full test suite without a prior separate build step (two explicit steps, not
    one); running a full smoke run when the task is about one spec (that's the wave gate's job).
  - Re-running the same green test "for reassurance" - it's deterministic, one run is enough.
  - git push without the orchestrator's explicit go-ahead.

  <WAVE_LABEL> SPECIFICS:
  <WAVE_SPECIFIC_RULES>
  --------------------------------------------------------------

===============================================================
FILE ISOLATION for <WAVE_LABEL>
===============================================================

<FILE_ISOLATION>
  For example:
  - All tasks ISOLATED: no file overlap.
  - <file A> - task 1 touches the zone near <anchor 1>, task 2 touches <anchor 2>. Different
    functions.
  - <config file> - each task adds one dependency entry. Different lines, low conflict.
  - <entry file> - all tasks each add one registration line. Sequential merge.

If a subagent reports a merge conflict - STOP, report, do not auto-merge.

===============================================================
<WAVE_LABEL> WAVE GATE (after <all / the subagent> report(s))
===============================================================

1. Verify commits:
   git log --oneline <BRANCH> ^main
   # <COMMITS_EXPECTED>

2. Build verify (build is a separate step from smoke - PIPELINE/WORKFLOW canon):
   <project build wrapper command> 2>&1 | tee <log path>
   echo "build exit: $?"
   # exit 0

   test -f <build artifact path>
   # exists, fresh mtime, sane size

   If the build is red - STOP, investigate via the log, do not run smoke.

3. Smoke verify (only after a green build, as a separate invocation):
   <project test wrapper command> 2>&1 | tee <log path>
   # see expected numbers in the PLAN.md wave-gate section

   One run. A second one ONLY on suspicion of a flake.
   The smoke log should carry no build noise - that's a sign the build step was skipped.

<WAVE_SPECIFIC_VERIFY>
  For example:
  4. Bundle-leak verify (task-specific):
     ls <dist assets> | xargs grep -l "<internal-only symbol>" | wc -l   # == 0
  5. Migration verify (task-specific):
     grep -c "<migration function name>" <target file>                  # >= 2

N. Summary report in the format:

   --------------------------------------
   <WAVE_LABEL> CLOSED
   --------------------------------------
   <Task1>: commit <hash>, DoD <N/N>
     <wave-specific bullets if any>
   <Task2>: commit <hash>, DoD <N/N>
   ...

   build: green (warnings: <count>)
   tests: green
   <wave-specific verify results>

   File conflicts during merge: <none | description>

   <IF UX_REPORT_NEEDED=true:>
   User-facing changes (for CHANGELOG):
   - <bullets - what the user will notice after the update>

   next: <NEXT_WAVE_DESC> (autonomous)
   --------------------------------------

N+1. Green gate -> autonomously start <NEXT_WAVE_DESC> (NO mid-flight stop).
     Red gate -> self-heal (<=2 attempts) -> still red -> HALT + report to the owner.
     The status block above is written for traceability, it does not block the transition.
     The only external STOP is the pre-merge gate (after the last substantive wave).

===============================================================
ESCALATION RULES
===============================================================

STOP and report immediately:
- DoD failed for any subagent.
- The build or test-suite invocation failed outright.
- A merge conflict that does not resolve by linear addition.
- A subagent went outside scope (touched a file outside the task).
- An anchor-based edit failed several times in a row (line drift).
- A sacred invariant of the wave was violated: <SACRED_INVARIANTS>.
- <WAVE_SPECIFIC_ESCALATIONS>
  For example: a subagent attempted the forbidden fallback option instead of the locked-in one.

Do NOT do any of the following, unless this IS the close wave under owner GO (Variation C):
push to main/origin; bump the version; build the installer; run the final full-suite smoke
gate; `merge --no-ff`.

(The transition to the next substantive wave on a green gate is, on the contrary, autonomous —
do NOT wait for a go-ahead.)
```

---

## Variation A — single sequential subagent

Applied to a single task that touches a file from a previous wave.

Specifics:
- In the launch block — "Launch one subagent. Sequential because it touches `<file>` from the
  previous wave."
- FILE ISOLATION simplified — "only `<file>`, sequential after `<previous task>`."
- WAVE GATE simpler — verify 1 commit + one report.

## Variation B — diag-then-fix pair

Applied to bugs harder than "one file, one line" (WORKFLOW.md §17). Diag and fix are two
separate wave-launches, with the orchestrator's own synthesis in between — no chat round with
the owner unless the synthesis hits the HALT menu (PIPELINE §9.1).

```
<EPIC_ID> <WAVE_LABEL> - diag-then-fix for <bug name>. Two phases.

Phase Diag (single subagent):
- task-NNa-diag-<name>.md -> add [DIAG <EPIC_ID>] instrumentation to <files>, run the narrow
  test, write a report to .claude/fitch/<slug>/diag/<name>.md structured as
  Symptom / Collection / Analysis / Recommendations.
- Do NOT apply a fix in this phase. Instrumentation + report only.

[The orchestrator synthesizes the fix from the diag report itself - no chat round unless a
HALT fork applies.]

Phase Fix (single subagent, right after synthesis):
- task-NNb-fix-<name>.md -> remove [DIAG] markers, apply the targeted fix per the synthesis.
- DoD: grep -c "[DIAG <EPIC_ID>]" file == 0, plus the passing test.

Wave gate (after Phase Fix): full smoke run, failure diff empty.
```

## Variation C — close wave (finalize WITHOUT merge)

The epic's last substantive wave is close mechanics: bump version, build the installer, run
the full smoke suite, update CHANGELOG/ROADMAP, archive task files, fold a lesson into the
reference doc. **Merging into main is NOT part of this wave.** Three-step model (PIPELINE
§8/§9.2/§9.3/§10):

1. **The close wave runs autonomously and WITHOUT merge** — lockstep version bump across every
   version source, build the installer + checksum, docs/CHANGELOG, archive fitch/tasks, a
   lesson into the reference doc, a clean `git status`.
2. **STOP with a report** (PIPELINE §8) — a close report in the epic's directory, commits
   remain on the branch.
3. **The owner's pre-merge acceptance** (PIPELINE §9.2, manual smoke) → **GO = FINAL
   immediately, performed by the orchestrator itself** (PIPELINE §9.3 → §10): `merge --no-ff`
   + an annotated tag on the merge commit + push + registries + disk cleanup.

Specifics:
- Can be a single subagent for everything, or the tech lead directly via the filesystem.
- Wave gate — a full smoke run + any manual security checks from `contract.json`.
- The wave executor **never** performs `merge`/`tag`/`push` — that's step 3, FINAL after the
  owner's GO at acceptance.

**Pre-merge grep for hardcoded old versions/epic ids** — after the version bump, before the
installer, grep the source for the previous version string and epic id:

```bash
grep -rn "<old-version>" <source dirs> | grep -v node_modules | grep -v target
grep -rn "<previous-epic-id>" <source dirs> | grep -v node_modules | grep -v target
# Expected - empty (or only known-acceptable, e.g. CHANGELOG history references).
# If there's a match - fix it, or explicitly record it as a known issue.
```

_Lesson: a version-bump script can miss a hand-written literal (e.g. a footer string) — caught
only visually, on a screenshot, during manual E2E. One grep pass before the installer closes
this class of regression for good._

## Variation D — multi-phase wave with an intentional broken intermediate state

_Lesson: a phase-A that declares modules phase-B hasn't created yet leaves a red build in
between. Without an explicit note, agents panic and "fix" it mid-wave._

```
<EPIC_ID> <WAVE_LABEL> - <name>. <M> tasks, 2 phases, intentional broken state between them.

Phase A (parallel <K> subagents, or sequential single subagent):
- task-NN-name.md - owner of file X (adds declarations for all modules in the wave)
- task-NN+M-name.md - (if parallel) isolated work in a different file scope

IMPORTANT: after Phase A the build is intentionally red - file X declares modules that Phase B
will create. Do NOT run the wave gate, do not escalate, do not "fix" it - this is OK.

Phase B (parallel after Phase A, <L> subagents):
- task-NN+1-name.md - creates a module without editing file X
- task-NN+2-name.md - creates a module without editing file X

File isolation: file X - only in Phase A. Phase B tasks carry an explicit "do NOT edit X"
prohibition.

Wave gate (ONLY after Phase B):
- build green (do not check between phases).
- full test run.
- File X commits: exactly 1 commit from this wave (the Phase A owner). If 2+, ownership was
  violated - escalate.
```

---

## Checklist before sending the command

- [ ] All wave task files read in full, scope confirmed.
- [ ] File isolation actually verified by grep, not from memory.
- [ ] PRE-FLIGHT reflects the current state (correct commit count, correct branch).
- [ ] WAVE_SPECIFIC_VERIFY contains the critical greps from each task's own DoD.
- [ ] Wave-specific rules/compromises listed explicitly, not left to "the subagent will figure
      it out."
- [ ] SACRED_INVARIANTS spelled out (violation = HALT + report, not auto-fix).
- [ ] UX_REPORT_NEEDED decided - true whenever there are user-facing changes.

---

## Patch vs rewrite (adjusting task files before launch)

_Lesson: a mid-epic rename was patched into the orchestrator prompt by hand for one wave — it
worked once, but was fragile: without the same driver on the next wave, the adjustment would
have been lost and the agent would have followed the stale task file._

When a discrepancy between a task file and the actual code surfaces between waves (a rename,
an API shape change, a new field) — choose patch vs rewrite:

**Patch via the orchestrator prompt** — OK when it's a one-off adjustment (1-2 identifiers), an
indirect change, 1-2 tasks affected, and explaining it takes a short paragraph.

**Rewrite the task file** — mandatory when the API contract changed structurally, a file/module
moved, 3+ tasks are affected, the DoD commands no longer work as written, or the "quick note"
would run past ~50 lines. Principle: a patch that long IS a task file — write it as one, not as
a prompt.

**Anti-pattern:** a patch covering more than 1-2 identifiers is fragile — it lives only in chat
history. **Single source of truth = the task file, not the prompt.**

---

## See also

WORKFLOW.md §10 (smoke) · §11.1-11.2 (E2E hygiene, build vs smoke) · §15 (DoD) · §17 (red
test/regression) · §19 (this template) · docs/03-task-files-anatomy.md ·
docs/05-diag-then-fix.md · `.claude/PIPELINE.md` · `templates/orchestrator-template.md`.
