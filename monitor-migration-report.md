# Отчёт по переносу мониторинга (Monitor) из workspace Codex-opencode_tests

## Резюме

Выделенного runtime-компонента `Monitor` в workspace `Codex-opencode_tests` не существует. Под понятием «монитор» здесь фактически понимается **протокол `TASK_STATUS`** и связанные с ним артефакты: отчёты (`_opencode_reports`), статусные smoke-файлы (`_opencode_yolo_status_smoke`, `_opencode_direct_mcp_smoke`) и конфигурация (`opencode.json`, `config.json`, skill `bounded-executor`). Переносить предлагается не приложение, а **пакет документации/протокола** с сопутствующими spec-снимками внешних зависимостей.

---

## Что здесь является `monitor`

В контексте данного workspace `monitor` — это **протокол обратной связи**, с помощью которого bounded-задача (агент OpenCode) сообщает Codex (головному агенту) о результате выполнения. Формально протокол состоит из:

- Строки `TASK_STATUS: <value>` (COMPLETED, FAILED, PARTIAL, BLOCKED, BLOCKED_PERMISSION, NEEDS_CODEX_DECISION) — обязательный маркер в начале meaningful-ответа.
- Evidence-блоков — кратких bullet points с подтверждением выполненной работы.
- Верификационного чеклиста (файл создан, git write не было, external path не использовался, маркер на месте, секции присутствуют).

Никакого отдельного сервиса, daemon, runtime-наблюдателя или CI-компонента с именем `Monitor` в репозитории нет.

---

## Что это такое сейчас

`TASK_STATUS`-протокол и сопутствующие smoke-артефакты — это **средство ручной координации** между Codex и OpenCode. Когда Codex делегирует подзадачу OpenCode через bounded-executor skill, OpenCode выполняет работу и возвращает ответ, начинающийся с `TASK_STATUS`. Codex читает статус и решает, что делать дальше.

Протокол не автоматизирован:
- Нет парсера `TASK_STATUS` на стороне Codex (проверка делается skill-инструкцией).
- Нет валидации структуры ответа (проверяется human-in-the-loop или bounded-executor).
- Нет логирования или метрик.

Всё, что есть — это текстовые отчёты в `_opencode_reports/` и smoke-файлы, создаваемые в тестовых стадиях (Stage 14–15).

---

## Где это находится и из каких частей состоит

### Артефакты внутри workspace:

| Артефакт | Путь | Роль |
|---|---|---|
| Отчёты | `_opencode_reports/` | Полные handoff-отчёты с протоколом TASK_STATUS |
| YOLO status smoke | `_opencode_yolo_status_smoke/` | Smoke-тест YOLO-режима (без спроса человека) |
| Direct MCP smoke | `_opencode_direct_mcp_smoke/` | Smoke-тест прямой записи через MCP |
| Основной конфиг | `opencode.json` | Настройка провайдера (Ollama local) |
| Extended конфиг | `config.json` | Провайдер Ollama, permission `*: allow` |
| Git sandbox | `_opencode_git_sandbox/` | Тестовый локальный git-репозиторий |
| GitHub test guide | `opencode-github-hands-test/` | Набор тестов GitHub MCP через Docker |

### Внешние зависимости:

| Зависимость | Путь/Локация | Тип |
|---|---|---|
| Bounded-executor skill | `C:\Users\andre\.config\opencode\skills\bounded-executor\SKILL.md` | Skill-инструкция OpenCode |
| OpenCode CLI | `opencode` (system PATH) | CLI-инструмент |
| Локальная модель Ollama | `http://localhost:11434/v1`, модель `gemma4:e2b-it-q8_0` | LLM runtime |
| Docker (для GitHub MCP) | Docker Desktop | Контейнеризация |
| Shell | Windows PowerShell 5.1 | Исполнение команд |

### Необязательные части:

- Git sandbox (`_opencode_git_sandbox/`) — используется только для тестов Git-безопасности.
- GitHub test guide (`opencode-github-hands-test/`) — экспериментальный набор для GitHub MCP.
- `README.md` и `work-note.md` в sandbox — неприкосновенные по условию.

---

## От чего зависит

Протокол `TASK_STATUS` зависит от:

1. **Bounded-executor skill** — именно skill предписывает формат `TASK_STATUS` и выполняет (или не выполняет) его валидацию.
2. **OpenCode CLI** — агент (OpenCode) выполняет команды в workspace и возвращает `TASK_STATUS`.
3. **Локальной модели Ollama** — OpenCode использует локальный LLM через OpenAI-совместимый API.
4. **Permission policy** — `config.json` разрешает `*: allow`, что важно для bounded-задач (иначе — BLOCKED_PERMISSION).
5. **PowerShell 5.1** — все команды исполняются в Windows PowerShell.
6. **Git** — для проверки git-безопасности (read-only status).

---

## Что переносить вместе

Ядро мониторинга нужно переносить как **набор документов и спецификаций**, а не как исполняемый код. В целевой репозиторий должны попасть:

1. **Спецификация протокола `TASK_STATUS`** — описание формата, допустимых значений, правил верификации.
2. **Примеры отчётов** — лучшие практики из `_opencode_reports/`.
3. **Снимок (snapshot) bounded-executor skill** — файл `SKILL.md` или spec-эквивалент, фиксирующий текущие правила.
4. **Шаблон конфигурации** — минимальный `opencode.json` + `config.json` (или spec, какие поля обязательны).
5. **Инструкция по настройке** — как развернуть OpenCode + Ollama + skill для работы протокола.

---

## Что можно не переносить

1. **`_opencode_git_sandbox/`** — специфичен для тестов Git-безопасности; если целевой репозиторий не про Git-тесты, sandbox не нужен.
2. **`_opencode_yolo_status_smoke/`** и **`_opencode_direct_mcp_smoke/`** — тестовые стадии, не часть ядра протокола.
3. **`opencode-github-hands-test/`** — отдельный экспериментальный проект.
4. **README.md, work-note.md** — неприкосновенны.
5. **Docker-обёртка GitHub MCP** — если переносится не GitHub-функциональность, Docker не нужен.
6. **Windows-specific launcher/скрипты** — целевой репозиторий может быть кроссплатформенным; скрипты PowerShell можно заменить.
7. **Локальная модель (Ollama)** — это runtime-зависимость среды, не артефакт репозитория. В документации указывается ссылка на настройку Ollama, но сама модель не переносится.

---

## Риски и точки отказа

| Риск | Описание | Уровень |
|---|---|---|
| **TASK_STATUS не найден** | Skill строго требует `TASK_STATUS` первой строкой. Если парсер изменится — протокол сломается. | Высокий |
| **Отсутствие bounded-executor** | Если skill не установлен в целевой OpenCode — протокол не будет принудительно соблюдаться. | Высокий |
| **Permission policy** | Если в целевой конфигурации `permission: {"*": "deny"}`, bounded-задачи будут падать с BLOCKED_PERMISSION. | Средний |
| **Версия OpenCode** | Протокол тестировался на конкретной версии OpenCode. На другой версии поведение может отличаться. | Средний |
| **Платформа** | Весь протокол завязан на Windows PowerShell. На Linux/macOS потребуется адаптация shell-команд. | Средний |
| **Ollama model** | Если в целевой среде другая модель или отсутствует Ollama, поведение агента может измениться. | Низкий |
| **Git safety** | Если целевой репозиторий не требует Git-безопасности, часть протокола (проверка git status) избыточна. | Низкий |

---

## Пошаговый безопасный план переноса

### Шаг 1: Создать целевой репозиторий
- Инициализировать новый репозиторий (например, `opencode-monitor-spec`).
- Добавить `README.md` с кратким описанием назначения.

### Шаг 2: Spec-пакет
1. **Спецификация протокола** (`protocol/task-status-spec.md`):
   - Формат: `TASK_STATUS: <value>`.
   - Допустимые значения: COMPLETED, FAILED, PARTIAL, BLOCKED, BLOCKED_PERMISSION, NEEDS_CODEX_DECISION.
   - Правила: первая строка meaningful-ответа, без лишнего текста перед ней.
   - Верификация: файл создан, git write не было, external path не использовался, маркер на месте.
2. **Снимок bounded-executor skill** (`skill-snapshot/bounded-executor-skill.md`):
   - Копия актуального `SKILL.md` (на момент переноса).
   - Примечание: это snapshot, а не активный skill — при изменении оригинального skill snapshot устаревает.
3. **Шаблон конфигурации** (`config/opencode.json`, `config/config.json`):
   - `opencode.json`: провайдер local/Ollama, baseURL.
   - `config.json`: permission `*: allow`, модель Ollama.
4. **Документация по настройке** (`docs/setup-guide.md`):
   - Установка OpenCode CLI.
   - Настройка Ollama и модели.
   - Установка bounded-executor skill.
   - Проверка работоспособности: smoke-тест.

### Шаг 3: Примеры отчётов
- Перенести 1–2 ключевых отчёта из `_opencode_reports/` как примеры (`examples/stage15s-handoff-report.md`).

### Шаг 4: Smoke-тесты
- Создать минимальный smoke-тест (`tests/smoke-task-status.md`) с инструкцией:
  - Запустить bounded-задачу.
  - Проверить, что ответ начинается с `TASK_STATUS: COMPLETED`.
  - Проверить, что файл-результат создан.
  - Проверить git status (read-only).

### Шаг 5: CI-проверка (опционально)
- Добавить GitHub Actions workflow, который автоматически проверяет, что все `.md` файлы в `protocol/` и `examples/` содержат валидный `TASK_STATUS` (на уровне форматирования — grep-проверка).

### Шаг 6: Подключение
- В bounded-executor skill или opencode.json целевого проекта добавить ссылку на spec-репозиторий: `"monitor_spec": "https://github.com/.../opencode-monitor-spec"`.

---

## Как проверить, что после переноса ничего не сломалось

### Проверка 1: Smoke-тест bounded-задачи
```
TASK_STATUS: COMPLETED
- Monitor spec протокола установлен
- Пример отчёта читается
- Git status: clean
- Permission policy: allow
```

### Проверка 2: Валидация spec-файлов
- Убедиться, что `protocol/task-status-spec.md` содержит все обязательные разделы.
- Убедиться, что `skill-snapshot/bounded-executor-skill.md` соответствует оригинальному `SKILL.md` (diff-сравнение).

### Проверка 3: Git safety
- Выполнить `git status` — не должно быть staged/modified файлов.
- Убедиться, что `git add/commit/push/merge` не выполнялись во время переноса.

### Проверка 4: Внешние зависимости
- Убедиться, что `opencode.json` и `config.json` работают с локальным Ollama.
- Убедиться, что bounded-executor skill загружается: `opencode skill list`.

### Проверка 5: Чтение файла обратно
- Прочитать `monitor-migration-report.md` (этот файл) — убедиться, что он не пуст и все секции присутствуют.

---

## Итоговые рекомендации

1. **Не переносить как приложение.** В целевом репозитории нет runtime-кода — только документация, spec и snapshot. Это снижает требования к поддержке.
2. **Внешние зависимости — spec-снимки.** Bounded-executor skill и конфигурация OpenCode/Ollama копируются как spec-файлы рядом, а не как активные модули.
3. **Замена на локальный spec-файл.** Если в целевом проекте уже есть своя конфигурация, snapshot-файлы заменяются коротким spec-документом с перекрёстными ссылками.
4. **Windows-specific — необязательно.** Весь протокол тестировался на Windows PowerShell, но работа с `TASK_STATUS` не зависит от платформы. Shell-команды (если нужны) адаптируются под целевую ОС.
5. **Git sandbox — не переносить.** Sandbox нужен только для тестов Git-безопасности, которые не входят в spec-пакет.
6. **Проверять версию OpenCode.** Перед использованием spec в целевом проекте убедиться, что версия OpenCode совместима с bounded-executor skill.
7. **Держать протокол живым.** Если bounded-executor skill изменится (например, `TASK_STATUS` станет необязательным), spec-пакет нужно обновлять.

---

## Приложение: ссылки на текущие артефакты

- Отчёты: `D:\Codex+opencode_new\Codex-opencode_tests\_opencode_reports\`
- YOLO smoke: `D:\Codex+opencode_new\Codex-opencode_tests\_opencode_yolo_status_smoke\stage15m-yolo-status-smoke.md`
- Direct MCP smoke: `D:\Codex+opencode_new\Codex-opencode_tests\_opencode_direct_mcp_smoke\`
- Основной конфиг: `D:\Codex+opencode_new\Codex-opencode_tests\opencode.json`
- Extended конфиг: `D:\Codex+opencode_new\Codex-opencode_tests\config.json`
- Bounded-executor skill: `C:\Users\andre\.config\opencode\skills\bounded-executor\SKILL.md`
- Git sandbox: `D:\Codex+opencode_new\Codex-opencode_tests\_opencode_git_sandbox\`
- GitHub test guide: `D:\Codex+opencode_new\Codex-opencode_tests\opencode-github-hands-test\`

---

MARKER_TASKSTATUS_MIGRATION_REPORT_001
