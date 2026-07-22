# PLAN-`<epic-id>`.md (template)

Главный план эпика. Лежит в `.claude/tasks/<epic-id>/` рядом с task-файлами. Orchestrator читает этот файл целиком при старте, дальше идёт по wave-командам от Techlead'а с reference на этот план.

---

# PLAN `<EPIC-ID>` — `<Epic name>`

**Эпик:** `<EPIC-ID>`
**Branch:** `feature/<epic-id>-<short-name>`
**Working dir для всех task'ов:** `<PROJECT_PATH>`
**Installer slot:** `<slot prefix> <version>`
**Длительность:** ~`<X>` содержательных волн + W`<X+1>` финализация
**Audit applied:** `<.audits/<date>-<epic-id>-fitch.md>` (если был audit-fix перед W1, перечислить findings которые применены)

## Pre-flight

1. Главный чат (Techlead, Chief) подтвердил состав, фильтр findings, audit-патчи и структуру.
2. `.claude/fitch/<epic-id>/contract.json` — контракт с `<N>` active R + `<M>` verification = `<N+M>` R.
3. `git status` чистый, `git branch --no-merged main` пустой кроме self.
4. Создать ветку: `git checkout -b feature/<epic-id>-<name> main`.

## Wave-by-wave execution (orchestrator → subagents)

### W1 — `<Wave name>`

**Цель:** `<one-line goal>`.
**Tasks:** `task-01..task-<NN>` (`<count>` файлов).
**Параллельные группы внутри волны:**
- Группа 1 (sequential): `<task-XX>` (`<one-line>`) — должен пройти первым потому что `<reason>`.
- Группа 2 (parallel): `<task-YY>`, `<task-ZZ>` (разные файлы).
- Группа 3 (sequential на одном файле): `<task-AA>` → `<task-BB>` (один subagent).

**Wave gate:** `<test command>` зелёный, baseline ≥ `<N>`. Confirmation от Chief перед W2.

### W2 — `<Wave name>`

**Цель:** `<goal>`.
**Tasks:** `<task-NN>..<task-MM>`.
**Sequential / parallel:** ...

**Wave gate:** ...

(... повтор для каждой волны ...)

### W`<final>` — Финализация

**Цель:** bump version + installer + merge → main + audit baseline.
**Tasks:** `task-<final>-finalization.md`.

См. `.claude/templates/orchestrator-wave-command.md` Variation 3 для шаблона команды финализации.

## File isolation audit

Перед стартом эпика — Techlead phys grep'ом верифицирует что в каждой волне один файл правит максимум один subagent.

**Файлы которые трогаются множеством task'ов через эпик:**
- `<file A>` — task-NN (W2), task-MM (W4), task-LL (W7). **Sequential** — нет проблемы.
- `<file B>` — task-AA (W3), task-BB (W3). **КОНФЛИКТ** — резолв: sequential в одной волне (один subagent) или один из task'ов перенести в W4.

**Файлы с многократными exposes (E2E hooks типа `__VITE_E2E__` blocks):**
- `<App entry file>` — централизовать ВСЕ exposes в task-02 W1 единым блоком. Все W2-W`<N>` task'и тогда **только используют** exposes, не трогают `<App entry>`. Один task — один файл — нет race.

## Confirmation loop

Sonnet orchestrator после каждой волны докладывает Chief'у по шаблону §19 WORKFLOW:

```
W<N> closed
- closed: <task-list>
- gate: <smoke metrics>, failure IDs diff = <empty | listed>
- commits: <hash-list> + comment
- next: W<N+1> по PLAN

STOP. Жду explicit confirmation перед W<N+1>.
```

Chief подтверждает или назначает diag-then-fix микро-волну (если failure diff non-empty). Без OK orchestrator не стартует следующую волну.

## Метрики ожидаемые

| Wave | Baseline before | Expected after | Notes |
|---|---|---|---|
| W1 | `<X passing / Y failing>` | `<X passing / Y failing>` (baseline-identical либо +N от новых тестов) | `<context>` |
| W2 | post-W1 | `<expected>` | ... |
| ... | | | |

Если post-WN отличается от expected — diag-then-fix микро-волна или эскалация.

## Выход эпика (Definition of Done эпика)

- [ ] Все W1..W`<final-1>` closed, gate'ы зелёные.
- [ ] Финализация W`<final>` closed: installer exists в `distr/`, CHANGELOG обновлён, merge done.
- [ ] `audit.js --baseline` после merge — diff vs pre-эпик пустой либо known acceptable.
- [ ] task-файлы перемещены в `.claude/tasks/done/<epic-id>/`.
- [ ] ROADMAP.md updated: эпик в секцию «🏁 Закрытые эпики», changelog запись, метрики.
- [ ] `memory_user_edits` обновлены (если эпик дал new lesson learned или сменил baseline metrics).

## Контр-доводы скептика принятые при планировании

(Если есть audit / skeptic feedback что НЕ делать в этом эпике — записать здесь явно. Это защита от scope creep на последних волнах.)

- НЕ декомпозировать `<File X>` дальше — `<reason>`.
- НЕ унифицировать `<Y>` — `<reason>`.
- НЕ мигрировать `<Z>` — `<reason>`.
- Backlog: `<list>` — отложено в `<PRJ-NNN.x pickup>`.
