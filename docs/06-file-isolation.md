# 06 — File Isolation in Waves

Главный инвариант параллельного выполнения. Без file isolation параллельные subagent'ы устраивают git race и портят друг другу работу. С ней — экономия в 2-5x против sequential выполнения той же волны.

## Правило

**Два subagent'а не могут трогать один файл в одной волне** — даже если они правят разные функции в нём. Race на read-modify-write через git.

Что считается «трогать»:
- `str_replace` / `edit_file`
- `write_file` (создание / перезапись)
- Перемещение / переименование
- Удаление

Что НЕ считается «трогать»:
- `read_text_file` (чтение)
- `view` через computer use
- Проверка через `get_file_info`

Read parallel'но между subagent'ами безопасен. Write — нет.

## Почему именно так

Sonnet'ы (executor'ы) работают через git commits, не через shared memory. Если два subagent'а параллельно правят `App.tsx`:

1. Subagent A — `git pull main`, читает App.tsx, делает str_replace, коммитит.
2. Subagent B — параллельно `git pull main` (до коммита A), читает App.tsx, делает свой str_replace, коммитит.

Когда обе ветки попытаются мержиться — конфликт. Если конфликт автоматически резолвится — теряются изменения одного из них. Если не резолвится — orchestrator должен вручную мержить. И то и другое — нарушение isolation.

В sequential варианте (один subagent после другого) — A коммитит → B делает `git pull` → видит изменения A → правит на основе свежего state'а → коммитит. Race нет.

## Резолвы конфликтов

### Sequential в одной волне

Один subagent делает task-A, потом task-B на тот же файл. Internal sequential — другие task'и волны идут parallel'но.

Применяется когда:
- Два task'а трогают один файл
- Они логически связаны (например diag + fix на одном модуле)
- Разделять на волны overhead

### Перенос task'а в следующую волну

Task-B уходит в W+1. В W subagent A работает один над файлом. В W+1 subagent B начинает с свежего state'а.

Применяется когда:
- Task'и логически независимы
- Sequential в одной волне раздувает её время выполнения
- Между task'ами нужна confirmation от Chief'а

### Декомпозиция файла до старта эпика

Если файл «гравитирует» все task'и (App entry, manager singleton, central config) — декомпозировать его на 2-3 модуля до старта эпика, как housekeeping'ovое pre-flight.

Применяется когда:
- Файл — chronic conflict source через несколько эпиков
- Архитектурно валидна декомпозиция (single responsibility)
- Cost декомпозиции окупается за следующие 2-3 эпика

## Audit перед стартом эпика

Techlead обязан проверить file isolation **перед** финализацией tech_lead-фазы fitch'а. Способы:

### 1. Grep по task-файлам

```bash
grep -l "<file path>" .claude/tasks/<epic-id>/task-*.md
```

Если возвращает >1 task'а — конфликт. Резолвить до старта.

### 2. Build matrix

Таблица `файл × task × волна`. Если в одной волне один файл фигурирует у двух task'ов — конфликт.

```markdown
| Файл | task-01 (W1) | task-05 (W2) | task-12 (W3) | ... |
|---|---|---|---|---|
| App.tsx | X | | | |
| manager.ts | | X | X | ... |
| collections.json | | | | X |
```

X в одной колонке (волне) на двух строках (файлах) — норма. X в одной строке (файле) на двух колонках разных task'ов одной волны — конфликт.

### 3. Явная file_isolation_audit секция в contract.json

```json
"file_isolation_audit": {
  "passed": true,
  "conflicts_resolved": [
    {"file": "App.tsx", "issue": "trog в W2 task-X и W4 task-Y", "resolution": "sequential — нет race"}
  ]
}
```

Заставляет fitch фазу tech_lead явно зафиксировать что проверка проведена.

## Особый случай — App entry / shared exposes

В крупных проектах одна точка обычно становится attractor'ом для изменений: App.tsx (web), main.rs (Rust), main.tsx, AppDelegate, etc. Любой task который добавляет E2E hook / новый store provider / global handler — хочет туда зайти.

**Решение: централизовать ВСЕ exposes в один task-02 в W1.**

```typescript
// task-02 W1 — все exposes за один проход
if (import.meta.env.MODE === 'test' || (window as any).__VITE_E2E__) {
  (window as any).__APP_STORE__ = useAppStore;
  (window as any).__COLLECTION_MANAGER__ = collectionManager;
  (window as any).__COLLECTIONS_STORE__ = useCollectionsStore;
  (window as any).__GRPC_STORE__ = useGrpcStore;
  (window as any).__HISTORY_MANAGER__ = historyManager;
  (window as any).__VARIABLES_API__ = variablesApi;
  (window as any).__E2E_FS_*__ = fsApi;
  (window as any).__E2E_APPDATA_PATH__ = appdataPath;
}
```

Все W2-W11 task'и **только используют** exposes (`window.__APP_STORE__.getState()`), не трогают App.tsx. Один task — один файл — нет race.

## Ownership declarations across tasks

_Опыт PRJ-018 W2 — в одной волне 4 task'а в две фазы: T04 владеет `skills/mod.rs` (добавляет 3 pub mod), T05 и T06 пишут свои skill-файлы но НЕ редактируют mod.rs. T07 пишет markdown в другой директории. Работало потому что ownership было зафиксировано в КАЖДОМ task'е явно._

File isolation invariant (один файл = один subagent в волне) это структурное правило для Techlead'а. Но subagent'ы **изолированы** — каждый видит только свой task-файл + промт оркестратора. Они не читают чужие task'и и не видят «общий план». Значит ownership надо прописывать в КАЖДОМ заинтересованном task'е явно.

### Pattern

**В task-файле owner'а файла:** в "ЧТО ДЕЛАТЬ" явно сказать что редактируем, и что ownership этого файла на нём.

```markdown
# task-04 — python-clean-code skill + skills/mod.rs (3 pub mod для волны)

## Контекст
... T04 владеет `skills/mod.rs` в этой волне — добавляет 3 pub mod сразу для всех новых скиллов (включая те которые будут реализованы T05/T06). T05 и T06 НЕ редактируют mod.rs.

## ЧТО ДЕЛАТЬ
1. Открыть `skills/mod.rs`, найти блок `pub mod ...;` и добавить 3 новых строки:
   pub mod python_clean_code;
   pub mod java_solid_patterns;
   pub mod js_async_patterns;
   Билд после этого временно красный (файлы T05/T06 ещё не созданы) — это ok, волна закрывается после всех 4 task'ов.
```

**В task-файлах других task'ов:** в "ЧТО НЕ ДЕЛАТЬ" явный запрет с указанием owner'а.

```markdown
# task-05 — java-solid-patterns skill

## ЧТО НЕ ДЕЛАТЬ
- **НЕ редактировать `skills/mod.rs`** — owns task-04, он уже добавил pub mod для java_solid_patterns. Создаём только `skills/java_solid_patterns.rs`.
- НЕ менять existing skills.
```

### Выверка

Verify что ownership работает после волны — grep'ом по истории коммитов на трогаемый файл:

```bash
git log --oneline -5 -- desktop/src-tauri/src/skills/mod.rs
# Ожидаемо: только 1 коммит из волны (T04). Если 2+ — нарушение isolation, эскалировать.
```

### Когда ownership pattern применять

- Группа task'ов в одной волне логически связана (например добавляет N новых модулей + их registration в mod.rs / index.ts / entry).
- Нельзя разделить волны или декомпозировать файл без овер-инжениринга.
- Ownership task'у добавляет пару строк кода (`pub mod Foo;` x N), не раздувая scope.
- Build между task'ами в волне может быть намеренно красный — wave gate проверяется ПОСЛЕ всех task'ов.

### Анти-паттерн

Owner объявлен в PLAN.md / orchestrator-промте, но НЕ в самих task'ах. Subagent T05 читает только свой task-файл, не видит PLAN, не видит что T04 владеет mod.rs → «улучшает заодно» и редактирует mod.rs равно как T04. Race.

**Дублирование «T04 owns mod.rs» в КАЖДОМ task'е — это не избыточно, это single source of truth для subagent'а.**

## Что считается «один файл»

Точно один файл = один путь. Например:
- `desktop/src/App.tsx` — один файл.
- `desktop/src/manager/orchestrator.ts` — один файл.
- `desktop/src-tauri/Cargo.toml` — один файл.

НЕ один файл (несмотря на близость):
- `desktop/src/components/Button.tsx` и `desktop/src/components/Button.css` — два файла, parallel'но безопасно.
- `desktop/src/api/users.ts` и `desktop/src/api/auth.ts` — два файла.
- `desktop/src-tauri/src/main.rs` и `desktop/src-tauri/src/lib.rs` — два файла.

Hint: если grep возвращает разные пути — это разные файлы.

## Granularity внутри файла

Один файл = один subagent в волне. Даже если task-A правит function foo() в файле, а task-B правит function bar() в том же файле — это конфликт.

Причина: str_replace operates на string-level, не AST-level. Если бы git был умнее (per-function lock) — можно было бы parallel. Сейчас — нет.

Если очень хочется parallel'но править разные функции одного файла — декомпозировать файл (вынести foo() и bar() в отдельные модули, импортировать в orchestrator).

## Edge cases

### 1. Append-only файлы (CHANGELOG.md)

Технически — конфликт, two append'а в одной волне race'ятся.

Практически — append-only обновления **не делаются параллельными subagent'ами**. CHANGELOG обновляется в task-финализации (один subagent), не в feature task'ах.

Если очень нужно — sequential в одной волне.

### 2. Файлы с machine-generated content (lock files)

`package-lock.json`, `Cargo.lock`, `yarn.lock` — обновляются автоматически при `npm install` / `cargo build`. Если два subagent'а параллельно делают `npm install` для разных deps — race на lock file.

**Решение:** package install / dep update — отдельный task в W1, не «между делом» в feature task'ах.

### 3. Generated source files (proto / GraphQL schemas)

Auto-generated source — same как lock files. Один task на регенерацию, остальные используют output.

### 4. Тестовые spec файлы

Если spec файл — общий spec на несколько features, и две features в одной волне → spec становится shared. Решение: либо sequential, либо разнести на разные spec файлы.

В реальном проекте spec файлы обычно разнесены по фичам (`specs/06-env.spec.ts`, `specs/19-extract-scope.spec.ts`), потому конфликтов мало.

## Что делать если file isolation нарушен

Признаки:
- Subagent B жалуется на conflict при `git pull`.
- Orchestrator после волны видит warning о merged изменениях.
- В коммите B пропали изменения A (пересохранилось без них).

Действия:
1. **Stop волну.** Не продолжать пока не резолв.
2. **Проверить через `git log <file>`** какие коммиты затронули файл, какие из них — текущей волны.
3. **Если коммиты накладывают изменения друг на друга** (одна функция в одном коммите, другая в другом) — конфликт можно вручную мержить. Создать merge commit с обоих, проверить что обе функции работают.
4. **Если коммиты overlap'ят** (одна и та же строка) — выбрать «правильный» коммит, второй откатить, переписать.
5. **Эскалация Techlead'у** для обновления PLAN.md — что было упущено в file isolation audit'е.

## Связь с другими инвариантами

- **DoD per task** — без isolation один task проверяет DoD на чужих изменениях, получает false positive.
- **Sequential confirmation between waves** — даже без isolation проблема была бы (race между волнами), но isolation её предотвращает в принципе.
- **Build wrapper scripts** — если subagent'ы делают `npm install` параллельно, lock conflicts. Wrapper'ы централизуют env setup, исключают race.
