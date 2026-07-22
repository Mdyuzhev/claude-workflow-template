# 03 — Task Files Anatomy

Расширенная анатомия task-файла. Параллельно `.claude/templates/task-file.md` (там копи-правь шаблон), здесь — глубокий гид с примерами good / bad и обоснование каждой секции.

## Зачем task-файл нужен

Subagent (Sonnet в Claude Code) работает в изолированной сессии — он не видит контекст эпика, чужие task'и, audit-отчёты. Task-файл должен быть **самодостаточным**: subagent открывает его, читает, выполняет, не задавая вопросов.

Если subagent задаёт вопросы / угадывает / расширяет scope — это сигнал что task-файл недостаточен. Не ругать subagent'а, переписать task.

## Идеальный размер

| Тип task'а | Размер строк | Пример |
|---|---|---|
| Wrapper / integration | 60-90 | использовать новый helper в существующем модуле |
| Feature implementation | 100-150 | реализовать новый компонент с тестами |
| Security / crypto fix-point | 150-200 | добавить layer encryption с фиксированным форматом |
| Diag (instrumentation only) | 60-80 | добавить [DIAG] логи для recurring bug |
| Tests-only (gate task) | 80-120 | spec файл с E2E phase'ами |

Дольше 200 строк — сигнал декомпозировать на два task'а. Короче 60 — обычно есть условные правила / контекст которого не хватает, дописать.

## Структура (полная)

### Шапка (5 строк)

```markdown
# task-NN — <one-line summary>

**Эпик:** <EPIC-ID> · **Волна:** W<N> · **Платформа:** <Desktop | Web | ...>
**Зависит от:** <task-NN | предыдущая волна | нет>
**Branch:** feature/<epic-id>-<short-name>
**Working dir:** <PROJECT_PATH>
```

`Working dir` явно — иначе subagent работает в worktree без node_modules / Cargo dependencies, results невоспроизводимы.

### Контекст (1-2 абзаца)

Факты, не инструкции. Что в коде сейчас, что меняется, audit finding ID если есть.

**Good:**
> Реализация KEK через Windows DPAPI. Текущий placeholder `kek_windows.rs` (создан task-02) экспортирует stub. На windows crate 0.58 API отличается от 0.50 (PCWSTR не PWSTR, Option<*const> не Option<&mut>, CRYPTPROTECT_FLAGS typed wrapper не raw u32). Audit B4 fix.

**Bad:**
> Нужно сделать KEK для Windows. Использовать DPAPI. Возможно понадобится правка модуля.

Bad теряет: какой файл, какие версии, что Audit B4, что было stub.

### ЧТО ДЕЛАТЬ (5-15 шагов)

Императив, конкретные команды или edit описания.

**Good шаг:**
> 2. В `src/persistence/crypto/kek_windows.rs` заменить весь body функции `wrap_dek` на:
>    ```rust
>    fn wrap_dek(plaintext_dek: &[u8]) -> Result<Vec<u8>, KekError> {
>        let blob = DATA_BLOB { cbData: plaintext_dek.len() as u32, pbData: plaintext_dek.as_ptr() as *mut u8 };
>        // ...
>    }
>    ```

**Bad шаг:**
> 2. Добавь функцию wrap_dek которая шифрует DEK через DPAPI. Если возможно — сделай также unwrap_dek. Можно использовать вспомогательные функции если потребуется.

Bad содержит: «если возможно», «если потребуется» — subagent интерпретирует буквально и либо не делает либо делает что-то не то.

### ЧТО НЕ ДЕЛАТЬ (3-8 пунктов)

Список запретов. Каждый — однозначный, с reason'ом.

**Good:**
> - НЕ менять `KekError` enum — другие task'и его используют. Только новые varianты добавлять если действительно нужны.
> - НЕ возвращаться к pattern `Vec<u8>::from_raw_parts` (audit S3 явно отвергает — нарушает invariant'ы).
> - НЕ интегрировать с DEK store — это task-04 (отдельный).
> - НЕ убирать `Zeroizing<>` обёртки — защита от memory dump.

**Bad:**
> - Не делать ничего лишнего.
> - Не ломать другие модули.

Bad — формальный, не блокирует ничего конкретного.

### DEFINITION OF DONE

Одна команда (или несколько коротких grep'ов) с machine-readable output. Проверяема третьим лицом без контекста.

**Good:**
```bash
grep -c "fn wrap_dek" desktop/src-tauri/src/persistence/crypto/kek_windows.rs
# == 1

grep -c "fn unwrap_dek" desktop/src-tauri/src/persistence/crypto/kek_windows.rs
# == 1

cd desktop && cargo test --features windows-test crypto::kek_windows 2>&1 | grep "test result"
# содержит "ok. 4 passed; 0 failed"
```

**Bad:**
- «Тесты проходят»
- «Build clean»
- «Документация обновлена»

Bad не верифицируется автоматически.

### Почему так (опционально)

1-3 абзаца архитектурного обоснования. Помогает будущему reviewer'у понять контекст.

Включать когда:
- Trade-off между несколькими вариантами (объяснить выбор)
- Counter-intuitive решение (объяснить почему)
- Audit finding со сложной exploit chain (ссылка)

Не включать когда task — простой wrapper / повторяющийся pattern.

## Реальный пример (полностью good)

```markdown
# task-09 — encrypted_read_appdata + encrypted_write_appdata Tauri commands

**Эпик:** PRJ-056.1b · **Волна:** W3 · **Платформа:** Desktop (Rust)
**Зависит от:** task-04 (DekStore), task-03 (AEAD)
**Branch:** feature/prj-056.1b-encrypted-persistence
**Working dir:** <PROJECT_PATH>/desktop

## Контекст

W3 экспортирует Tauri commands для frontend'а: encrypted read/write поверх AEAD из task-03 + DEK из task-04. Audit B4 fix — block_on в Policy::custom приводит к deadlock в multi-thread runtime, поэтому commands должны быть `async fn`, не sync с block_on.

Файл `commands/encrypted.rs` пустой (создан task-02 как placeholder). Регистрация в `commands/mod.rs` уже есть (task-02). Добавить только тела функций.

## ЧТО ДЕЛАТЬ

1. Открыть `desktop/src-tauri/src/commands/encrypted.rs`. Должен быть пустой `pub mod encrypted;` placeholder.

2. Записать содержимое целиком:
   ```rust
   use tauri::State;
   use crate::persistence::crypto::{aead, dek_store::DekStore};
   use crate::persistence::storage::{appdata_path, atomic_write};
   
   #[tauri::command]
   pub async fn encrypted_read_appdata(
       app: tauri::AppHandle,
       file: String,
       dek: State<'_, DekStore>,
   ) -> Result<String, String> {
       let path = appdata_path(&app, &file).map_err(|e| e.to_string())?;
       let dek_bytes = dek.get().map_err(|e| e.to_string())?;
       let blob = tokio::fs::read(&path).await.map_err(|e| e.to_string())?;
       let plaintext = aead::decrypt(&blob, &dek_bytes, file.as_bytes())
           .map_err(|e| format!("decrypt failed: {}", e))?;
       String::from_utf8(plaintext).map_err(|e| e.to_string())
   }
   
   #[tauri::command]
   pub async fn encrypted_write_appdata(
       app: tauri::AppHandle,
       file: String,
       content: String,
       dek: State<'_, DekStore>,
   ) -> Result<(), String> {
       let path = appdata_path(&app, &file).map_err(|e| e.to_string())?;
       let dek_bytes = dek.get().map_err(|e| e.to_string())?;
       let blob = aead::encrypt(content.as_bytes(), &dek_bytes, file.as_bytes())
           .map_err(|e| format!("encrypt failed: {}", e))?;
       atomic_write(&path, &blob).await.map_err(|e| e.to_string())
   }
   ```

3. Прогнать build:
   ```bash
   cd desktop && npm run e2e:build 2>&1 | tail -10
   ```
   Должно завершиться `BUILD SUCCESSFUL` без warnings про unused или missing.

4. Коммит:
   ```bash
   git add desktop/src-tauri/src/commands/encrypted.rs
   git commit -m "PRJ-056.1b T09: encrypted_read/write_appdata commands (R09, audit B4)"
   ```

## Что НЕ делать

- НЕ менять signature `aead::encrypt` / `aead::decrypt` — task-03 их зафиксировал, другие commands их используют.
- НЕ добавлять `block_on` или `tokio::runtime::Handle::current()` — audit B4 явно отвергает (deadlock в multi-thread runtime).
- НЕ интегрировать с collections / cookies / history manager'ами — это task-17/18/19 (W5, отдельные).
- НЕ убирать AAD `file.as_bytes()` — защита от swap-attack между файлами.
- НЕ использовать `cargo build` напрямую — только через `npm run e2e:build` (CLAUDE.md инвариант).

## DEFINITION OF DONE

```bash
grep -c "encrypted_read_appdata" desktop/src-tauri/src/commands/encrypted.rs
# == 1

grep -c "encrypted_write_appdata" desktop/src-tauri/src/commands/encrypted.rs
# == 1

grep -c "block_on" desktop/src-tauri/src/commands/encrypted.rs
# == 0

cd desktop && npm run e2e:build 2>&1 | grep -c "warning:"
# == 0

cd desktop && npm run e2e:build 2>&1 | grep "Finished"
# содержит "Finished `release-dev`"
```

## Почему так

`async fn` с `tokio::fs::read/write` вместо sync `std::fs` — task должен запускаться внутри Tauri command runtime (multi-thread tokio). `block_on` на async runtime приводит к panic'у current_thread варианта или deadlock'у multi-thread варианта (audit B4 со ссылкой на reqwest issue).

AAD = `file.as_bytes()` — защита от swap-attack: encrypted blob от collections.json не дешифруется как cookies.json. Если AAD не использовать, attacker может swap'нуть файлы и получить decrypt с valid integrity check.
```

Размер: 95 строк. В пределах "Wrapper / integration" категории.

## Часто встречающиеся анти-паттерны

### 1. Placeholder типа «если возможно»

```markdown
2. Если возможно, добавь handler для гранулярной ошибки.
```

Subagent: либо добавляет какой попало handler, либо игнорирует pattern. Что хотел Techlead — неясно. **Удалить «если возможно», написать конкретное условие или убрать пункт.**

### 2. Phantom signature

```markdown
2. Вызвать `manager.exportToPostman(collection, options)` для сериализации.
```

А реальная signature — `manager.exportToPostman(collection)` без options. Subagent либо упадёт на compile, либо угадает что-то под именем options. **Перед task'ом — read реального файла через Filesystem MCP, копирнуть точную signature.**

### 3. Drift line numbers

```markdown
2. В `RequestPanel.tsx` на строках 245-260 заменить...
```

Реально 360-373 (drift после PRJ-045). Subagent делает str_replace на блок 245-260, не находит match → free-form правки. **Использовать anchor by content** (например «найти блок начинающийся с `const handleSendRequest = useCallback`»).

### 4. DoD «Тесты проходят»

```markdown
## DoD
- Тесты проходят.
- Build clean.
```

Не verifиable. **Заменить на concrete команды с expected output.**

### 5. Длинное прибитое тело функции для не-security task'а

100 строк task-файла занимает code body который subagent сам бы написал по signature + DoD. **Прибиваем body только для security / crypto / format-критичных task'ов** — остальное subagent пишет сам.

### 6. Отсутствие "Что НЕ делать" блока

Без явных запретов subagent склонен «улучшить заодно» — отрефакторить соседний код, добавить «полезные» комментарии, мигрировать на новый pattern. **Список запретов это блокирует.**

### 7. Task на несколько файлов без указания platform

```markdown
2. Также обновить аналогичный код в IDEA plugin.
```

Subagent работает в Desktop репо, не знает что делать с IDEA plugin. **Разбить на два task'а с явным platform в каждом**, или зафиксировать что platform только Desktop.

### 8. Type / struct name collision с существующим кодом

_Опыт PRJ-018 W3 — task-09 объявил `RunWithReport`, не зная что в репо это имя уже использовалось другой структурой (`{run: Run, report: Report}` для `report_repo::get_with_run`). Subagent адаптировал на лету (переименовал в `RunStatsRow`) — пронесло, но мог сломать существующий код._

Phantom signature (anti-pattern #4 выше) ловит «метод которого нет». Type collision — обратная проблема: **имя занято существующим типом**, который task-файл не упомянул.

Проявления:
- `pub struct Foo` объявляется заново → compile error на duplicate definition.
- `interface Foo` в TS объявляется → silent shadowing если import order такой что новый перекрывает.
- Subagent при collision либо переименовывает на ходу (best case, как в PRJ-018), либо ломает existing code (worst case).

**Решение — verify перед declaring новых types в task-файле:**

```bash
# Перед написанием task-файла — Techlead grep'ает имена которые планирует объявить
grep -rn "struct RunWithReport\|struct RunStatsRow" desktop/src-tauri/src/
grep -rn "interface RunWithReport\|interface RunStatsRow" desktop/src/
# Если match — выбрать имя без коллизии или явно описать в task'е что переименовываем
```

Если коллизия найдена и не фиксится переименованием в task'е — вынести в отдельную housekeeping micro-task ("переименовать существующий Foo в FooLegacy") в начало волны.

**В task-файле явно фиксировать, если новый тип близок по семантике к существующему:**
```markdown
## Контекст
... Создаём `RunStatsRow` (НЕ путать с `RunWithReport` — последний используется в report_repo::get_with_run для drill-down API; новый — denormalized flat row для Statistics screen).
```

_Это блокирует и subagent'а, и читателя через 6 месяцев от попытки «упростить дублирование» удалив один из типов._

## Чек-лист перед отправкой task-файла оркестратору

- [ ] Размер 60-200 строк под тип task'а.
- [ ] Working dir явно указан.
- [ ] Контекст содержит факты, не инструкции.
- [ ] Все signatures / line refs verified Filesystem MCP'шной read'ой реального кода.
- [ ] **Все новые имена типов / структур / интерфейсов проверены grep'ом на коллизии с существующим кодом** (опыт PRJ-018 W3).
- [ ] Каждый шаг ЧТО ДЕЛАТЬ — однозначен, без «если возможно».
- [ ] Список ЧТО НЕ ДЕЛАТЬ — конкретные запреты с reason'ами.
- [ ] **Если task зависит от ownership другого task'а (не редактирует X — owns task-NN) — это явно в "ЧТО НЕ ДЕЛАТЬ"** (опыт PRJ-018 W2, см. `06-file-isolation.md` раздел "Ownership declarations across tasks").
- [ ] DoD — concrete bash команды с expected output.
- [ ] Если task сложный — есть «Почему так» абзац.
- [ ] Anchor'ы — by content, не line numbers.
