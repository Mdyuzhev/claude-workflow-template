# ROADMAP.md (template)

Живой документ с порядком работ. **Парный файл к `CLAUDE.md`.** Любой Claude который спрашивает «какое сейчас состояние проекта» — читает этот файл первым.

Замени `<PROJECT>` / `<CHIEF>` / placeholder'ы под свой проект. Удали этот блок hint'ов после первого commit'а.

---

# `<PROJECT>` — Roadmap

**Текущая дата:** `<YYYY-MM-DD>`
**Статус проекта:** `<Platform> <version> (<installer slot>)`. `<краткое описание состояния>`. **Метрики:** `<passing/failing/skip>` (smoke принято `<CHIEF>` `<date>`).

Закрытые эпики: `<list>`. REVERTED: `<list если есть>`.

---

## Принципы

- Идём по шагам сверху вниз, не скачем.
- После каждого шага — confirmation loop с `<CHIEF>`'ом, обновление этого файла, переход к следующему.
- Между шагами можно передумать и переставить — но только осознанно, с отметкой в changelog роадмапа внизу.

---

## 📅 `<EPIC-ID>` — `<Epic name>` (in progress / backlog / closed)

- **Status:** `<status>` (дата создания / обновления)
- **Slot:** `<installer prefix + version>`
- **Predecessors:** `<list>` (если есть зависимости)
- **Estimate:** `<X-Y дней>`
- **Brief:** `.claude/fitch/<epic-id>/brief.md`

### Scope

(содержимое volume 1 / wave 1)

### Что НЕ в эпике

- `<deferred item>` → `<куда уходит (PRJ-NNN.x pickup или другой эпик)>`

### Pre-flight (если требуется)

- `<grep / verify / mockup approval>`

---

## 📅 `<NEXT-EPIC-ID>` — `<Name>` (backlog)

(копия структуры выше)

---

## 📦 Infrastructure backlog (параллельно эпикам)

Backlog задач которые не привязаны к feature/security волнам — обычно технический долг, инфра-улучшения, инструменты.

Пример:
### `<infra-task>` (PAUSED / queued)

`<short description + blocker если PAUSED>`

---

## 📊 Прогресс по `<external metric>`

Если есть внешний QA-отчёт / acceptance critera матрица — таблица прогресса здесь.

| Категория | Всего | Закрыто | Остаток | Эпики |
|---|---|---|---|---|
| `<cat>` | `<N>` | `<M>` ✅ | `<R>` | `<epics list>` |

---

## 🏁 Закрытые эпики

### 🏁 `<EPIC-ID>` — ЗАКРЫТ `<date>`

`<краткое описание + installer + метрики>`
- Installer: `<slot version>`
- Metrics: `<numbers>`
- Known gaps → `<следующий эпик>`: `<list>`

**Changelog роадмапа:**
- `<date>`: `<EPIC-ID>` закрыт (`<short>`)

---

(копии для каждого закрытого эпика, последний — самый ранний)

---

## Отменённые планы

`~~<original plan>~~` — отменён `<CHIEF>`'ом `<date>`. **Причина:** `<reason>`.

---

## Возможные направления за пределами плана

После закрытия `<последний планируемый эпик>` можно рассмотреть:

- `<option 1>` — `<context>`
- `<option 2>` — `<context>`

---

## Changelog роадмапа

| Дата | Изменение | Причина |
|---|---|---|
| `<date>` | Roadmap создан | Первый коммит проекта |
| `<date>` | `<EPIC-ID>` closed | `<short>` |
| `<date>` | План пересмотрен (vN) | `<reason>` |

---

*Обновляется после каждого закрытого шага.*
