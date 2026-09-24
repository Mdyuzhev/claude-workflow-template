> hint: one committed file per delegated unit, written BEFORE the task
> call. Fill every `<placeholder>`; delete this hint line after the first
> fill. Size cap: body under 3 000 characters, write zone under 6 files or
> one module directory, budget under 20 minutes, one gate command.

# `<link>`-`<NN>` — `<short title>`

**Dispatch prompt (fixed string — do not reinvent):**
> Read and execute the card `<path>`. Report per its "Return" section only.

## Worktree
`<path>`. Branch `<branch>`.

## Base sha
`<sha>` — do not reset or rebase.

## Write zone
- `<path 1>`
- `<path 2>`

## Do not touch
`<explicit list: files/dirs/actions forbidden even in passing — no version
bump, no new dependency, no mutating MCP call, no git merge/tag/push, no
destructive shell here>`

## Steps
1. `<step 1>`
2. `<step 2>`
3. Capture (timeout `<n>`s): `<capture command — redirect only, no
   PowerShell content cmdlets, kit/RULES.md §14(в) and
   docs/10-gates-and-evidence.md §4>` → `<transcript path>`.
   Last line must be `EXIT=0`.
4. One commit `<commit-message-prefix>:`, English. Listed files only.

## Gate
`<one command>`
Expected: exit `<n>`.

## Budget
`<N>` min. Every command inside carries its own explicit timeout.

## Return
≤ 40 lines: files touched; gate exit; transcript path(s); deviations from
the steps above. First line `PARTIAL:` if the budget ran out with work
still on disk; first line `DIAG:` if stopped by a stop rule.

## Stop rules
Red after 2 self-heals → `DIAG:`, stop. `<sacred file/state changed>` →
`DIAG: sacred`, do not commit. N identical failures of one class → `DIAG:`.
Do not start the next card.
