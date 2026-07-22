# WORKFLOW.md — Operational Procedures

Все процессы проекта с § нумерацией. Task-файлы и orchestrator commands ссылаются на эти разделы (например «§15.1 smoke policy» или «§19 wave command template»). При изменении нумерации — обновлять ссылки grep'ом.

Документ операционный и stack-agnostic. Конкретные команды сборки/тестов/путей — в `docs/09-runtime-cookbook.md` (per-stack), не здесь.

---

## §0 — Карта документов проекта

Разделение ответственности между документами. Operational-документы **линкуют** на REFERENCE/archive, не дублируют содержимое.

| Документ | Роль | Кто и когда обновляет |
|---|---|---|
| `.claude/CLAUDE.md` | Тонкий operational: идентификация, модель работы, инварианты, ссылки. Читается первым. | Techlead, при смене инвариантов/состояния |
| `.claude/ROADMAP.md` | Очередь эпиков + текущее состояние + lineage закрытых | Techlead, на каждом close |
| `.claude/WORKFLOW.md` | Этот файл — процессы §1-§N | Techlead, когда паттерн эволюционировал |
| `.claude/REFERENCE.md` | Живые правила + lessons learned (накопительно) | Techlead, на каждом close (новый L-NNN) |
| `.claude/archive/HISTORY.md` | Летопись закрытых эпиков (хронология) | Techlead, на каждом close |
| `.claude/<handoff>.json` | Контракт переноса состояния между сессиями | Techlead, на чекпоинте + при close |
| `CHANGELOG.md` | Release notes per installer/version | Techlead, в финализации |
| `docs/01..09` | Учебники: примеры good/bad + обоснование (WORKFLOW = императив, docs = почему) | Techlead, редко |

Принцип: **CLAUDE.md не хранит lessons, REFERENCE.md не хранит процессы, WORKFLOW.md не хранит состояние.** Каждый факт — в одном месте.

---

## §1 — Сессионный старт

Любая сессия Claude (Opus или Sonnet) где требуется планирование, audit, или task-файлы начинается так:

0. **Если есть handoff-контракт** (`.claude/<handoff>.json`) и Chief дал session-start триггер — прочитать его **первым** через Filesystem MCP (см. §21). Это самый свежий снимок состояния, свежее памяти.
1. `Filesystem:read_text_file` → `.claude/CLAUDE.md` — главные правила
2. `Filesystem:read_text_file` → `.claude/ROADMAP.md` — текущее состояние и очередь
3. `Filesystem:read_text_file` → `CHANGELOG.md` (head, ~100 строк) — последние релизы
4. Опционально `Filesystem:read_text_file` → `.claude/REFERENCE.md` — lessons learned

Без этих чтений Claude работает с устаревшими предположениями и плодит regression. **Память может отставать от диска** — версии, HEAD, статусы долга verify по диску, не по памяти.

---

## §2 — Стандартный цикл эпика

```
session start (§1) — handoff/CLAUDE/ROADMAP
   ↓
git clean check (через orchestrator): git status + git branch --no-merged main
   ↓
pre-epic audit (опц., §23 проход 1) — разведка поверхности → фактура под fitch
   ↓
brief.md (lite) ИЛИ fitch contract.json (full, 5 ролей §6)
   ↓
PLAN-<epic-id>.md в .claude/tasks/<epic-id>/ + task-файлы
   ↓
fitch-package audit (опц., §23 проход 2) — A-G ДО W1, HIGH = блокер
   ↓
W1 → gate+verify (§7) → confirm → W2 → ... → Wn → confirm
   ↓
финализация: bump version (§24 lockstep) + installer + CHANGELOG + merge --no-ff → main
   ↓
verify по диску: .git/refs HEAD = origin/main = tag (§22)
   ↓
post-close: REFERENCE lesson + HISTORY + ROADMAP + memory + process prune (§25)
   ↓
архив .claude/tasks/<epic-id>/ → done/<epic-id>/
```

Каждая стрелка между волнами — обязательная confirmation от Chief. Без подтверждения orchestrator не стартует следующую волну.

---

## §3 — Кто что пишет

| Артефакт | Автор | Платформа |
|---|---|---|
| Brief (для backlog эпика) | Techlead | main chat |
| `contract.json` | Techlead через `/fitch` skill (5 ролей) | main chat |
| `FITCH-PLAN.md` | Techlead (синтез контракта) | main chat |
| `PLAN-<epic-id>.md` | Techlead | main chat |
| `task-NN-*.md` | Techlead | main chat |
| Orchestrator wave command | Techlead | main chat |
| Pre-epic / package audit | Techlead (или external auditor) через swarm + triangulation | отдельная сессия |
| ROADMAP / CHANGELOG / REFERENCE / HISTORY | Techlead | main chat |
| handoff.json | Techlead (чекпоинт) | main chat |
| Code commits | Subagent | Claude Code |
| Wave gate report | Orchestrator | Claude Code → main chat (текстом) |

Subagent НИКОГДА не пишет meta-документы (ROADMAP / CHANGELOG / REFERENCE / handoff). Только commits.

---

## §4 — Стандартный цикл задачи

1. `git status` + `git branch --no-merged main` (через orchestrator) — должно быть чисто.
2. (опц.) pre-epic audit (§23 проход 1) — если эпик крупный / brownfield / high-risk.
3. Чтение реального кода: `list_directory` + `read_multiple_files` по затрагиваемым файлам. **Не угадывать сигнатуры.**
4. `/fitch` skill для планирования (если эпик с UI/feature/security импликациями).
5. Создание `PLAN-<epic-id>.md` с волнами + confirmation loop.
6. (опц.) fitch-package audit (§23 проход 2) ДО W1.
7. Оркестратор + subagent'ы по task-файлам, с verify по диску между волнами (§22).
8. Финализация: bump version (§24) → installer → push → confirm.
9. Post-close (§25): REFERENCE / HISTORY / ROADMAP / memory / process prune.
10. Архив `.claude/tasks/<epic-id>/` → `done/<epic-id>/`.

---

## §5 — Diag-then-fix паттерн

Для всего сложнее «один файл одна строка» — пара task-NNa-diag + task-NNb-fix:

- **Diag-фаза:** добавляет `[DIAG <EPIC-NAME>]` инструментацию (console.log, log::info, dbg!), прогоняет узкий тест, пишет отчёт в `.claude/fitch/<epic-id>/diag/<name>.md` со структурой Симптом / Сбор / Анализ / Рекомендации. Subagent на этой фазе НЕ применяет fix.
- **Synthesis (Techlead в main chat):** проверяет диагноз через Filesystem MCP (subagent защищает первое объяснение, Techlead — второй взгляд). Готовит summary для Chief. Получает scope confirmation.
- **Fix-фаза:** убирает [DIAG] инструментацию и применяет точечные правки. DoD проверяет `grep=0` на DIAG markers + passing test.

Synthesis между фазами **обязателен** — без него subagent защищает свой первый диагноз и предлагает fix который не работает. Тот же принцип в audit'ах (§18, §23): один источник = слабый сигнал.

---

## §6 — Fitch (full vs lite)

**Full fitch** — для UI/feature/security эпиков:
- `contract.json` через 5 ролей: analyst (требования R01-RNN), designer (UI / UX), tech_lead (волны и задачи T01-TNN), toster (QA matrix QA01-QANN), devops (installer / merge / rollback).
- Каждая роль либо `done` либо `not_applicable` с reason.
- Итог в `FITCH-PLAN.md` для Chief.
- Если в проекте есть skill `fitch` — использовать, прочитав его `SKILL.md` перед запуском.

**Lite fitch** — для инфра/cleanup/hotfix эпиков:
- `brief.md` (1-2 страницы) + `PLAN-<epic-id>.md` + task-файлы. Без 5 ролей, без contract.json.
- Допустим full с честными `not_applicable` если хочется трассируемости.

Решение Full vs Lite — на момент старта, Techlead обосновывает Chief'у.

---

## §7 — Wave gate

Каждая волна в PLAN.md заканчивается **gate** — конкретный набор проверок до перехода к следующей:

1. **Build green:** компиляция через wrapper-скрипт (§13). Нулевая ошибка.
2. **Test green:** полный automated test run без `--spec` фильтра (§10). Метрики passing/failing/skip vs baseline.
3. **Failure IDs diff:** список failed test'ов после волны минус список до волны = пустой (нет новых regression).
4. **Verify по диску (§22):** критичные заявления оркестратора (smoke-метрики, существование файла, hash, timestamp) Techlead проверяет сам через Filesystem, НЕ принимает на слово.
5. **Confirmation от Chief:** orchestrator пишет отчёт по §19 шаблону, Chief подтверждает.

Gate без всех пяти пунктов — invalid. Если failure_diff non-empty — diag-then-fix микро-волна перед следующей основной.

**Fix-in-wave (стандарт):** если spec в текущей волне поймал баг из предыдущих волн — fix в этой же волне, не отдельным эпиком. После fixes волны — installer пересборка обязательна, если эпик с bump (§24).

---

## §8 — File isolation в волне

Главный инвариант параллельного выполнения. Два subagent'а не могут трогать один файл в одной волне — даже если правят разные функции. Race на read-modify-write.

**Резолвы:**
- Sequential в одной волне (один subagent делает task-A, потом task-B на тот же файл).
- Перенос task'а в следующую волну.
- Декомпозиция файла на два модуля до старта эпика (если архитектурно валидно).

App entry / shared singletons (роутер, главный layout, manager-синглтоны) — **одна задача за волну**. Между волнами modify ок, но subagent ДО edit читает current state файла. Techlead проверяет file isolation grep'ом по task-файлам **перед** финализацией tech_lead-фазы fitch'а. См. `docs/06-file-isolation.md`.

---

## §9 — Confirmation loops

Обязательные точки подтверждения от Chief:

1. **После brief / fitch** — approve scope перед PLAN.md.
2. **Между diag и fix** — approve рекомендации diag-отчёта перед fix-фазой.
3. **Между волнами эпика** — approve gate report (после verify §22) перед следующей волной.
4. **Перед финальным smoke** — approve что готовы к финализации.
5. **Перед merge → main** — approve build + push.

В потоковом режиме (Chief сказал «пиши пачкой» / «продолжай» / «дальше») — сигнализация волнами без confirmation между микро-шагами («W1 готова, иду к W2»). Confirmation эпика (точки 1, 5) остаётся всегда.

---

## §10 — Smoke policy

**Стандартная политика — один полный automated test run закрывает gate.** Не три, не два — один.

Три прогона только при подозрении на flake (метрики меняются между runs без code changes). Тогда intersection из 3 runs (только consistent failures) = baseline.

`--spec` / `--filter <name>` — **только для diagnostic** одного файла. Закрывать gate узким прогоном нельзя — другие тесты регрессируют незаметно.

**Парсинг лога — скриптом, не глазами.** Если есть `scripts/parse-smoke-log` (или аналог) — orchestrator ОБЯЗАН использовать его для извлечения P/F/S, не читать весь лог вручную (дорого по токенам + риск ошибки парсинга). Путь tee лога указывать **явно** в команде (`.audits/<epic>-<wave>/smoke.log`) — иначе лог уезжает не в тот каталог и hook его не подхватывает. На all-green (0 failed) формат summary может отличаться — fallback на «N passed» из заголовка.

### §10.1 — Manual visual smoke для UI-волн

Automated прогон закрывает gate, но **не заменяет визуальную проверку**. В волне создания UI-компонента (W5-style) manual visual check **обязателен** и делается в той же волне, не в конце эпика.

Причина: tsc + unit + E2E ловят не всё. Пустая панель пройдёт автотесты, если spec ассертит только данные в manager/store, а UI рисуется из другого канала. CSS-переменная записанная в DOM (WRITTEN) не доказывает что она реально потреблена (CONSUMED) — utility-классы перекрывают через cascade. Manual check / computed-style assertion ловит это, автотест на «var в DOM» — нет.

Manual smoke не может закрыть gate (gate = зелёный automated run), но его отсутствие в UI-волне = известный риск пустого/сломанного UI при зелёных автотестах.

---

## §11 — E2E session hygiene

Перед полным smoke прогоном:
1. **Свежий бинарь** — собран после последнего commit'а (build перед smoke, не переиспользовать старый).
2. **Чистый persistent storage** — удалить **весь каталог** persistent state приложения (`rm -rf` каталога, не только `*.json` — WebKit cache / localStorage / IndexedDB тоже конфликтуют).
3. **Закрытые предыдущие процессы** приложения.

Порядок финализационного прогона: build → cleanup → smoke → merge. Нарушен порядок — результат недоверителен.

### §11.1 — Async UI assertion timing

UI меняется через store/state с async-эффектом (React useEffect, Vue watch) — sync callback теста НЕ ждёт rerun. Паттерн «set → pause → read в одном execute-блоке» падает на части прогонов. Правильно: set в execute #1 → pause(100) → read в execute #2. Применять для любого E2E где assert на DOM/CSS-эффект после изменения состояния.

---

## §12 — Branch management

- Feature эпик: `feature/<epic-id>-<short-name>`.
- Hotfix: `hotfix/<issue-id>-<short>`.
- Housekeeping: `chore/<epic-id>.0-housekeeping`.

Merge только через `git merge --no-ff main` (preserve history). После merge — удалить feature branch. `git status` + `git branch --no-merged main` — обязательная проверка чистоты перед стартом нового эпика.

После чужого merge/push/tag — Techlead verify через Filesystem чтением `.git/refs/heads/main` + `.git/refs/remotes/origin/main` + `.git/refs/tags/*` + `HEAD` (§22).

---

## §13 — Build system discipline

Все builds — **только через wrapper-скрипты** (npm/make/таск-раннер). Subagent НИКОГДА не вызывает базовый toolchain напрямую (cargo build, vite build, rustc, tsc-as-build) — нарушение CLAUDE.md инвариант.

Причина: wrapper'ы содержат guards (версия рантайма, рабочий каталог, env-флаги), env setup, и cleanup (kill старых процессов). Прямой вызов теряет это, ломает воспроизводимость.

Точные имена скриптов и подводные камни (какой скрипт для какого target, чем заменить отсутствующий `build:check`, особенности per-crate тестов) — в `docs/09-runtime-cookbook.md`, не здесь.

---

## §14 — Скиллы (skills)

Если в проекте используются Claude Skills — Claude **читает `SKILL.md` перед использованием**, не угадывает по названию.

Типичный набор ролей-скиллов: планировщик эпика (5 ролей), multi-role audit, декомпозиция ТЗ в task-файлы, QA matrix, devops/инфра, аналитик ТЗ, UI/UX. Skill — folder с `SKILL.md` и опц. скриптами/шаблонами. Сами скиллы в шаблон не входят — заводи свои. Никогда не угадывать содержимое — читать.

Personal skills живут в Skills UI клиента, НЕ в репо. Project skills — `.claude/skills/`. Templates — `.claude/templates/`.

---

## §15 — Definition of Done

DoD task-файла = **одна команда с конкретным machine-readable output**. Не «код написан», не «тесты добавлены», не «работает».

Good DoD:
- `<test-cmd> 2>&1 | grep -c "0 failed" >= 1`
- `<unit-cmd> crypto::aead 2>&1` → `9 passed; 0 failed`
- `grep -c "fn validate_name" <path> == 1`

Bad DoD: «Тесты проходят» / «Build clean» / «Документация обновлена».

DoD проверяема третьим лицом без контекста эпика. Subagent выполняет команду, видит число, comparison verifies done.

**Грейп аккуратно:** `export function X` = ОДНА строка, не считать `def` + `export` = 2. `testid` в task-файле = `testid` в spec — grep-verify из spec ДО написания task'а, не брать из памяти (testid drift между task и spec — частый ложный gate).

### §15.1 — Smoke policy (см. §10)
### §15.2 — E2E session hygiene (см. §11)

---

## §16 — Scope discipline

- Не чинить «заодно» вне scope эпика.
- Не добавлять в fix больше чем нашёл diag.
- Не писать «N bugs fixed» если часть отложена — отложенное в CHANGELOG known issues + ROADMAP pickup backlog.
- Сомневаешься — спроси Chief'а, не додумывай.

Ловушка: эпик начинается как «21 bugs fix», по факту W7 не делается, в CHANGELOG идёт «21 fixed» — враньё. Правильно: «15 fixed, 6 deferred → <epic>.x pickup».

Для миграционных эпиков — grep + категоризация ВСЕХ источников хардкода (prod-код / specs / fixtures / scripts / configs) ДО brief'а, не только «очевидной» категории. Иначе scope expansion на середине + known issues при close.

---

## §17 — Diag-then-fix expanded (см. §5)

Полный паттерн с примерами в `docs/05-diag-then-fix.md`. Дополнительно: когда баг прошёл existing E2E незамеченным — написать failing test ПЕРВЫМ коммитом `[FAIL-TEST]`, прогнать на pre-fix состоянии чтобы подтвердить падение, затем fix.

---

## §18 — Audit triangulation (in-flight триада)

Audit через несколько subagent'ов параллельно в разных ролях (architect, engineer, skeptic, опц. security и qa). Techlead агрегирует findings:

- **Triangulated (3+ источника)** — твёрдый блокер. Чинить.
- **Probable (2 источника, верифицировано кодом)** — вероятный дефект. P1.
- **Solo (1 источник, верифицировано Techlead'ом)** — возможный дефект. P2 или backlog.
- **Backlog / dropped** — single source без верификации. Документировать, не чинить.

Audit не превращается в продакшн. Findings не применимые в реальной жизни (deprecated patterns, false positives, theatre defense) — отклонять явно с обоснованием. См. `docs/04-audits-triada.md`.

---

## §19 — Orchestrator wave command template

Когда Techlead передаёт волну оркестратору, использует шаблон `.claude/templates/orchestrator-wave-command.md`. Самодостаточный — placeholder'ы заполняются под волну, оркестратор не ищет контекст.

5 блоков: PRE-FLIGHT / WAVE START / FILE ISOLATION / WAVE GATE / ESCALATION.

Variations: Parallel N subagent'ов / Sequential 1 subagent через несколько task'ов / Diag-then-fix (две фазы) / Финализация (bump + installer + merge).

Команда оркестратору — короткая отсылка: «Прочитай `<path>` и выполни. После close — `diag/wN-report.md` и СТОП». Тело команды — в файле, не в чат (single source of truth, экономия контекста).

---

## §20 — Housekeeping эпики

Между фичами — короткие housekeeping эпики: disk cleanup, docs refresh, archive moves, pre-flight перед крупным эпиком.

Pattern: `chore/<epic-id>.0-housekeeping`. Один-два коммита (cleanup + docs sync). Audit baseline после = ±0 (код не правили). См. `docs/08-housekeeping-and-pre-flight.md`.

---

## §21 — Перенос состояния между сессиями (handoff + checkpoint)

Контекст сессии теряется между чатами. Чтобы следующая сессия Techlead'а не начинала с нуля — **handoff-контракт**: один JSON-файл (`.claude/<handoff>.json`) который держит снимок состояния.

**Структура контракта** (см. `.claude/templates/handoff.json`):
- `identity` — роль, модель, executor, стиль общения с Chief.
- `project_state` — версия, baseline-метрики, теги, main HEAD, замороженные платформы.
- `current_state` / `CLOSE_STATUS` — что закрыто/в работе, активный эпик.
- `open_items` — отложенное, pickup'ы, известные мелочи.
- `awaiting_chief` — на каком решении ждём Chief'а.
- `next_session_first_steps` — первые действия следующей сессии.
- `lessons_this_epic` — уроки текущего эпика (потом мигрируют в REFERENCE).
- `tooling` / `invariants` — снимок рабочих инструментов и инвариантов.

**Триггеры:**
- Session-start фраза от Chief (например «привет техлид») = прочитать handoff **первым** действием через Filesystem MCP, до CLAUDE/ROADMAP.
- Фраза «чекпоинт» = перезаписать handoff **целиком** через `Filesystem:write_file`, отчёт одной строкой.
- При завершении ключевого шага, при сжатии контекста, перед концом сессии — чекпоинт немедленно. Финальный чекпоинт сессии включает итог + `next_session_first_steps`.

Handoff пишется **целиком через write_file** (verify чтением), не точечными правками — это снимок, не нарастающий лог. Если в проекте был внешний checkpoint-сервис и он недоступен — handoff-файл единственный канал, дисциплина чекпоинтов критична.

---

## §22 — Verify по диску (не верить рапортам на слово)

Orchestrator/subagent возвращают **заявления**: «smoke 514/0/17», «файл создан», «hash X», «installers rebuilt», «merge сделан». Для **критичного** Techlead проверяет сам, не принимает на слово.

**Обязательно verify:**
- Smoke-метрики — читать реальный лог (`.audits/<epic>-<wave>/smoke.log`), не рапорт. Был кейс: рапорт «514/0/17», реальный лог «31/1» — потребовался re-run.
- Installer / артефакты / timestamps — `get_file_info` mtime, `list_directory_with_sizes` размер.
- Merge / push / tag — `.git/refs/heads/main` + `refs/remotes/origin/main` + `refs/tags/*` + `HEAD` читаемы через Filesystem.
- Существование/размер созданных файлов — `list_directory_with_sizes`.

Расхождения бывают и это нормально: «delta=0» а реально +1 passing; «installers rebuilt» а timestamps старые; «docs обновлены» а grep по baseline показывает старое число. Поймал расхождение — «поправка: реально X, не Y» и дальше. **Не разбирать как именно ошибся executor** — это шум, важен факт.

Принцип: доверяй, но проверяй критичные точки. Не-критичное (формулировки, промежуточные шаги) — на слово ок.

---

## §23 — Внешний аудит в два прохода

Дополнение к in-flight триаде (§18). Внешний аудитор (отдельная сессия, read-only по working tree, deliverables в gitignored `.audits/`) работает swarm'ом subagent'ов + triangulation. Два прохода:

**Проход 1 — Pre-epic разведка (до fitch).** Автономный swarm по поверхности эпика → self-audit находок (triangulation 3+ subagent = firm) → сведённая фактура под планирование. Techlead готовит REQUEST: мандат + seed-якоря (verified ground truth, НЕ контракт) + оси разведки + constraints + формат фактуры + out-of-scope. Inventory аудитора = **starting point**, не истина: subagent grep-verify ДО migrate/правки. Опционально аудитор сам пишет черновой fitch-пакет — тогда три страховки обязательны: (1) Techlead verify-downgrade по диску, (2) человек закрывает design-развилки, (3) материализацию в `.claude/` делает Techlead (аудитор read-only).

**Проход 2 — Fitch-package audit (до W1).** Аудит готового пакета (contract + PLAN + task'и) ДО запуска по 7 направлениям:
- **A. Coverage** — все требования покрыты задачами?
- **B. Feasibility** — сигнатуры в task'ах совпадают с реальным кодом?
- **C. File isolation** — два subagent'а не трогают один файл в волне?
- **D. Waves** — порядок волн корректен, зависимости соблюдены?
- **E. DoD greps** — ловит ложно-зелёные гейты (grep который пройдёт всегда)?
- **F. Invariants** — не нарушены CLAUDE.md инварианты?
- **G. Missing edges** — пропущенные edge-кейсы.

HIGH-находка = блокер W1 с `file:line` + готовой правкой. Окупается — ловит false-green ДО запуска. Techlead synthesis в Slack-стиле, patches до W1.

Внешний аудитор на Opus может отдавать 529 — re-run на Sonnet. Перед аудитом task-файлы держать на уровне signature + DoD (тело пишет subagent после) — прибитое гвоздями тело, которое аудитор попросит переделать = двойная работа. Исключение — фикс-точки (security literal, crypto blob, AEAD/HMAC, SSRF resolver): полное тело сразу.

---

## §24 — Версия и installer integrity

**Версия — lockstep по всем источникам.** Список источников версии (например VERSION + package.json + tauri.conf.json + Cargo.toml `[package]`) обновляется синхронно, один bump = все файлы. Lock-файлы (Cargo.lock, package-lock) — сборкой, не руками. Точный список источников — в CLAUDE.md инвариантах проекта.

**Installer integrity gate:**
- После ЛЮБЫХ fixes на feature-branch — пересборка installer'ов перед closing.
- Timestamp-гейт = сравнение mtime бинаря с **последним CODE-коммитом**, НЕ с version-bump коммитом. Build часто идёт ДО commit'а (бинарь старше bump-коммита) — это не регрессия. Сравнение с version-bump даст ложный «бинарь устарел».
- Tag на installer'ах отстающих от tagged commit = блокер.
- Installer копировать в `distr/` с платформенным префиксом и версией.

---

## §25 — Process-artifact hygiene + post-close

**Два класса артефактов в `.claude/`:**
- **Knowledge** (committed, никогда не удалять): REFERENCE, archive/HISTORY, CLAUDE, ROADMAP, intent-specs, contract'ы.
- **Process** (временное): next-session-*, *-cowork-*, diag-логи закрытых волн, runtime-артефакты, скриншоты. Живут в `.claude/tmp/<epic>/` (gitignored).

**При close:**
- Process-артефакты — физический prune (`git rm` НЕ подметает untracked; удалять физически).
- gitignore — recursive паттерны (`.audits/**/*.log`, не shallow).
- Closing checklist: lessons → REFERENCE (новый L-NNN) → HISTORY запись → ROADMAP перенос в closed → memory update → git rm process → gitignore patterns → `git status` clean.
- Orchestrator может рапортовать «close done» без cleanup — Techlead verify по `git status` (§22).

**Post-close обновления (Techlead, не subagent):**
- REFERENCE.md — новый lesson learned.
- archive/HISTORY.md — запись закрытого эпика.
- ROADMAP.md — перенос в `🏁 Закрытые`, changelog роадмапа.
- `memory_user_edits` — новый baseline, актуальный installer slot, task counter; cap 30 (при переполнении заменить самый локальный/устаревший, не дублировать принципы).
