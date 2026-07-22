# 07 — Skills Overview

Claude Skills — это folders с `SKILL.md` (+ опционально скрипты, шаблоны). Они расширяют Claude фокусированными процессами для частых задач: планирование, audit, ТЗ, QA, deploy.

Ниже — референсный набор из 8 ролей-скиллов, вокруг которого построены остальные документы шаблона. Сами скиллы в репозиторий не входят: это описание того, какие роли нужны и когда, чтобы завести свои под свой проект.

## Главное правило

**Перед использованием skill'а — прочитать его `SKILL.md`.** Не угадывать содержимое skill'а по названию или предыдущему опыту. Skills эволюционируют, версии меняются, актуальная информация в SKILL.md.

## Список skills и роли

### `/fitch` — Полный цикл проработки задачи (5 ролей)

**Когда:** UI/feature/security эпик где нужна структурированная разработка с трассируемостью требований до тестов.

**Что делает:** Создаёт `.claude/fitch/<epic-id>/contract.json`, последовательно запускает analyst → designer → tech_lead → toster → devops, итог — `FITCH-PLAN.md`.

**Output:**
- Контракт со всеми ролями (включая `not_applicable` с reason для не-релевантных).
- FITCH-PLAN.md для Chief'а — синтез контракта.
- Готовая база для `/tech-lead` который декомпозирует план в task-файлы.

**Не использовать для:** hotfix, cleanup, простых infra задач — там lite fitch (brief.md + PLAN.md без contract.json).

### `/audit` — Multi-role audit с triangulation rule

**Когда:** Перед стартом крупного эпика, перед merge'ом сложного refactor'а, для security review модуля.

**Что делает:** Параллельно запускает subagent'ов в ролях architect / engineer / skeptic (+ опционально security / qa). Применяет sliding scale (Mini → Large) под размер цели. Triangulation rule отделяет твёрдые блокеры от шума.

**Output:** Отчёт `.audits/<date>-<scope>.md` со структурой Triangulated / Probable / Solo / REJECT / Backlog.

**Не использовать для:** разовой проверки одной функции — overhead больше чем benefit.

### `/tech-lead` — Декомпозиция ТЗ в task-файлы

**Когда:** После fitch'а, нужно превратить план в исполняемый набор task'ов для Claude Code.

**Что делает:** Создаёт `.claude/tasks/<epic-id>/PLAN.md` + файлы `task-NN-*.md`. PLAN содержит wave-разбивку с parallel groups + финальный шаг merge + push.

**Output:** Самодостаточные task-файлы для subagent'ов + PLAN для orchestrator'а.

**Не использовать для:** одиночных hotfix'ов где task-файл пишется вручную в одну минуту.

### `/toster` — QA matrix + acceptance criteria

**Когда:** Эпик требует структурированной приёмки с конкретными test cases. Часть стандартного fitch'а в роли toster.

**Что делает:** Создаёт coverage matrix R → QA, каждый QA — конкретная команда с ожидаемым output'ом (никаких субъективных «работает / не работает»).

**Output:** QA01..QANN с verification commands + matrix покрытия требований.

**Не использовать для:** эпиков без test'ов (cleanup, docs refresh).

### `/dev-ops` — Инфра + deployment

**Когда:** Эпик трогает build pipeline, installer, deployment. Часть стандартного fitch'а в роли devops.

**Что делает:** План для bump version + installer build + merge + rollback. Если у проекта есть инфраструктурный MCP — может использовать его для observability.

**Output:** Финализационный task с явными командами + rollback plan.

**Не использовать для:** code-only эпиков без инфра-touchpoint'ов.

### `/pm` — Project Manager оркестратор

**Когда:** Chief даёт абстрактную задачу («хочу сделать X») и не знает с чего начать. PM решает какие роли нужны и в каком порядке.

**Что делает:** Принимает задачу на естественном языке. Сам решает: нужен ли analyst, нужен ли UI-дизайнер, нужен ли tech-lead, нужен ли toster. Запускает их последовательно, ведёт до готовых артефактов.

**Output:** Полный комплект артефактов под задачу (brief / fitch / PLAN / task-файлы / acceptance).

**Не использовать для:** конкретных команд («запусти tech-lead») — там запускается прямо нужный skill.

### `/ba-spec` — Системный/бизнес-аналитик ТЗ

**Когда:** Нужно написать ТЗ / спецификацию / требования к фиче для разработчиков. Часть стандартного fitch'а в роли analyst.

**Что делает:** Создаёт полноценное ТЗ с требованиями, контекстом, приёмкой. Учитывает существующий код проекта.

**Output:** ТЗ-документ готовый для разработчиков (или для tech-lead skill'а который превратит его в task-файлы).

**Не использовать для:** мелких изменений где ТЗ overhead — там идёт сразу task.

### `/ui-designer` — UI/UX отладка с design tokens

**Когда:** Chief жалуется на UI («некрасиво», «выровняй», «как в Confluence»). Часть стандартного fitch'а в роли designer.

**Что делает:** По строгому протоколу: видит UI (скриншот / описание), читает дизайн-систему проекта, формулирует проблемы, делает точечные изменения только токенами проекта.

**Output:** Конкретные правки + обоснование почему именно так.

**Не использовать для:** функциональных багов — это не UI, это logic.

## Композиция skills

Стандартный feature эпик использует несколько skills последовательно:

```
Chief: «хочу страницу настроек security»

  ↓

PM (или Techlead напрямую):
  /ba-spec → краткое ТЗ
  /ui-designer → mockup секции (если нужен новый UI)
  /fitch → contract.json + FITCH-PLAN.md (внутри: analyst, designer, tech_lead, toster, devops)
  /audit (опционально) → проверка fitch'а перед W1
  /tech-lead → PLAN.md + task-файлы

  ↓

Orchestrator в Claude Code → subagent'ы по task-файлам

  ↓

Финализация (через devops роль): bump version + installer + merge

  ↓

Closeся эпик. ROADMAP / CHANGELOG обновлены.
```

Skills могут запускаться напрямую (Techlead знает что нужно `/audit`) или через `/pm` (PM решает сам).

## Когда skill не нужен

1. **Hotfix.** Bug найден, fix очевиден. Один task без fitch / audit. Прямая работа с Filesystem MCP.

2. **Cleanup эпик.** Disk wipe, archive moves, docs refresh. См. `08-housekeeping-and-pre-flight.md`. Skill overkill.

3. **Известный паттерн.** Если уже сделали 5 одинаковых эпиков (например «add new importer for X tool»), 6-й не требует full fitch — копи паттерн.

4. **Discovery / spike.** Chief хочет понять «возможно ли». Один subagent с task'ом «изучи feasibility, напиши отчёт».

## Skills и память

Skills не хранят state между сессиями — каждый запуск читает SKILL.md заново. Это намеренно — позволяет улучшать skill без боязни legacy.

В памяти Techlead'а хранятся **проектные правила** (схема работы, task-format, инварианты проекта), не skill-specific детали. Skills — generic instruments, project rules — specific applications.

## Создание новых skills

Если паттерн повторяется 3+ раз через эпики и стабилизировался — кандидат на skill. Скил это:
- `SKILL.md` — описание когда применять, что делает, как запускать.
- Опционально вспомогательные скрипты.
- Опционально шаблоны output'а.

До того как паттерн стал скиллом — он живёт в `.claude/templates/` или в WORKFLOW.md.

## Анти-паттерны

1. **Skill запускается без чтения SKILL.md** — теряются специфичные правила skill'а.

2. **Skills вызываются «потому что есть»** — для тривиальной задачи запускается `/pm` который запускает 4 других skill'а. Wall-clock минуты вместо секунд.

3. **Output skill'а игнорируется** — Chief не читает FITCH-PLAN.md, говорит «давай так». Через два дня выясняется что в FITCH-PLAN.md было предупреждение про этот case.

4. **Skill output редактируется ad-hoc, не сохраняется в репо** — теряется trail. Все skill output'ы должны быть в `.claude/fitch/<epic-id>/` или `.audits/`.

5. **Несколько skills запускаются параллельно над тем же scope** — теряется coherence (analyst даёт одно ТЗ, designer строит на старой версии). Skills последовательны (хотя subagent'ы внутри одного skill'а параллельны).

## Список minimum skills для нового проекта

Если копируешь шаблон в новый проект и не хочешь сразу разбираться со всеми 8 skills — minimum:

1. **`/fitch`** — структурированный fitch.
2. **`/audit`** — проверка артефактов.
3. **`/tech-lead`** — декомпозиция в task'и.

Остальные подключаются по мере роста проекта.
