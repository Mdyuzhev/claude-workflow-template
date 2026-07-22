# Task File Template

Имперaтив, размер 60-200 строк под сложность темы. Полный body функций прибивать гвоздями только для security/crypto fix-points (где формат критичен). Wrapper / integration / single-helper task = 60-90 строк.

Заменяй `<placeholder>` на свои значения. Удаляй любые блоки которые не применимы к task'у. **НЕ оставляй placeholders типа «если возможно», «по необходимости», «примерно», «около»** — они интерпретируются буквально и ломают исполнение.

---

# task-NN — `<one-line summary>`

**Эпик:** `<EPIC-ID>` · **Волна:** W`<N>` · **Платформа:** `<Desktop | Web | Backend | Mobile>` (язык: `<TS / Rust / Python>`)
**Зависит от:** `<task-NN из этой же волны | предыдущая волна | placeholder>` (если нет зависимостей — пиши «нет»)
**Branch:** `feature/<epic-id>-<short-name>`
**Working dir:** `<PROJECT_PATH>` (важно для subagent'а — иначе работает в worktree без node_modules)

## Контекст

1-2 абзаца **фактов** (не инструкций):
- Какая ситуация в коде сейчас (файл / строка / функция).
- Что именно надо изменить и почему (audit finding ID если есть, например «Audit B5 fix»).
- Что НЕ нужно менять (если есть смежные изменения которые могут показаться очевидными но в другом scope'е).

Пример хорошего контекста:

> Реализация KEK через Windows DPAPI. Текущий placeholder `kek_windows.rs` (создан task-02) экспортирует stub. На windows crate 0.58 API отличается от 0.50 (PCWSTR не PWSTR, Option<\*const> не Option<&mut>, CRYPTPROTECT_FLAGS typed wrapper не raw u32). Audit B4 fix.

## ЧТО ДЕЛАТЬ

Список конкретных шагов. Императив. Каждый шаг — единственное действие с явной командой или str_replace.

1. Открыть `<file path>`. (Read через `view` или Filesystem MCP.)

2. Записать реализацию (целиком, заменяет всё содержимое файла) **или** сделать str_replace в указанном блоке:

   ```rust
   // <PROJECT> <REQ-ID> — <one-line>
   //
   // Audit <FINDING-ID> fix: <what changes>
   <code body>
   ```

   **Условные правила** (если применимо):
   - Если cargo build / npm build выдаст конкретный type mismatch — адаптировать под сообщение компилятора. Запрещено возвращаться к ранее отвергнутым antipattern'ам (явно перечислить какие).
   - Если grep по anchor не находит — НЕ применять free-form правки, эскалировать Techlead'у.

3. Прогнать unit / integration tests:
   ```bash
   cd <PROJECT_PATH> && <test command> 2>&1 | tail -20
   ```

   Ожидание: `<concrete output expectation>` (например `test result: ok. >= 8 passed; 0 failed`).

4. Коммит с conventional message:
   ```bash
   git add <files>
   git commit -m "<EPIC-ID> T<NN>: <one-line summary> (<REQ-ID>, audit <FINDING-ID>)"
   ```

## Что НЕ делать

Список запретов. Каждый — однозначный.

- НЕ менять `<API surface>` — другие task'и его используют.
- НЕ возвращаться к `<deprecated pattern>` (audit `<FINDING-ID>` явно отвергает).
- НЕ интегрировать с `<higher-layer thing>` — это task-NN+M (отдельный).
- НЕ убирать `<defensive check>` — защита от `<scenario>`.
- НЕ использовать `<wrong tool>` — нужен `<right tool>` (project invariant).

## DEFINITION OF DONE

Одна команда с machine-readable output, либо несколько коротких grep'ов проверяющих ключевые маркеры.

```bash
grep -c "<expected pattern>" <PROJECT_PATH>/<file>
```
Должно быть `<exact number>`.

```bash
grep -c "<another marker>" <PROJECT_PATH>/<file>
```
Должно быть >= `<min>`.

```bash
cd <PROJECT_PATH> && <test command> 2>&1 | grep "test result"
```
Содержит `<concrete passing line>`.

**DoD проверяется третьим лицом без контекста эпика.** Если subagent выполнил DoD команды и они прошли — task'а закрыт.

## Почему так (опционально)

1-3 абзаца обоснования архитектурного решения. Кратко — цель раздела помочь будущему reviewer'у понять контекст без копания в audit'е и предыдущих task'ах.

Включать когда:
- Есть несколько вариантов и выбран один с trade-off (объяснить).
- Audit finding со сложной exploit chain (ссылка на сценарий).
- Counter-intuitive решение (например «возвращаем empty Vec потому что иначе leak DEK через dek.bin»).

Не включать когда task — простой wrapper или паттерн повторно применяемый по проекту.

---

## Анти-паттерны task-файлов (что точно не делать)

1. **Длинные тела функций прибитые гвоздями** для не-security task'а. Subagent сам напишет тело по сигнатуре. Прибиваем только signature + DoD + ключевые точки (например crypto blob layout).

2. **Placeholder типа «если возможно», «примерно ~X строк», «около функции Y»** — subagent интерпретирует буквально, не находит exact match, делает free-form правки. Запрещено.

3. **DoD «Тесты проходят» без команды.** Не верифицируется. Используй concrete grep / count.

4. **Phantom-сигнатура** (метод которого нет в реальном коде, угадан по памяти). Перед каждым task'ом — `read` реального файла через Filesystem MCP.

5. **Drift line numbers** («строки 245-260» в task'е, реально 360-373). Использовать anchor by content, не line number.

6. **Task на платформу которой в проекте нет** или работа в неправильной working directory. Явно указывать platform + working dir.

7. **Task который трогает file isolation owner другого task'а в той же волне.** File isolation invariant — в одной волне один файл правит один subagent.

8. **Task без явного «что НЕ делать» блока.** Subagent склонен «улучшить заодно» — список запретов это блокирует.

---

## Размер таска под тип работы

| Тип | Размер | Пример |
|---|---|---|
| Wrapper / integration / single helper | 60-90 строк | task-19 cookies.rs use encrypted helpers |
| Feature implementation (с testами) | 100-150 строк | task-23 schema visitor depth + warnings |
| Security / crypto fix-point | 150-200 строк | task-04 DEK lifecycle (chmod 0600, retry-on-corrupt, Zeroizing) |
| Diag (instrumentation only) | 60-80 строк | task-NNa-diag перед task-NNb-fix |
| Tests-only (gate task NNc) | 80-120 строк | task-22c cookie suite spec |

Если ловишь себя на дублировании boilerplate или избыточных комментариях — ужимай. Если task длиннее 200 строк — это сигнал что декомпозировать на два task'а.
