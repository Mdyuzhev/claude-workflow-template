# 02 — Epic Lifecycle

Полный цикл эпика от появления идеи до архива. Параллельная читка `WORKFLOW.md §2` (там краткий flow), здесь — детальный walkthrough с примерами.

## Шаг 0 — Идея

Эпик возникает одним из трёх способов:

1. **Chief даёт задачу** в чат. Может быть конкретно («хочу страницу настроек security») или абстрактно («хочу чтобы мы сделали наши настройки строже»).
2. **Audit обнаружил блокеры** — несколько finding'ов которые имеет смысл закрыть одним эпиком.
3. **Backlog pickup** — отложенные пункты из закрытых эпиков созрели для работы.

Для (1) и (3) Techlead создаёт **brief** — короткий документ что предлагается делать. Для (2) audit-отчёт сам уже содержит план.

## Шаг 1 — Brief

`brief.md` лежит в `.claude/fitch/<epic-id>/brief.md`. Структура:

- **Контекст** — почему эпик нужен сейчас (1-2 абзаца)
- **Scope** — что входит, гранулярно по блокам если эпик крупный
- **Reject list** — что Chief / Techlead отклонили из первоначальной идеи и почему
- **Variations** — варианты объёма (Slim / Full) с estimate
- **Что НЕ в эпике** — куда уходят deferred пункты
- **File isolation предварительный** — какие файлы трогаются, есть ли пересечения с активными эпиками
- **Pre-flight** — что нужно проверить перед /fitch

Brief обычно подходит чтобы Chief одобрил scope перед тратой времени на полный fitch.

## Шаг 2 — Audit (опционально)

Для крупных эпиков (Medium / Large по `audit-report.md`) — multi-role audit перед fitch'ем. Audit смотрит существующий код / fitch других людей / brief на блокеры до того как мы начнём писать task-файлы.

Triangulation rule: 3+ источника = твёрдый блокер. См. `04-audits-triada.md`.

Findings которые признаны блокерами идут в **audit-fix pass** — серия мини-волн (MW1 / MW2 / …) которая правит fitch / task-файлы / контракт перед W1 эпика. Это специфичный case когда «эпик» начинается с правки своих же артефактов.

## Шаг 3 — Fitch

Полный цикл проработки — analyst → designer → tech_lead → toster → devops. Если есть skill `/fitch` — запускается через него (см. `07-skills-overview.md`). Без skill — Techlead делает то же самое вручную итеративно.

Артефакты после fitch'а:
- `.claude/fitch/<epic-id>/contract.json` — структурированный JSON со всеми ролями
- `.claude/fitch/<epic-id>/FITCH-PLAN.md` — итоговый план для Chief'а (синтез контракта)

См. `.claude/templates/fitch-contract.json` для структуры.

## Шаг 4 — PLAN.md и task-файлы

После approval'а fitch'а — Techlead создаёт:
- `.claude/tasks/<epic-id>/PLAN-<epic-id>.md` — описание волн с parallel groups, file isolation, gate'ами, confirmation loops
- `.claude/tasks/<epic-id>/task-NN-*.md` — task-файлы под subagent'ов (60-200 строк каждый)

Принципиальные моменты:
- Перед task-файлом — **read реального кода** через Filesystem MCP. Никаких phantom-сигнатур.
- File isolation проверяется grep'ом по task-файлам перед стартом эпика.
- App entry / shared exposes (E2E hooks типа `__VITE_E2E__`) — централизуются в один task в W1, остальные task'и только используют.

## Шаг 5 — Wave execution

Orchestrator запускается в Claude Code, читает PLAN.md. Techlead отправляет первую wave-command по шаблону `orchestrator-wave-command.md`.

Внутри волны — parallel или sequential subagent'ы. Каждый subagent выполняет один task-файл, проверяет DoD, коммитит. Orchestrator собирает результаты, делает wave gate (полный automated test run).

Gate состоит из:
1. Build green (npm run build / cargo build)
2. Test green (полный automated прогон)
3. Failure IDs diff пустой (по сравнению с baseline предыдущей волны)
4. Confirmation от Chief'а

Если gate red — diag-then-fix микро-волна перед следующей основной (см. `05-diag-then-fix.md`).

## Шаг 6 — Confirmation loop

Orchestrator пишет отчёт в формате §19:

```
W<N> closed
- closed: <task-list>
- gate: <smoke metrics>, failure IDs diff = <empty | listed>
- commits: <hash-list> + comment
- next: W<N+1> по PLAN

STOP. Жду explicit confirmation перед W<N+1>.
```

Techlead в main chat синтезирует это для Chief'а. **Перед синтезом — verify по диску (WORKFLOW §22):** критичные заявления оркестратора (smoke-метрики — реальный лог а не рапорт, hash, существование файлов, timestamps) Techlead проверяет сам через Filesystem, не на слово. Поймал расхождение — «поправка: реально X» и дальше. Chief даёт OK или отправляет в diag.

Без OK — следующая волна не стартует. Это критично: если orchestrator самовольно стартанёт W2, регрессии в W2 будут смешаны с unfix'ами W1, диагностика становится невозможной.

## Шаг 7 — Финализация

Последняя волна — финализация (обычно `task-<final>-finalization.md`). Что в ней:
1. `bump-version` script (или эквивалент) — обновить версию в VERSION / package.json / Cargo.toml.
2. CHANGELOG секция [<version>] с Added / Tests / Known issues / Deferred.
3. Build installer через wrapper script (`-Prefix <slot>`).
4. `audit.js` без `--baseline` — diff vs pre-эпик baseline.
5. Если diff пустой / known acceptable — proceed. Если есть unexpected — эскалация.
6. `git checkout main && git merge --no-ff <branch>`.
7. `git push origin main && git branch -d <branch>`.
8. `audit.js --baseline` — фиксация нового baseline.
9. `mkdir done/<epic-id> && mv tasks/<epic-id>/* done/<epic-id>/`.

**Перед merge (step 6) — обязательная confirmation от Chief'а.** Это последняя точка возврата.

## Шаг 8 — ROADMAP / Memory updates

После merge:
- Эпик переезжает в секцию `🏁 Закрытые эпики` в ROADMAP.md.
- В `Changelog роадмапа` (внизу ROADMAP) — запись с датой / ID / one-line summary / installer.
- Если эпик дал new lesson learned (например «не возвращаться к pattern X») — добавить в `REFERENCE.md`.
- `memory_user_edits` обновляется: новый baseline metrics, актуальный installer slot, текущий task counter.

Все эти обновления — **Techlead**, не subagent. Subagent'ы code commit'ят, но не трогают meta-документы.

- **Чекпоинт handoff** (`.claude/<handoff>.json`, WORKFLOW §21) — перезаписать целиком через `write_file` с итогом эпика + `next_session_first_steps`. Это финальный снимок состояния для следующей сессии, свежее памяти. Обязателен при close и при сжатии контекста посреди эпика.
- **Verify close по диску** (WORKFLOW §22): `.git/refs/heads/main` = `refs/remotes/origin/main` = `refs/tags/<tag>` — убедиться что merge/push/tag реально на месте, а не только в рапорте оркестратора.

## Длительность эпика — ориентиры

Порядок величин по опыту:
- **Hotfix** (1-2 task'а, без fitch) — 1-3 часа.
- **Small эпик** (5-10 task'ов, lite fitch) — 1-2 дня.
- **Medium эпик** (15-30 task'ов, full fitch + audit) — 3-7 дней.
- **Large эпик** (40-60 task'ов, full fitch + audit + audit-fix pass) — 10-20 дней.

Темп ускоряется по мере calibration: первые 1-2 эпика обычно идут с overhead'ом 30-50%, дальше плавнее.

## Анти-паттерны жизненного цикла

1. **Эпик начинается без brief** — Chief сразу говорит «делай», Techlead начинает писать task-файлы. Через два дня выясняется что Chief хотел другое. Brief — 30 минут, экономит дни.

2. **Audit пропускается «потому что я и так знаю что там»** — потом в W7 всплывает блокер который audit нашёл бы за 10 минут.

3. **Fitch без проверки реального кода** — analyst пишет требования по памяти, designer добавляет компоненты которые конфликтуют с существующими, tech_lead планирует task'и на phantom-сигнатуры. После fitch'а нужен audit-fix pass.

4. **W7 pickup эпик, который не делается** — эпик задумывался «21 bugs fixed», в финале делается 15, 6 уходит в backlog. CHANGELOG пишется как «21 fixed» — враньё. Правильно: явно зафиксировать «15 fixed, 6 deferred → PRJ-NNN.x pickup».

5. **Orchestrator стартует следующую волну без confirmation** — особенно опасно если Chief в потоке и пишет «давай», думая что речь про W2, а orchestrator подразумевает финализацию.

6. **Финализация без manual confirmation merge'а** — installer уже собран, hash в CHANGELOG зафиксирован, Chief на встрече, orchestrator merge'ит сам. Если в installer регрессия — rollback болезненный.

7. **`memory_user_edits` не обновляется** после закрытия эпика — следующая сессия Techlead'а думает что текущий installer всё ещё предыдущий, делает план опираясь на устаревшее состояние.

## Чек-лист перед стартом эпика

- [ ] Brief approved Chief'ом.
- [ ] Audit (если Medium/Large) сделан, audit-fix pass применён.
- [ ] Fitch contract / FITCH-PLAN существуют и approved.
- [ ] PLAN.md написан, file isolation проверена grep'ом.
- [ ] Все task-файлы physically существуют, signatures verified.
- [ ] Branch создан от чистого main.
- [ ] Pre-flight checks пройдены (если есть).
- [ ] Chief дал OK на старт W1.
