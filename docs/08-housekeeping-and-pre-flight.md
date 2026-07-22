# 08 — Housekeeping and Pre-flight

Cleanup-эпики между фичами и pre-flight чек перед каждым новым крупным эпиком. Невидимая часть workflow которая критична для долгого проекта.

## Зачем нужен housekeeping

Без регулярного cleanup'а:
- В корне репо накапливаются ad-hoc лог-файлы и hotfix скрипты.
- `target/release/` (Rust) и `dist/` (Node) занимают гигабайты на диске.
- Старые installer'ы в `distr/` забивают backup'ы.
- `.claude/fitch/` после 30+ эпиков превращается в свалку.
- `git worktree` от прошлых параллельных работ остаются locked.
- `docs/CLAUDE.md` файлы по всему дереву устаревают (ссылки на удалённые модули, старые версии).
- Closed эпики в `.claude/tasks/` мешают видеть active.

После 6 месяцев работы без housekeeping'а — repo становится unnavigable, agent'ы тратят токены на разбор cruft'а.

## Housekeeping эпики — структура

Pattern: `chore/<epic-id>.0-housekeeping`. Делается между фичами как pre-flight перед следующим крупным эпиком.

### Типичный scope

1. **Disk cleanup:**
   - Удаление старых installer'ов (оставить последние 3-4 в `distr/`).
   - Удаление E2E логов (`.audits/*.log` старее 14 дней).
   - Удаление `target/`, `dist/`, `node_modules/` если есть в worktrees.
   - Удаление 12+ git worktrees.

2. **Archive moves:**
   - Closed эпики `.claude/tasks/<epic-id>/` → `.claude/tasks/done/<epic-id>/`.
   - Старые fitch'и `.claude/fitch/<epic-id>/` → `.claude/fitch/archive/<bracket>/` (например `pre-prj-040`, `prj-040..050`).
   - EPIC-STATUS файлы → `.claude/archive/`.
   - Research notes → `research/done/<epic-id>/`.

3. **Process-artifact prune (см. WORKFLOW §25):**
   - `.claude/tmp/<epic-id>/` закрытых эпиков (next-session-*, *-cowork-*, diag-логи, скриншоты) — **физический** prune. `git rm` НЕ подметает untracked — удалять физически.
   - gitignore — recursive паттерны (`.audits/**/*.log`, не shallow).
   - **Knowledge не трогать:** REFERENCE / HISTORY / CLAUDE / ROADMAP / intent-specs / contract'ы — committed, никогда не удалять.
   - `git status` clean после prune — оркестратор может рапортовать «close done» без cleanup, Techlead verify по `git status`.

4. **Docs refresh:**
   - Обновить корневой `CLAUDE.md` если устарел (проверка через `Filesystem:get_file_info` mtime).
   - Обновить per-module `CLAUDE.md` файлы (проверить ссылки на существующие модули).
   - Обновить `WORKFLOW.md` если паттерны эволюционировали.
   - Обновить `REFERENCE.md` с новыми lessons learned.
   - ROADMAP — текущий status, baseline metrics, lineage installer'ов.

5. **Memory refresh:**
   - `memory_user_edits` пересмотр — что устарело, что добавить.
   - Recent_updates секция в memories — чистка старых записей.

### Типичная длительность

1-3 часа wall-clock. Один или два коммита (cleanup commit + docs sync commit). Без code изменений — `audit.js --baseline` после показывает diff = 0.

### Когда делать

- **Между крупными эпиками** (Medium / Large) — pre-flight для следующего.
- **Раз в 4-6 эпиков** даже без специфичного триггера.
- **Перед sharing repo** новому разработчику.
- **После большой series регрессий** — housekeeping помогает с context'ом.

## Pre-flight чек перед эпиком

Отдельная активность — перед стартом конкретного крупного эпика проверка готовности. 30 минут — пара часов работы.

### Чек-лист

1. **Git clean state:**
   - `git status` чистый.
   - `git branch --no-merged main` пустой кроме self (или ожидаемых).
   - Нет stash'ей которые могут конфликтовать.

2. **Базовые метрики:**
   - Текущий passing/failing/skip — записать в ROADMAP как baseline для эпика.
   - Текущий installer slot — из ROADMAP.
   - Текущая версия — из VERSION или package.json.

3. **Зависимости от других эпиков:**
   - Если эпик использует артефакты другого эпика (например crypto layer из PRJ-056.1b) — этот эпик должен быть merged в main.
   - Verify через `git log --all --grep "<EPIC-ID>"` что merge commit есть.

4. **File isolation предварительный:**
   - Grep по task-файлам что нет конфликтов внутри эпика (см. `06-file-isolation.md`).
   - Проверка что parallel эпики не трогают те же файлы (если есть параллельная work).

5. **Pre-flight измерения (опционально, для refactor эпиков):**
   - `git log --stat --since=180days desktop/src/` → топ-10 по LOC change.
   - CHANGELOG grep по фразам типа `RUN-`, `ENV-`, `POLISH-` — какие компоненты дают >50% багов.
   - React Profiler trace в Runner (или эквивалент) — где медленные re-render'ы.
   - Без этих данных subjective metrics («13 useState много» / «1500 строк много») — theatre. Skeptic-rule из audit'а.

6. **Mockups / external dependencies:**
   - Если эпик с UI — Chief approved mockup (или явно «делай минимально, без редизайна»).
   - Если эпик с external API — verified что API endpoint stable, no deprecation на ближайшее время.

7. **Audit (если scope требует):**
   - Перед Medium / Large эпиком — audit fitch'а перед W1.
   - Audit-fix pass применён.

## Примеры housekeeping-эпиков

### Кейс 1 — pre-flight housekeeping перед PRJ-046.1

Scope:
- Docs refresh: 9 CLAUDE.md обновлены, 2 новых созданы (`desktop/src/core/`, `desktop/e2e/`). Корневой переписан как redirect + карта.
- Cleanup: 3 hotfix логов в корне, legacy `plan.md`, 85 старых E2E логов (оставлены 6 актуальных), пустая директория `prj-042-w2-stabilization`, 12 git worktrees (`git worktree remove`).
- Memory edit: карта проекта.

Один коммит cleanup, один — docs sync. Длительность ~2 часа.

### Кейс 2 — disk cleanup перед PRJ-052

Scope:
- ~170 MB освобождено: `distr/old`, installer'ы старых слотов, 12 E2E логов PRJ-04*, 2 git worktrees.
- Archive moves: PRJ-037-EPIC-STATUS + 4 CHECKPOINT → `.claude/archive/`, `research/<topic>` → `research/done/PRJ-050`.
- Docs refresh: 5 CLAUDE.md обновлены до текущего installer + baseline metrics.

Один коммит. Длительность ~1.5 часа.

### Кейс 3 — fitch directory restructure перед PRJ-053

Scope:
- 35 pre-проектных fitch'ей → `archive/pre-project/`.
- 23 закрытых PRJ-NNN → `archive/closed/`.
- Активные fitch'и на верхнем уровне `fitch/`.
- 10 git worktrees удалены (1 locked — текущая сессия).
- 2 task-директории PRJ-052 архивированы в `done/`.

Длительность ~2 часа. После — fitch директория навигируема.

## Анти-паттерны

### 1. Housekeeping откладывается «потом сделаем»

Полгода без cleanup'а → repo unmanageable, новый агент тратит первые 30 минут разбора cruft'а. **Делать раз в 4-6 эпиков обязательно.**

### 2. Cleanup делается во время feature эпика «заодно»

Subagent в feature task'е удаляет старый файл «потому что не нужен». Через две недели — кто-то жалуется что файл нужен был. **Cleanup — отдельный chore эпик, не inline.**

### 3. Pre-flight пропускается для «известных» эпиков

«Эта серия уже отработана, без pre-flight». Через два дня — обнаруживается что зависимый эпик не merged, или git state не чистый, или mockup не approved. **Pre-flight — несколько минут, экономит часы.**

### 4. Pre-flight measurements не делаются для refactor эпиков

«13 useState много, явно нужно useReducer». Без profiler trace — theatre. Refactor дальше делает много шума, в итоге performance не меняется. **Skeptic-rule: без данных не оптимизируем.**

### 5. Docs refresh забывается

ROADMAP последнее обновление два эпика назад, ссылается на старый installer. Новый агент думает что текущая версия — старая, делает план опираясь на устаревшее. **Docs refresh — обязательная часть финализации каждого эпика, плюс отдельный housekeeping раз в 4-6 эпиков.**

### 6. `memory_user_edits` забывается

После закрытия эпика memories не обновляются. Следующая сессия думает что текущий task counter — N, реально N+5. Конфликты ID, путаница. **`memory_user_edits` обновляется в рамках финализации эпика.**

### 7. `git worktree` накапливаются

Параллельные сессии оставляют locked worktrees. После 10+ — их `git worktree list` занимает страницу, путаница где работает кто. **`git worktree remove` устаревших — часть pre-flight'а.**

## Чек-лист housekeeping эпика

- [ ] Branch `chore/<epic-id>.0-housekeeping` от чистого main.
- [ ] Disk cleanup сделан, освобождённое место зафиксировано.
- [ ] Archive moves сделаны, structure после — навигируема.
- [ ] Docs refresh сделан, mtime обновлён, ссылки проверены.
- [ ] Memory pruning / new entries добавлены.
- [ ] `audit.js --baseline` после = ±0 от before (код не правили).
- [ ] Один или два коммита (cleanup commit + docs sync), conventional message.
- [ ] Merge → main, branch deleted.
- [ ] ROADMAP запись с housekeeping changelog'ом.

## Чек-лист pre-flight перед эпиком

- [ ] Git state clean.
- [ ] Baseline metrics зафиксированы.
- [ ] Зависимости merged.
- [ ] File isolation проверен grep'ом.
- [ ] Pre-flight измерения сделаны (если refactor).
- [ ] Mockup approved (если UI).
- [ ] Audit / audit-fix pass сделан (если Medium / Large).
- [ ] Chief дал OK на старт.
