# 04 — Audits Triada

Multi-role audit с triangulation rule. Параллельная читка `.claude/templates/audit-report.md` (там копи-правь шаблон), здесь — описание процесса полного цикла + scope sizing + механика triangulation rule.

## Два контекста аудита

Triangulation rule один, но применяется в двух разных контекстах:

1. **In-flight триада (этот документ, WORKFLOW §18)** — Techlead запускает subagent'ов в ролях architect/engineer/skeptic (+опц. security/qa) на код или артефакт. Аггрегация по triangulation rule. Используется на любой стадии: ревью модуля, аудит чужого fitch'а, проверка перед merge.

2. **Внешний аудит в два прохода (WORKFLOW §23)** — отдельная read-only сессия (external auditor), swarm subagent'ов, deliverables в gitignored `.audits/`. Проход 1 — pre-epic разведка поверхности ДО fitch (фактура под планирование, опц. черновой fitch-пакет с тремя страховками). Проход 2 — fitch-package audit ДО W1 по 7 направлениям A-G (Coverage / Feasibility / File isolation / Waves / DoD greps / Invariants / Missing edges); HIGH = блокер W1 с `file:line` + правкой. Окупается тем что ловит false-green гейты и phantom-сигнатуры ДО запуска волн.

Оба прохода — тот же triangulation принцип (3+ subagent согласны = firm). Различие: триада смотрит на **готовый код**, двухпроходный — на **поверхность эпика и план до запуска**. Внешний аудитор на Opus может отдавать 529 — re-run на Sonnet. Inventory аудитора = starting point, не истина: grep-verify ДО правки.

## Идея

Один Claude (даже Opus) — один точка зрения. На сложных артефактах он пропускает проблемы которые видны другому ракурсу. Решение — **запустить несколько subagent'ов параллельно в разных ролях**, каждый смотрит на тот же артефакт со своей перспективы. Затем Techlead агрегирует findings и применяет triangulation rule.

## Роли audit'а

### Architect

Структурный фокус. Смотрит на архитектуру, layering, abstractions, single responsibility, file isolation, contract'ы между модулями. Игнорирует мелкие баги — его job найти структурные проблемы.

Типичные finding'и от architect'а:
- App.tsx модифицируется в 6 task'ах из 5 разных волн (file isolation violation)
- pipeline.sendHttpRequest нарушает чистоту через store.getState()
- task-NN добавляет работу которая дублируется task-MM в следующей волне

### Engineer

Реализационный фокус. Смотрит на конкретные баги в коде, type signatures, edge cases, off-by-one errors, race conditions, неправильные API вызовы.

Типичные finding'и от engineer'а:
- task-19 old_str использует 8-пробельный отступ, реальный код 6-пробельный
- task-29 cargo build напрямую, нарушение CLAUDE.md
- mod.rs не updated → cargo build не найдёт новый command
- max_redirects не в cache key → silent regression custom maxRedirects

### Skeptic

Композиция и REJECT-фильтр. Спрашивает «а это вообще нужно?», «не theatre ли?», «кост-бенефит». Особенно ценен в backlog'е — отсекает теоретические дефекты, которые не материализуются в реальном использовании.

Типичные finding'и от skeptic'а:
- Cookie split-brain между W4 (frontend jar) и W6 (Rust pool) — реальный сценарий
- B19 logging — backlog по операционной visibility, не блокер
- U+2028 strip в QuickJS — sandbox без fetch, blast radius мал, в backlog по запросу

### Security (опционально, для security-эпиков)

OWASP focus. Threat model, attack chains, exploit scenarios, защита данных, supply chain.

Типичные finding'и от security:
- URL mask пропускает Authorization header → JWT в screenshot leak
- Redirect SSRF через DNS rebinding TOCTOU
- Protective reviver на manager-level вместо registry-level (regression vector при добавлении нового importer'а)

### QA (опционально, для feature-эпиков)

Coverage gaps focus. Coverage matrix verification, weak assertions, hostile fixture absent, baseline flake risk.

Типичные finding'и от QA:
- task-33c физически отсутствует — R15/R16/R17 без gate-теста
- e2e/mocks/ directory отсутствует — mock-зависимые phase silently passing
- assertion `expect(error).toBeUndefined()` — слабая, не ловит lost update bug

## Triangulation rule

Главный принцип agregation'а findings от ролей.

**3+ источника** = твёрдый блокер. Если architect, engineer и skeptic независимо нашли ту же проблему — это реальный блокер, чинить обязательно перед merge.

**2 источника, верифицировано Techlead'ом** = probable. P1 priority, чинить если не блокирует merge — то в W1 hotfix.

**1 источник, верифицировано Techlead'ом** = solo. P2 или backlog. Документировать, не чинить в этом эпике.

**1 источник без verification** = backlog / dropped. Документировать как «возможный риск», ответственность за follow-up на Chief'е.

### Почему именно так

Один subagent (Sonnet) защищает первое объяснение которое пришло в голову. Если ровно тот же finding пришёл от двух разных ролей независимо — вероятность совпадения мала, finding реален. Три ролей — практически гарантия.

Solo finding'и не отбрасываются — Techlead их verifies через Filesystem MCP (читает реальный код, проверяет gap). Если verified — это реальный finding, просто менее критичный (одна роль увидела, две другие нет — может быть narrow scope).

### Composition bugs

Отдельная категория — finding'и из разных audit'ов (по разным модулям) которые **складываются в attack chain**. Они могут не быть triangulated в одном audit'е, но визуально складываются в одну реальную проблему.

Пример: hostile import → выполнение pre-request скрипта в песочнице → path traversal в form-data → XSS через отключённый CSP. Каждый компонент найден отдельно и выглядит некритично, все вместе = single-click compromise.

Triangulation не отлавливает их автоматически — Techlead делает synthesis pass поверх finding'ов всех ролей и ищет cross-cutting patterns.

## Scope sizing

| Scale | Состав | Wall-clock | Применение |
|---|---|---|---|
| **Mini** | 1 роль (engineer или skeptic) | 5-15 мин | Single file / single function review |
| **Small** | 2 роли (architect + engineer) | 30-60 мин | Single module / 5-10 файлов |
| **Medium** | 3 роли (architect + engineer + skeptic) | 1-2 часа | Feature / эпик / 20-50 файлов |
| **Large** | 5 ролей (+ security + qa) | 2-4 часа | Multi-module / cross-cutting / fitch эпика |

Wall-clock — параллельный (subagent'ы стартуют одновременно), Techlead synthesis после.

Если scope не очевиден — берёшь Medium по умолчанию. Лучше переборщить с ролями чем пропустить блокер.

## Процесс полного цикла

### Шаг 1 — Подготовка scope

Techlead определяет:
- Что аудируется (файл / модуль / эпик / fitch).
- Какой scale (Mini / Small / Medium / Large).
- Какие роли запускать.
- Что роли должны прочитать (исходный код, fitch контракт, brief).

### Шаг 2 — Параллельный запуск

Если есть skill `/audit` — через него (см. `07-skills-overview.md`). Без skill — Techlead запускает subagent'ов в Claude Code параллельно с инструкциями для каждой роли.

Каждый subagent получает:
- Scope (что читать)
- Роль (architect / engineer / skeptic / …)
- Output format — список finding'ов с pattern: `<id>` `<one-line>` / `<where>` / `<risk>` / `<fix>`

### Шаг 3 — Синтез

Techlead собирает все finding'ы в один файл `.audits/<date>-<scope>.md` по структуре `audit-report.md`:

1. **Triangulated findings (3+ sources)** — сначала. Это блокеры.
2. **Probable findings (2 sources, verified)** — после. Это P1.
3. **Solo findings (1 source, verified)** — после. P2.
4. **REJECT под сомнением** — где skeptic / security разошлись с другими ролями. Techlead резюмирует.
5. **Backlog / dropped** — single source без verification.

В каждом finding'е — `<refs>` показывающий какие роли согласились. Это даёт Chief'у возможность проверить.

### Шаг 4 — Reject pass с Chief'ом

Перед применением findings — Techlead проходит с Chief'ом каждый блокер и spec'ит:
- Реальный exploit / impact (не теоретический)?
- Cost фикса оправдан?
- В scope текущего эпика или backlog?

Skeptic-фильтр на этой стадии важен — не каждый finding нужно чинить здесь и сейчас. Theatre defense (защита от того что не происходит в реальной жизни) — отбрасываем явно с reason'ом.

### Шаг 5 — Application

Approved findings применяются как **audit-fix pass** — серия микро-волн (MW1 / MW2 / …) которая правит fitch / task-файлы / контракт перед W1 эпика.

Audit-fix pass может включать:
- Переписывание task-файлов с phantom-сигнатурами
- Создание физически отсутствующих task'ов
- Обновление contract.json (file isolation, sequential parallel_groups)
- Доработку FITCH-PLAN с честными disclaimer'ами
- Обновление ROADMAP с deferred pickup эпиками

После audit-fix pass'а — W1 стартует на чистой базе.

## Когда audit нужен, когда нет

**Нужен:**
- Medium / Large эпик (15+ task'ов).
- Security-чувствительный эпик (auth, persistence, network).
- Эпик с большим audit-debt (давно не аудировали, накопилось findings).
- Перед merge'ом крупного refactor'а.

**Не нужен:**
- Hotfix (1-2 task'а).
- Cosmetic / UX полировка.
- Cleanup эпик между фичами.

Решение «делать или нет» — Techlead с обоснованием Chief'у. Если Chief скажет «давай без audit'а» — без audit'а, но Techlead фиксирует что risk-on-Chief.

## Анти-паттерны audit'а

1. **Audit делается одной ролью** — теряется triangulation, finding'и subjective. Минимум две роли для audit'а.

2. **Audit без verification finding'ов через Filesystem MCP** — Techlead копи-пастит finding'и subagent'ов в отчёт без проверки. Реально половина их false positive (subagent угадал по памяти).

3. **Audit без reject pass с Chief'ом** — все finding'и идут в fix, чинится theatre, эпик растягивается на лишние недели.

4. **Composition bugs пропускаются** — каждая роль смотрит на свой модуль, но cross-module attack chain никто не видит. Techlead делает synthesis pass.

5. **Backlog finding'и забываются** — single source finding документируется но никогда не пересматривается. Решение: В ROADMAP заводить «pickup backlog» секцию которая ревьюится регулярно.

6. **Audit становится продакшн-блокером навсегда** — каждый finding обязан быть закрыт перед merge, эпик не закрывается. Решение: triangulated блокеры обязательны, остальное — приоритезируется по cost-benefit.

7. **Скиллы audit / fitch / pm запускаются без чтения SKILL.md** — они выполняются «как обычно», теряются специфичные правила skill'а. Перед каждым — прочитать его `SKILL.md`.

## Пример — audit-driven эпик

Эпик начинался как «расширить E2E coverage». Внешний аудит нашёл 10+ верифицированных блокеров в task'ах + 12 systemic risks. Решение — fix in place вместо pivot на security-эпик.

Цикл:
1. Update task'ов на ground truth через Filesystem MCP (исправить все phantom signatures).
2. PLAN.md с file isolation на реальных путях.
3. Wave gate с расширенным regression scope.
4. Honest CHANGELOG (что реально сделано, что отложено).

Эпик задумывался «expand E2E», по факту 6 phases + 13 prod fixes (folder merge, runner singleton, env prop). Это нормально — audit показал реальное состояние, эпик адаптировался.
