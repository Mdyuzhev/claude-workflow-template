# Audit Report Template

Шаблон отчёта аудита. Лежит в `.audits/<date>-<scope>.md`. Использует triangulation rule (3+ источника = твёрдый блокер, 2 = probable, 1 = solo verified).

Multi-role audit делается через несколько subagent'ов параллельно в Claude Code, каждый в своей роли. Затем Techlead агрегирует findings в этот отчёт.

---

# Audit: `<Scope>` — `<Date>`

**Date:** `<YYYY-MM-DD>`
**Scope:** `<что аудируется — fitch эпика, конкретный модуль, целая кодовая база>`. `<size в строках кода / количество task-файлов / etc>`.
**Scale:** `<Mini | Small | Medium | Large>` (определяет состав ролей)
**Sources:** `<self-inspection (Techlead) + architect + engineer + skeptic + security + qa>`

## Сводка

1-2 абзаца честной оценки:
- Что хорошо в целевом артефакте.
- Что является блокером для merge / запуска.
- Главный системный сигнал (что-то что повторяется через несколько finding'ов и указывает на root cause).

Пример:
> Эпик хорошо структурирован — N волн с правильной идеей "bugfix перед refactor", coverage matrix формально полна, file-isolation декларирована. Но при буквальной верификации обнаружены два жёстких блокера (вся волна WX описывает уже сделанную работу + физически отсутствующий task-NNc) и серия implementation-блокеров в WY Rust-task'ах.
>
> Главный системный сигнал: WX пройдено через analyst+tech_lead+toster fitch-фазу со ссылкой на «реальное чтение кода» — но сама верификация для этого файла не выполнена. Это нарушение нашего инварианта «не угадывать сигнатуры по памяти». Перед стартом нужен сплошной grep-pass всех task-файлов.

---

## Triangulated findings (3+ sources) — твёрдые блокеры

Findings подтверждённые тремя или более ролями. **Чинить обязательно перед merge.**

### ❌ B1 — `<one-line summary>`

**Где:**
- `<file:line>` — `<one-line context>`
- `<another file:line>` — `<context>`
- `<contract reference / fitch-plan reference>` — `<claim that contradicts reality>`

**Согласны:** architect (`<finding ref>`), engineer (`<ref>`), skeptic (`<ref>`), self-inspection.

**Риск:**
1. `<concrete failure scenario 1>`
2. `<scenario 2>`

**Фикс:** `<concrete actionable resolution с command или edit description>`.

---

### ❌ B2 — `<...>`

(копия структуры)

---

## Probable findings (2 sources, проверены кодом)

Findings подтверждённые двумя ролями + verified самим Techlead'ом через Filesystem MCP. **P1 priority, чинить если не блокирует merge — то в W1 hotfix.**

### ⚠️ P1 — `<summary>`

**Где:** `<refs>`.
**Согласны:** `<two role refs>`, `<verified by self>`.
**Риск:** `<scenario>`.
**Фикс:** `<resolution>`.

---

## Solo findings (verified by Techlead)

Findings из одного источника но verified Techlead'ом через реальный код. **P2 — backlog или quick fix если cost мал.**

### ⚠️ S1 — `<summary>`

**Source:** `<single role>`.
**Verified:** Read `<file>` confirms `<observation>`.
**Exploit scenario / risk:** `<concrete>`.
**Impact:** `<who is affected, blast radius>`.
**Фикс:** `<resolution>`.

---

## REJECT под сомнением

Findings которые скептик / security предложили отклонить. Приводим оба мнения и резюме Techlead'а.

### `<finding name>`

**Skeptic / Security:** `<position>`.
**Counter-argument:** `<other position>`.
**Резюме Techlead'а:** `<reject | accept | partial accept with backlog item>`.

---

## Backlog / dropped

Findings которые имеют один источник и **не critical**, либо нуждаются в коде-проверке которую Techlead не провёл, либо — теоретика. Документируем как «возможный риск», не чиним в этом эпике. Ответственность за follow-up — на Chief'е.

- `<finding>` (`<source ref>`). `<one-line context>`. Cost: `<estimate>`. Эпик-pickup: `<PRJ-NNN.x | next epic>`.
- (... список ...)

---

## Sources detail

### Architect summary (`<N>` findings)
Структурный фокус. Top: `<B-list>`, `<P-list>`. Полный отчёт прилагался.

### Engineer summary (`<N>` findings)
Реализационный фокус. Top: `<list>`. Verified реальный код `<file refs>`.

### Skeptic summary (`<N>` findings)
Композиция и REJECT-фильтр. Top: `<list>`.

### Security summary (`<N>` findings, OWASP)
Threat model focus. Top: `<list>`. Maps to OWASP `<A03 / A09 / A10>`.

### QA summary (`<N>` findings + matrix verification)
Coverage gaps focus. Top: `<list>`. Coverage matrix verification: `<hard gaps + risks + weak assertions count>`.

---

## Что делать перед W1 (action list для Techlead)

Конкретный пошаговый план как closeать blocker'ов перед стартом эпика:

1. `<Action 1>` — `<one-line>`.
2. `<Action 2>` — `<one-line>`.
3. `<Action N>` — `<one-line>`.

После этих изменений запускать W1 с пониманием что matrix `uncovered_requirements: []` после фикса всё ещё формальна — реальный coverage проверится только когда `<infrastructure>` встанет на место.

---

## Triangulation rule mechanics

Audit использует triangulation rule:
- **1 source** = слабый сигнал. Один роль может ошибиться, дать false positive, защитить свой первый диагноз. Solo finding документируется но не делается без Techlead verification.
- **2 sources** = вероятно. Probable finding, чинить с приоритетом P1 если не блокер.
- **3+ sources** = твёрдый блокер. Triangulated finding, чинить обязательно.

Composition bugs (когда отдельные finding'и складываются в attack chain через несколько модулей) — отдельная категория. Они могут быть **не triangulated в одном модуле** но визуально складываются в одну реальную проблему. Triangulation не отлавливает их автоматически — Techlead делает synthesis pass.

## Когда какой scope audit'а

**Mini** — single file / single function review. 1 роль (engineer или skeptic). 5-15 минут.

**Small** — single module / 5-10 файлов. 2 роли (architect + engineer). 30-60 минут.

**Medium** — feature / эпик / 20-50 файлов. 3 роли (architect + engineer + skeptic). 1-2 часа.

**Large** — multi-module / cross-cutting / fitch эпика. 5 ролей (architect + engineer + skeptic + security + qa). 2-4 часа.

Wall-clock — параллельный (subagent'ы стартуют одновременно), Techlead synthesis после.
