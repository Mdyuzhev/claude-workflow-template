# 05 — Diag-then-Fix Pattern

Паттерн для сложных багов где первичная диагностика может быть ошибочной. Sonnet'ы (executor'ы) склонны защищать первое объяснение которое пришло в голову, поэтому fix без diag-фазы часто чинит не то.

## Когда применять

Стандартный bugfix (одна функция, понятная причина, тривиальный fix) — без diag, обычный task.

Diag-then-fix применяется когда:
- Bug прошёл через существующие E2E незамеченным.
- Симптом не однозначно указывает на причину.
- Есть несколько кандидатов (race condition vs timing vs config).
- В прошлом аналогичный fix давал regression.
- Subagent в первой попытке защищает свой первый диагноз и fix не работает.

## Структура

Bug разбивается на два task'а:
- **task-NNa-diag** — добавляет инструментацию, прогоняет узкий тест, пишет отчёт. Не применяет fix.
- **task-NNb-fix** — убирает инструментацию, применяет точечные правки.

Между ними — обязательный **synthesis** от Techlead'а.

## Phase 1 — Diag

### Что делает

Subagent добавляет `[DIAG <EPIC-ID>]` markers в код:
- `console.log('[DIAG <EPIC-ID>] state at <point>:', JSON.stringify({...}))` в TypeScript
- `log::info!("[DIAG <EPIC-ID>] state at <point>: {:?}", state)` в Rust
- `dbg!()` macro в Rust для quick инспекции значений

Marker `[DIAG <EPIC-ID>]` важен — позволяет grep'ом потом проверить что все убраны в fix-фазе.

### Где добавлять

В точках которые Techlead подозревает:
- На входе в функцию (что приходит)
- На выходе (что возвращается)
- В ветвях if/match (какая ветка выбрана)
- Перед/после side-effect (state change, network call, file write)
- В catch/error блоках (что за ошибка прилетела)

Не «везде» — конкретный набор подозреваемых точек. Слишком много DIAG'ов = шум, легко пропустить сигнал.

### Узкий тест

После инструментации — прогон **узкого** теста, не full smoke:

```bash
npm run e2e:test -- --spec specs/06-env.spec.ts 2>&1 | tee .audits/diag-<bug-name>.log
```

Узкий потому что:
- Быстрее (минуты, не десятки минут).
- Сигнал не теряется в шуме других тестов.
- Можно повторить много раз для flake-проверки.

### Отчёт

Subagent пишет отчёт в `.claude/fitch/<epic-id>/diag/<bug-name>.md` со структурой:

```markdown
# Diag — <Bug name>

## Симптом
<one-line: что failin'ит, на каком тесте, как часто>

## Сбор
<какие DIAG markers добавлены, в каких файлах, какой тест запустил>

## Наблюдения
<что показали логи, какие state'ы, какие ветки выбраны>

## Анализ
<гипотезы что причина, какая наиболее вероятна, обоснование>

## Рекомендации
<2-3 конкретных fix'а с trade-off'ами, какой Techlead рекомендует>
```

**Subagent НЕ применяет fix на этой фазе.** Только инструментация + отчёт. Это критично — иначе теряется второй взгляд от Techlead'а.

## Phase 2 — Synthesis (Techlead, main chat)

Techlead читает diag-отчёт + проверяет реальное состояние через Filesystem MCP. Цель — second opinion на subagent'ский диагноз.

Subagent в diag-фазе уже инвестировал в одну гипотезу (придумал инструментацию под неё, увидел что подтверждается). Subagent склонен **защищать** эту гипотезу. Techlead должен:

1. **Прочитать subagent'ский анализ.**
2. **Проверить наблюдения через Filesystem MCP** — открыть файлы где DIAG'и стоят, прочитать вокруг них код.
3. **Сформулировать альтернативные гипотезы** — что ещё могло вызвать тот же симптом.
4. **Отбросить альтернативы** через дополнительный grep / read если они менее вероятны.
5. **Подтвердить или скорректировать** subagent'ский диагноз.

Бывают случаи когда subagent диагноз правильный, бывают — где Techlead находит другую причину. Без synthesis'а subagent применит fix под свой первый диагноз, fix не сработает или сломает что-то ещё.

После synthesis'а — Techlead готовит summary для Chief'а:

```
Diag closed.
Симптом: <one-line>
Причина: <root cause>
Fix scope: <one-line — что меняем>
Альтернативы рассмотрены: <list, почему отвергнуты>
Estimate: <X minutes/hours>
```

Chief approve scope. Если Chief хочет другой подход (или просит ещё один diag round) — обсуждаем перед fix'ом.

## Phase 3 — Fix

После approval'а scope от Chief'а — Techlead пишет `task-NNb-fix-<bug-name>.md`.

Subagent в fix-фазе:
1. Убирает все `[DIAG <EPIC-ID>]` markers (verifies grep'ом что не осталось).
2. Применяет точечные правки по согласованным рекомендациям.
3. Прогоняет узкий тест — должен passing.
4. Прогоняет полный smoke (если волна закрывается) — failure_diff пустой.
5. Коммит fix.

DoD проверяет:
- `grep -c "[DIAG <EPIC-ID>]" <touched-files> == 0` — markers убраны.
- Узкий тест passing.
- Failure diff vs previous baseline пустой.

## Полный пример

Bug: «specs/06-env.spec.ts падает в 3 describe блоках через cascade от первого `before all` hook».

### Phase Diag — task-09a-diag-06env.md

> ЧТО ДЕЛАТЬ:
> 1. В `e2e/helpers/setup.ts` добавить `console.log('[DIAG PRJ-056.1a] cleanupE2ECollections start, files:', files)` перед циклом удаления.
> 2. В `e2e/helpers/setup.ts` после `createCollection` добавить `console.log('[DIAG PRJ-056.1a] createCollection done, store state:', JSON.stringify(__COLLECTIONS_STORE__))`.
> 3. В `e2e/helpers/wait.ts::waitForAppReady` добавить timing log: `console.time('[DIAG PRJ-056.1a] waitForAppReady')` / `console.timeEnd(...)`.
> 4. Прогнать `npm run e2e:test -- --spec specs/06-env.spec.ts 2>&1 | tee .audits/diag-06env.log`.
> 5. Написать отчёт в `.claude/fitch/prj-056.1a/diag/06env.md`.
>
> Запрещено применять fix на этой фазе. Только инструментация + лог + отчёт.

### Synthesis (Techlead reads diag-06env.md)

Subagent рапортует: «cleanupE2ECollections не дожидается завершения createCollection — race на пустом store при start of next test».

Techlead reads:
- helpers/setup.ts — verified `cleanupE2ECollections` is sync, `createCollection` is async.
- helpers/wait.ts — verified `waitForAppReady` имеет timeout 5s.
- логи — `waitForAppReady` падает с timeout в первом describe, последующие — cascade.

Альтернативная гипотеза: **timeout в `waitForAppReady` на холодном старте** — слишком короткий 5s, особенно после AppData wipe + fresh build.

Verified через лог: первый прогон занял 4.8s, marginal close. Второй — 6.1s, fall over timeout.

Synthesis для Chief'а: «Diag показал не race в cleanup, а timeout 5s слишком жёсткий на холодном старте. Bump timeout до 10s или 2x baseline».

### Phase Fix — task-09b-fix-06env.md

> ЧТО ДЕЛАТЬ:
> 1. В `helpers/wait.ts::waitForAppReady` поднять timeout: `5000` → `10000`.
> 2. Убрать все `[DIAG PRJ-056.1a]` markers из `setup.ts` и `wait.ts`.
> 3. Прогнать `npm run e2e:test -- --spec specs/06-env.spec.ts` — passing.
> 4. Прогнать полный `npm run e2e:test` — failure_diff vs post-W1 baseline пустой.
> 5. Коммит.
>
> DoD: grep `[DIAG PRJ-056.1a]` по setup.ts/wait.ts == 0. 06-env spec passing 4/4. Full smoke unchanged.

## Анти-паттерны

### 1. Diag-фаза с применением fix'а

Subagent в diag «заодно» применяет fix потому что «увидел очевидное». Synthesis от Techlead'а пропускается. Fix не работает или ломает что-то ещё.

**Запрещено явно в task-NNa.** В DoD: «грep на DIAG markers > 0» — должны остаться. В «Что НЕ делать»: «НЕ применять fix на этой фазе».

### 2. Synthesis без verification

Techlead читает diag-отчёт, кивает subagent'скому диагнозу, пишет fix-task. Если subagent ошибся — fix чинит не то.

**Synthesis обязательно включает Filesystem MCP read** реального кода в подозреваемых местах.

### 3. Fix без удаления DIAG markers

Subagent применил fix но забыл убрать `[DIAG]` логи. В production летят debug логи, performance страдает.

**DoD проверяет grep на markers == 0.**

### 4. Diag-then-fix для тривиального бага

Один типо в `if (foo == bar)` где должно быть `===`. Diag-фаза избыточна, тратит сессию.

**Diag применяется только когда первичная диагностика неоднозначна.**

### 5. Узкий тест не прогоняется в diag

Subagent добавляет инструментацию, ничего не прогоняет, пишет «вероятно проблема в X». Гипотеза без данных.

**Diag без узкого прогона = просто чтение кода. Прогон обязателен.**

### 6. Между diag и fix — confirmation от Chief'а пропускается

Techlead делает synthesis, сразу пишет fix-task, отправляет orchestrator'у. Chief узнаёт что fix применился post-factum.

**Phase boundary между diag и fix — обязательная confirmation от Chief'а.** Это последняя точка где можно отменить или скорректировать direction.

## Связь с triangulation rule

Audit использует triangulation — multiple ролей независимо находят то же. Diag-then-fix использует **synthesis-pass** — разные level'и (subagent + Techlead) проверяют одну гипотезу. Принцип общий: один источник = слабый сигнал, нужен второй взгляд.

Triangulation работает горизонтально (роли в audit'е одного уровня). Synthesis работает вертикально (subagent → Techlead → Chief). Оба нужны.
