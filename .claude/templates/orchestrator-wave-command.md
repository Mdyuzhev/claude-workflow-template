# Orchestrator Wave Command Template

Самодостаточный шаблон команды которую Techlead отправляет оркестратору в Claude Code когда стартует новую волну эпика. Все placeholder'ы заполняются под конкретную волну, оркестратор не должен идти искать контекст в других файлах.

Структура: 5 блоков (PRE-FLIGHT / WAVE START / FILE ISOLATION / WAVE GATE / ESCALATION) + variation (parallel / sequential / diag-then-fix / финализация).

---

## Шаблон (parallel volna, N subagent'ов)

```
[<EPIC-ID> W<N>] start. Branch <branch>, working dir <PROJECT_PATH>.

<EPIC-ID> W<N> — <human-readable wave name> (<R-ids covered>).

<M> tasks, <K> фаз:

Phase A (<single subagent | parallel N subagent'ов | sequential>):
- <task-NN-name>.md → <one-line action>
- <task-NN+1-name>.md → <one-line action>
- File isolation: <which files this phase touches>

Phase B (<...> после confirm Phase A):
- <task-NN-name>.md → <one-line>
- File isolation: <files>

Phase C (<gate phase>, single subagent после confirm Phase B):
- <task-NN-name>.md → <gate task or final integration>

File isolation summary: <File X> только в Phase A. <File Y> только в Phase B. Spec/test files в Phase C — изолированы.

Confirmation между фазами обязательна. Между task'ами внутри одной фазы (если sequential на одном файле) — confirmation НЕ нужна, один subagent делает оба.

Wave gate (после Phase C): npm run e2e:test без --spec, full smoke. Failure IDs vs post-W<N-1> baseline (<X passed / Y failed>) — пустой diff. Spec <name> passing <count>.

Старт — Phase A. Доложить closed/gate/next по шаблону §19 WORKFLOW.
```

---

## Variation 1 — Single subagent + sequential (один subagent, несколько task'ов на одном файле)

```
[<EPIC-ID> W<N>] start. Branch <branch>, working dir <PROJECT_PATH>.

<EPIC-ID> W<N> — <name>. <M> tasks, single subagent, sequential на одном файле <file>:

Sequential (один subagent делает все три подряд, между task'ами без confirmation):
1. <task-NN-name>.md
2. <task-NN+1-name>.md
3. <task-NN+2-name>.md

File isolation: <file> — только этот subagent. Никаких параллельных trog'ов.

Wave gate: <DoD команда + ожидание>. Confirmation после всех трёх.

Старт. Доложить closed/gate/next по шаблону §19.
```

---

## Variation 2 — Diag-then-fix

```
[<EPIC-ID> W<N>] start. Branch <branch>, working dir <PROJECT_PATH>.

<EPIC-ID> W<N> — diag-then-fix для <bug name>. Две фазы.

Phase Diag (single subagent):
- task-NNa-diag-<name>.md → добавить [DIAG <EPIC-ID>] инструментацию в <files>, прогнать узкий <test command>, написать отчёт `.claude/fitch/<epic-id>/diag/<name>.md` со структурой Симптом / Сбор / Анализ / Рекомендации.
- НЕ применять fix на этой фазе. Только инструментация + отчёт.

[STOP. Confirmation от Chief'а после diag отчёта.]

Phase Fix (single subagent после Synthesis от Chief'а):
- task-NNb-fix-<name>.md → убрать [DIAG] markers, применить точечные правки по согласованным рекомендациям.
- DoD: grep -c "[DIAG <EPIC-ID>]" === 0, plus passing test.

Wave gate (после Phase Fix): npm run e2e:test full smoke, failure_diff пустой.

Старт — Phase Diag. Доложить когда отчёт готов.
```

---

## Variation 3 — Финализация (bump + installer + merge)

```
[<EPIC-ID> Финализация] start. Branch <branch> → main.

Pre-flight:
- git status чистый, git branch --no-merged main только feature.
- Все W1..W<N-1> closed, баговый diff vs pre-эпик baseline пустой.

Steps (single subagent, sequential):
1. node scripts/bump-version.js <version> — bump в VERSION + package.json + Cargo.toml.
2. CHANGELOG.md секция [<version>] — <release name>. Added/Tests/Known issues blocks.
3. **grep hardcoded по UI / source на старую версию и epic-id** (опыт PRJ-018 T14 — footer в App.tsx содержал hardcoded `v0.4.0` и `PRJ-017 architecture`, T14 их не поймал, поймали визуально на скриншоте в manual E2E):
   ```bash
   grep -rn "<old-version>" desktop/src/ src-tauri/src/ 2>&1 | grep -v node_modules | grep -v target
   grep -rn "<previous-epic-id>" desktop/src/ src-tauri/src/ 2>&1 | grep -v node_modules | grep -v target
   # Результат — пусто (или только known-acceptable, например CHANGELOG history-референсы).
   # Если match — поправить или явно зафиксировать как known issue.
   ```
4. <build wrapper script> -Prefix <slot> — собрать installer в distr/.
5. node scripts/audit.js — diff vs pre-эпик baseline. New findings = 0 или known acceptable.
6. git checkout main && git merge --no-ff <branch>.
7. git push origin main && git branch -d <branch>.
8. node scripts/audit.js --baseline.
9. mkdir -p .claude/tasks/done/<epic-id> && mv .claude/tasks/<epic-id>/* .claude/tasks/done/<epic-id>/ && rmdir .claude/tasks/<epic-id>.

Wave gate (для финализации):
- distr/<installer name>.exe + .msi exist.
- audit.js diff пустой.
- main branch up-to-date, feature branch deleted.
- **grep hardcoded старой версии / epic-id в src/ — 0 matches** (опыт PRJ-018).

Confirmation от Chief'а перед merge (step 6). Без явного OK не мержить.

Старт. Доложить когда дойдёт до step 5 (перед merge confirmation).
```

---

## Variation 4 — Multi-phase волна с intentional broken intermediate state

_Опыт PRJ-018 W2 — 4 task'а в 2 фазы. После Phase A (T04 добавил 3 pub mod в mod.rs) build красный — файлы T05/T06 ещё не созданы. Это ожидаемо, build green только после Phase B. Без явного указания subagentы/orchestrator паникуют и бросают "фиксить" билд._

```
[<EPIC-ID> W<N>] start. Branch <branch>, working dir <PROJECT_PATH>.

<EPIC-ID> W<N> — <name>. <M> tasks, 2 фазы, intentional broken state между ними.

Phase A (parallel <K> subagent'ов, или sequential single subagent):
- task-NN-name.md — owner файла X (добавляет declarations для всех модулей волны)
- task-NN+M-name.md — (если parallel) изолированная работа в другом file scope

**ВАЖНО:** после Phase A build намеренно красный — в файле X есть declarations на модули которые будут созданы в Phase B. НЕ запускать wave gate, не эскалировать, не «фиксить» — это ok.

Phase B (parallel после Phase A confirmation, <L> subagent'ов):
- task-NN+1-name.md — создаёт модуль без редактирования файла X (owns task-NN)
- task-NN+2-name.md — создаёт модуль без редактирования файла X (owns task-NN)

File isolation: файл X — только в Phase A (task-NN). Phase B task'и имеют явный запрет "НЕ редактировать X" в своих "ЧТО НЕ ДЕЛАТЬ".

Wave gate (ТОЛЬКО после Phase B):
- build green (не проверять между фазами).
- <test command> full run.
- File X commits: git log --oneline -5 -- <file X> должен показать ровно 1 коммит из этой волны (Phase A owner). Если 2+ — ownership нарушен, эскалировать.

Старт — Phase A. Доложить когда Phase A closed и готов к Phase B. НЕ проверять build между фазами — он намеренно красный.
```

---

## 5 блоков обязательной структуры

Каждая wave command — даже custom variation — содержит эти пять элементов:
### 1. PRE-FLIGHT
- `<EPIC-ID> W<N>` идентификатор.
- Branch + working dir.
- Если есть зависимости (closed previous wave, merged dep PR) — явно.

### 2. WAVE START
- Имя волны + R-ids которые покрывает.
- Список task'ов с phase grouping (parallel / sequential / single).
- Краткий one-line action per task.

### 3. FILE ISOLATION
- Какие файлы трогает каждая фаза.
- Если есть пересечение — явно sequential, не parallel.
- Если есть файл который трогается в нескольких task'ах одной фазы — это нарушение, эскалировать.

### 4. WAVE GATE
- Конкретная команда для gate проверки.
- Baseline для failure diff (числа метрик предыдущей волны).
- Specific spec / test name которая должна passing с количеством.

### 5. ESCALATION
- Что делать если gate не зелёный (diag-then-fix микро-волна, эскалация Techlead'у).
- Что делать если task не находит anchor (НЕ free-form правки, эскалация).
- Что делать если build fail (cargo / npm error в логе) — эскалировать с full error message.

---

## Checklist перед отправкой команды

- [ ] Все task-файлы physically existуют в `.claude/tasks/<epic-id>/`.
- [ ] Read реального кода сделан перед task-файлами.
- [ ] File isolation проверен grep'ом (`grep -l <file>` по task-файлам — не больше одного match per файл per волна).
- [ ] DoD каждого task'а — concrete команда, не «тесты проходят».
- [ ] Baseline числа для failure diff — взяты из логов предыдущей волны, не из памяти.
- [ ] Spec name verified (реальный файл в `desktop/e2e/specs/` или эквивалент, не phantom).

---

## Корректировки к task-файлам перед запуском (patch vs rewrite)

_Опыт PRJ-018 W4 — в W3 subagent адаптировал struct rename (`RunWithReport` → `RunStatsRow`) из-за коллизии имени. Task-файлы W4 (T10/T11/T12) были написаны раньше и ссылались на старое имя. Techlead в чате добавил «КРИТИЧНО: TS interface называть RunStatsRow» в orchestrator-промт перед запуском. Работало, но fragile — без Techlead'а в чате agent сломал бы._

Когда между волнами выявляется расхождение между task-файлом и реальным кодом (struct переименовался, API вернул другую shape, поле добавилось) — Techlead выбирает patch vs rewrite:

### Patch через orchestrator-промт (OK в этих случаях)

- Одноразовая корректировка (переименование 1-2 идентификаторов).
- Изменение косвенное (имя Tauri command осталось, поля результата другие).
- 1-2 task'а в волне затронуты.
- Исправление task'а файла равносильно объяснить в промте «N словами».

```markdown
# Promp оркестратору

## КОРРЕКТИРОВКИ К TASK-FILES W4

В W3 subagent переименовал структуру RunWithReport → RunStatsRow из-за коллизии имени. Все task-файлы W4 (T10/T11/T12) должны это учесть:

1. TypeScript interface = `RunStatsRow` (НЕ `RunWithReport`)
2. Поля TS interface совпадают с Rust struct в db/models.rs::RunStatsRow
3. Везде в task-файлах где упомянуто «RunWithReport» — читать как «RunStatsRow».
```

### Rewrite task-файлов (обязательно в этих случаях)

- Изменился API контракт (signature, return type структурно другой).
- Изменился file path / module structure (файл переехал, модуль переименовался).
- 3+ task'а в волне затронуты.
- DoD команды больше не работают как написаны.
- Исправление в промте займёт больше чем обновить task-файл.

Принцип: если «важные корректировки» в промте занимают >50 строк — вы пишете новый task-файл. Пишите его как task, не как промт.

### Анти-паттерн

Patch через промт больше чем на 1-2 идентификатора — fragile. Если Techlead'а нет в чате (другой разработчик запускает эпик или sеssion crash) — корректировки из промта теряются, agent идёт по устаревшему task-файлу. **Single source of truth = task-файл, не промт.**
