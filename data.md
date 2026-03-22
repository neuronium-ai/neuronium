# Data Vault модель для MVP AI-first editor

## Цель

Спроектировать базу для MVP в логике **Data Vault**, но без избыточной сложности. База должна:

* поддерживать локальный MVP;
* не ломать простую разработку на `Node.js + SQLite`;
* хранить историю изменений и действий AI;
* не смешивать бизнес-ключи, состояние UI и временные вычисления;
* позволять потом безболезненно перейти к Postgres и более взрослой аналитике.

Ключевой принцип: **используем Data Vault как дисциплину моделирования ядра метаданных, а не как религию**.

---

## Практический подход для MVP

Для этого продукта не нужно строить огромный «классический enterprise vault». Достаточно 3 уровней:

### 1. Raw Vault

Историзируемые сущности и связи:

* workspace / repo
* branch
* file
* document session / tab
* AI request
* AI suggestion
* user action
* prompt template

### 2. Business Vault lite

Небольшой слой производных представлений:

* current file state
* latest suggestion per file
* unresolved suggestions
* action timeline per file

### 3. Serving / App tables or views

Удобные для backend/UI представления:

* дерево файлов
* активные вкладки
* история AI-действий по документу
* diff queue
* pending accept/reject actions

Для MVP можно держать **Raw Vault физически в таблицах**, а Business Vault и Serving — как **SQL views** и пару materialized-like кешей на стороне backend.

---

## Что считаем бизнес-объектами

Ниже — реальные бизнес-ключи для продукта.

1. **Workspace** — конкретный локальный проект пользователя.
2. **Repository** — git-репозиторий в рамках workspace.
3. **Branch** — git-ветка.
4. **File** — логический файл в репозитории.
5. **Document Session** — конкретная сессия работы пользователя с документом.
6. **AI Request** — запрос на AI-операцию над активным документом.
7. **AI Suggestion** — результат AI как предложенное изменение.
8. **User Action** — решение пользователя: accept / reject / create new file / retry / cancel.
9. **Prompt Template** — шаблон системной команды.

Важно: **Diff не делаем отдельным Hub**. Для MVP это не самостоятельная бизнес-сущность, а атрибут результата AI suggestion.

---

## Минимальный набор Hubs

## HUB_WORKSPACE

**Назначение:** контейнер локального проекта.

**Business key:** `workspace_key`
Рекомендуемо: нормализованный абсолютный путь или UUID, если путь может меняться.

**Поля:**

* `hub_workspace_id` PK
* `workspace_key` UNIQUE
* `load_dts`
* `record_source`

---

## HUB_REPOSITORY

**Назначение:** git-репозиторий.

**Business key:** `repository_key`
Например: `workspace_key + repo_root_path`

**Поля:**

* `hub_repository_id` PK
* `repository_key` UNIQUE
* `load_dts`
* `record_source`

---

## HUB_BRANCH

**Назначение:** git-ветка.

**Business key:** `branch_key`
Например: `repository_key + branch_name`

**Поля:**

* `hub_branch_id` PK
* `branch_key` UNIQUE
* `load_dts`
* `record_source`

---

## HUB_FILE

**Назначение:** логический файл.

**Business key:** `file_key`
Например: `repository_key + relative_file_path`

**Поля:**

* `hub_file_id` PK
* `file_key` UNIQUE
* `load_dts`
* `record_source`

Примечание: если файл переименован, это либо новый business key, либо отдельная tracked relation. Для MVP проще считать rename новым `file_key`, а связь со старым именем вести через action/event.

---

## HUB_DOCUMENT_SESSION

**Назначение:** пользовательская рабочая сессия с документом.

**Business key:** `document_session_key`
UUID на открытие/переоткрытие таба.

**Поля:**

* `hub_document_session_id` PK
* `document_session_key` UNIQUE
* `load_dts`
* `record_source`

---

## HUB_AI_REQUEST

**Назначение:** факт вызова AI-команды.

**Business key:** `ai_request_key`
UUID.

**Поля:**

* `hub_ai_request_id` PK
* `ai_request_key` UNIQUE
* `load_dts`
* `record_source`

---

## HUB_AI_SUGGESTION

**Назначение:** выданное AI-предложение.

**Business key:** `ai_suggestion_key`
UUID.

**Поля:**

* `hub_ai_suggestion_id` PK
* `ai_suggestion_key` UNIQUE
* `load_dts`
* `record_source`

---

## HUB_USER_ACTION

**Назначение:** пользовательское решение по AI-результату.

**Business key:** `user_action_key`
UUID.

**Поля:**

* `hub_user_action_id` PK
* `user_action_key` UNIQUE
* `load_dts`
* `record_source`

---

## HUB_PROMPT_TEMPLATE

**Назначение:** шаблон промпта.

**Business key:** `prompt_template_key`
Например: `template_name + version`

**Поля:**

* `hub_prompt_template_id` PK
* `prompt_template_key` UNIQUE
* `load_dts`
* `record_source`

---

## Минимальный набор Links

Суть: links фиксируют связи бизнес-объектов и почти не несут описательных атрибутов.

## LINK_WORKSPACE_REPOSITORY

* `link_workspace_repository_id` PK
* `hub_workspace_id` FK
* `hub_repository_id` FK
* `load_dts`
* `record_source`

---

## LINK_REPOSITORY_BRANCH

* `link_repository_branch_id` PK
* `hub_repository_id` FK
* `hub_branch_id` FK
* `load_dts`
* `record_source`

---

## LINK_REPOSITORY_FILE

* `link_repository_file_id` PK
* `hub_repository_id` FK
* `hub_file_id` FK
* `load_dts`
* `record_source`

---

## LINK_BRANCH_FILE

Нужен, если важно видеть файл в контексте ветки.

* `link_branch_file_id` PK
* `hub_branch_id` FK
* `hub_file_id` FK
* `load_dts`
* `record_source`

Для MVP можно оставить, потому что AI-операции и коммиты почти всегда branch-aware.

---

## LINK_FILE_DOCUMENT_SESSION

* `link_file_document_session_id` PK
* `hub_file_id` FK
* `hub_document_session_id` FK
* `load_dts`
* `record_source`

---

## LINK_DOCUMENT_SESSION_AI_REQUEST

* `link_document_session_ai_request_id` PK
* `hub_document_session_id` FK
* `hub_ai_request_id` FK
* `load_dts`
* `record_source`

---

## LINK_AI_REQUEST_PROMPT_TEMPLATE

* `link_ai_request_prompt_template_id` PK
* `hub_ai_request_id` FK
* `hub_prompt_template_id` FK
* `load_dts`
* `record_source`

---

## LINK_AI_REQUEST_FILE

Фиксирует, к какому файлу был применён AI-запрос.

* `link_ai_request_file_id` PK
* `hub_ai_request_id` FK
* `hub_file_id` FK
* `load_dts`
* `record_source`

---

## LINK_AI_REQUEST_SUGGESTION

* `link_ai_request_suggestion_id` PK
* `hub_ai_request_id` FK
* `hub_ai_suggestion_id` FK
* `load_dts`
* `record_source`

---

## LINK_SUGGESTION_USER_ACTION

* `link_suggestion_user_action_id` PK
* `hub_ai_suggestion_id` FK
* `hub_user_action_id` FK
* `load_dts`
* `record_source`

---

## Набор Satellites

Именно satellites несут описательные атрибуты и историю изменения состояния.

## SAT_WORKSPACE_ATTR

* `hub_workspace_id` FK
* `load_dts`
* `hashdiff`
* `workspace_name`
* `root_path`
* `is_active`
* `created_at`
* `updated_at`
* `record_source`

---

## SAT_REPOSITORY_ATTR

* `hub_repository_id` FK
* `load_dts`
* `hashdiff`
* `repo_name`
* `repo_root_path`
* `git_remote_origin`
* `default_branch_name`
* `last_scanned_at`
* `record_source`

---

## SAT_BRANCH_ATTR

* `hub_branch_id` FK
* `load_dts`
* `hashdiff`
* `branch_name`
* `head_commit_sha`
* `is_current`
* `last_seen_at`
* `record_source`

---

## SAT_FILE_ATTR

Базовые описательные атрибуты файла.

* `hub_file_id` FK
* `load_dts`
* `hashdiff`
* `relative_path`
* `file_name`
* `extension`
* `mime_type`
* `is_deleted`
* `last_known_size_bytes`
* `last_modified_at`
* `record_source`

---

## SAT_FILE_CONTENT_STATE

История наблюдаемых состояний файла.

* `hub_file_id` FK
* `load_dts`
* `hashdiff`
* `content_sha256`
* `git_blob_sha` NULL
* `content_snapshot_ref` NULL
* `line_count`
* `char_count`
* `encoding`
* `record_source`

Замечание: **не надо тащить весь текст файла в SQLite** для MVP, если source of truth — filesystem + git. Лучше хранить:

* hash,
* ссылку на snapshot,
* при необходимости cached preview.

Полный текст можно читать с диска по path.

---

## SAT_DOCUMENT_SESSION_ATTR

* `hub_document_session_id` FK
* `load_dts`
* `hashdiff`
* `opened_at`
* `closed_at` NULL
* `tab_index`
* `is_active`
* `cursor_position_json`
* `selection_range_json`
* `record_source`

---

## SAT_AI_REQUEST_ATTR

Метаданные AI-вызова.

* `hub_ai_request_id` FK
* `load_dts`
* `hashdiff`
* `command_type`  -- rewrite / summarize / explain / fix / transform
* `user_instruction`
* `model_name`
* `model_provider`
* `temperature`
* `token_budget`
* `request_status` -- queued / running / completed / failed / cancelled
* `started_at`
* `finished_at`
* `error_message` NULL
* `record_source`

---

## SAT_AI_REQUEST_CONTEXT

Контекст, отправленный в модель.

* `hub_ai_request_id` FK
* `load_dts`
* `hashdiff`
* `active_file_path`
* `selected_text_range_json`
* `selected_text_hash`
* `surrounding_context_hash`
* `prompt_rendered_hash`
* `input_tokens_estimate`
* `record_source`

Для MVP достаточно хранить хэши и диапазоны, а не весь промпт целиком. Полный prompt можно логировать опционально в файл-аудит.

---

## SAT_AI_SUGGESTION_ATTR

Главный satellite результата.

* `hub_ai_suggestion_id` FK
* `load_dts`
* `hashdiff`
* `suggestion_status` -- pending / accepted / rejected / exported / superseded
* `output_text_ref`
* `diff_text_ref`
* `summary_text`
* `target_mode` -- inplace / new_file
* `new_file_path` NULL
* `confidence_note` NULL
* `created_at`
* `record_source`

Здесь важно: сам текст результата и diff лучше хранить **как refs** на файл или blob-cache, а не гигантским TEXT в SQLite.

---

## SAT_USER_ACTION_ATTR

* `hub_user_action_id` FK
* `load_dts`
* `hashdiff`
* `action_type` -- accept / reject / export_new_file / close / retry
* `action_reason` NULL
* `acted_at`
* `record_source`

---

## SAT_PROMPT_TEMPLATE_ATTR

* `hub_prompt_template_id` FK
* `load_dts`
* `hashdiff`
* `template_name`
* `template_version`
* `system_prompt_ref`
* `instruction_schema_json`
* `is_active`
* `record_source`

---

## Что НЕ надо тащить в Vault в MVP

Чтобы не перемудрить, это лучше не выделять в самостоятельные vault-сущности на первом этапе:

1. **UI layout state**

   * ширина панелей,
   * collapsed / expanded AI panel,
   * размеры sidebar,
   * theme.

   Это лучше держать в простых app tables типа `ui_preferences`.

2. **Transient editor state**

   * текущий scroll,
   * ephemeral selection,
   * незавершённый ввод.

3. **Filesystem scan cache**
   Это техническая оптимизация, не бизнес-ядро.

4. **Model pricing / telemetry deep logs**
   Можно добавить позже.

---

## Рекомендуемое разделение: Vault vs App tables

## В Vault

То, что имеет бизнес-смысл и историю:

* workspace
* repository
* branch
* file
* session
* ai request
* ai suggestion
* user action
* prompt template

## В обычных app tables

То, что purely operational:

* `ui_preferences`
* `app_settings`
* `local_model_config`
* `filesystem_scan_state`
* `job_queue`
* `dev_logs`

Это нормально. Не всё должно быть Data Vault.

---

## Физическая схема SQLite для MVP

Ниже — pragmatic вариант. Не обязательно делать surrogate keys как hash keys. Для SQLite MVP проще:

* `TEXT` UUID для hub/link id
* `TEXT` business key
* индексы по business key
* timestamps в ISO format

---

## Пример таблиц Hubs

```sql
CREATE TABLE hub_workspace (
  hub_workspace_id TEXT PRIMARY KEY,
  workspace_key TEXT NOT NULL UNIQUE,
  load_dts TEXT NOT NULL,
  record_source TEXT NOT NULL
);

CREATE TABLE hub_repository (
  hub_repository_id TEXT PRIMARY KEY,
  repository_key TEXT NOT NULL UNIQUE,
  load_dts TEXT NOT NULL,
  record_source TEXT NOT NULL
);

CREATE TABLE hub_branch (
  hub_branch_id TEXT PRIMARY KEY,
  branch_key TEXT NOT NULL UNIQUE,
  load_dts TEXT NOT NULL,
  record_source TEXT NOT NULL
);

CREATE TABLE hub_file (
  hub_file_id TEXT PRIMARY KEY,
  file_key TEXT NOT NULL UNIQUE,
  load_dts TEXT NOT NULL,
  record_source TEXT NOT NULL
);

CREATE TABLE hub_document_session (
  hub_document_session_id TEXT PRIMARY KEY,
  document_session_key TEXT NOT NULL UNIQUE,
  load_dts TEXT NOT NULL,
  record_source TEXT NOT NULL
);

CREATE TABLE hub_ai_request (
  hub_ai_request_id TEXT PRIMARY KEY,
  ai_request_key TEXT NOT NULL UNIQUE,
  load_dts TEXT NOT NULL,
  record_source TEXT NOT NULL
);

CREATE TABLE hub_ai_suggestion (
  hub_ai_suggestion_id TEXT PRIMARY KEY,
  ai_suggestion_key TEXT NOT NULL UNIQUE,
  load_dts TEXT NOT NULL,
  record_source TEXT NOT NULL
);

CREATE TABLE hub_user_action (
  hub_user_action_id TEXT PRIMARY KEY,
  user_action_key TEXT NOT NULL UNIQUE,
  load_dts TEXT NOT NULL,
  record_source TEXT NOT NULL
);

CREATE TABLE hub_prompt_template (
  hub_prompt_template_id TEXT PRIMARY KEY,
  prompt_template_key TEXT NOT NULL UNIQUE,
  load_dts TEXT NOT NULL,
  record_source TEXT NOT NULL
);
```

---

## Пример таблиц Links

```sql
CREATE TABLE link_workspace_repository (
  link_workspace_repository_id TEXT PRIMARY KEY,
  hub_workspace_id TEXT NOT NULL,
  hub_repository_id TEXT NOT NULL,
  load_dts TEXT NOT NULL,
  record_source TEXT NOT NULL,
  UNIQUE (hub_workspace_id, hub_repository_id)
);

CREATE TABLE link_repository_branch (
  link_repository_branch_id TEXT PRIMARY KEY,
  hub_repository_id TEXT NOT NULL,
  hub_branch_id TEXT NOT NULL,
  load_dts TEXT NOT NULL,
  record_source TEXT NOT NULL,
  UNIQUE (hub_repository_id, hub_branch_id)
);

CREATE TABLE link_repository_file (
  link_repository_file_id TEXT PRIMARY KEY,
  hub_repository_id TEXT NOT NULL,
  hub_file_id TEXT NOT NULL,
  load_dts TEXT NOT NULL,
  record_source TEXT NOT NULL,
  UNIQUE (hub_repository_id, hub_file_id)
);

CREATE TABLE link_branch_file (
  link_branch_file_id TEXT PRIMARY KEY,
  hub_branch_id TEXT NOT NULL,
  hub_file_id TEXT NOT NULL,
  load_dts TEXT NOT NULL,
  record_source TEXT NOT NULL,
  UNIQUE (hub_branch_id, hub_file_id)
);

CREATE TABLE link_file_document_session (
  link_file_document_session_id TEXT PRIMARY KEY,
  hub_file_id TEXT NOT NULL,
  hub_document_session_id TEXT NOT NULL,
  load_dts TEXT NOT NULL,
  record_source TEXT NOT NULL,
  UNIQUE (hub_file_id, hub_document_session_id)
);

CREATE TABLE link_document_session_ai_request (
  link_document_session_ai_request_id TEXT PRIMARY KEY,
  hub_document_session_id TEXT NOT NULL,
  hub_ai_request_id TEXT NOT NULL,
  load_dts TEXT NOT NULL,
  record_source TEXT NOT NULL,
  UNIQUE (hub_document_session_id, hub_ai_request_id)
);

CREATE TABLE link_ai_request_prompt_template (
  link_ai_request_prompt_template_id TEXT PRIMARY KEY,
  hub_ai_request_id TEXT NOT NULL,
  hub_prompt_template_id TEXT NOT NULL,
  load_dts TEXT NOT NULL,
  record_source TEXT NOT NULL,
  UNIQUE (hub_ai_request_id, hub_prompt_template_id)
);

CREATE TABLE link_ai_request_file (
  link_ai_request_file_id TEXT PRIMARY KEY,
  hub_ai_request_id TEXT NOT NULL,
  hub_file_id TEXT NOT NULL,
  load_dts TEXT NOT NULL,
  record_source TEXT NOT NULL,
  UNIQUE (hub_ai_request_id, hub_file_id)
);

CREATE TABLE link_ai_request_suggestion (
  link_ai_request_suggestion_id TEXT PRIMARY KEY,
  hub_ai_request_id TEXT NOT NULL,
  hub_ai_suggestion_id TEXT NOT NULL,
  load_dts TEXT NOT NULL,
  record_source TEXT NOT NULL,
  UNIQUE (hub_ai_request_id, hub_ai_suggestion_id)
);

CREATE TABLE link_suggestion_user_action (
  link_suggestion_user_action_id TEXT PRIMARY KEY,
  hub_ai_suggestion_id TEXT NOT NULL,
  hub_user_action_id TEXT NOT NULL,
  load_dts TEXT NOT NULL,
  record_source TEXT NOT NULL,
  UNIQUE (hub_ai_suggestion_id, hub_user_action_id)
);
```

---

## Пример Satellites

```sql
CREATE TABLE sat_file_attr (
  hub_file_id TEXT NOT NULL,
  load_dts TEXT NOT NULL,
  hashdiff TEXT NOT NULL,
  relative_path TEXT NOT NULL,
  file_name TEXT NOT NULL,
  extension TEXT,
  mime_type TEXT,
  is_deleted INTEGER NOT NULL DEFAULT 0,
  last_known_size_bytes INTEGER,
  last_modified_at TEXT,
  record_source TEXT NOT NULL,
  PRIMARY KEY (hub_file_id, load_dts)
);

CREATE TABLE sat_file_content_state (
  hub_file_id TEXT NOT NULL,
  load_dts TEXT NOT NULL,
  hashdiff TEXT NOT NULL,
  content_sha256 TEXT NOT NULL,
  git_blob_sha TEXT,
  content_snapshot_ref TEXT,
  line_count INTEGER,
  char_count INTEGER,
  encoding TEXT,
  record_source TEXT NOT NULL,
  PRIMARY KEY (hub_file_id, load_dts)
);

CREATE TABLE sat_ai_request_attr (
  hub_ai_request_id TEXT NOT NULL,
  load_dts TEXT NOT NULL,
  hashdiff TEXT NOT NULL,
  command_type TEXT NOT NULL,
  user_instruction TEXT,
  model_name TEXT,
  model_provider TEXT,
  temperature REAL,
  token_budget INTEGER,
  request_status TEXT NOT NULL,
  started_at TEXT,
  finished_at TEXT,
  error_message TEXT,
  record_source TEXT NOT NULL,
  PRIMARY KEY (hub_ai_request_id, load_dts)
);

CREATE TABLE sat_ai_request_context (
  hub_ai_request_id TEXT NOT NULL,
  load_dts TEXT NOT NULL,
  hashdiff TEXT NOT NULL,
  active_file_path TEXT,
  selected_text_range_json TEXT,
  selected_text_hash TEXT,
  surrounding_context_hash TEXT,
  prompt_rendered_hash TEXT,
  input_tokens_estimate INTEGER,
  record_source TEXT NOT NULL,
  PRIMARY KEY (hub_ai_request_id, load_dts)
);

CREATE TABLE sat_ai_suggestion_attr (
  hub_ai_suggestion_id TEXT NOT NULL,
  load_dts TEXT NOT NULL,
  hashdiff TEXT NOT NULL,
  suggestion_status TEXT NOT NULL,
  output_text_ref TEXT,
  diff_text_ref TEXT,
  summary_text TEXT,
  target_mode TEXT NOT NULL,
  new_file_path TEXT,
  confidence_note TEXT,
  created_at TEXT NOT NULL,
  record_source TEXT NOT NULL,
  PRIMARY KEY (hub_ai_suggestion_id, load_dts)
);

CREATE TABLE sat_user_action_attr (
  hub_user_action_id TEXT NOT NULL,
  load_dts TEXT NOT NULL,
  hashdiff TEXT NOT NULL,
  action_type TEXT NOT NULL,
  action_reason TEXT,
  acted_at TEXT NOT NULL,
  record_source TEXT NOT NULL,
  PRIMARY KEY (hub_user_action_id, load_dts)
);
```

---

## Hashdiff: как не переусложнить

Для MVP достаточно правила:

* на каждый satellite считаем `hashdiff` по набору бизнес-атрибутов;
* новую запись в satellite вставляем только если `hashdiff` изменился;
* end-date не обязателен, текущую версию получаем как `MAX(load_dts)`.

Это сильно упрощает реализацию в SQLite.

---

## Как это работает в ключевых сценариях

## Сценарий 1. Пользователь открыл проект

Что создаётся:

* `hub_workspace`
* `sat_workspace_attr`
* `hub_repository`
* `link_workspace_repository`
* `sat_repository_attr`
* `hub_branch`
* `link_repository_branch`
* `sat_branch_attr`

---

## Сценарий 2. Backend просканировал файлы

Для каждого файла:

* `hub_file`
* `link_repository_file`
* `link_branch_file`
* `sat_file_attr`
* `sat_file_content_state`

---

## Сценарий 3. Пользователь вызывает AI-команду над активным документом

Создаётся:

* `hub_document_session` если нет активной сессии
* `link_file_document_session`
* `hub_ai_request`
* `link_document_session_ai_request`
* `link_ai_request_file`
* `sat_ai_request_attr` со статусом `queued`/`running`
* `sat_ai_request_context`

После ответа модели:

* `hub_ai_suggestion`
* `link_ai_request_suggestion`
* `sat_ai_request_attr` новая версия со статусом `completed`
* `sat_ai_suggestion_attr` со статусом `pending`

---

## Сценарий 4. Пользователь нажал Accept

Создаётся:

* `hub_user_action`
* `sat_user_action_attr` с `action_type = accept`
* `link_suggestion_user_action`
* новая запись `sat_ai_suggestion_attr` со статусом `accepted`
* новая запись `sat_file_content_state` с новым hash файла

---

## Сценарий 5. Пользователь нажал Reject

Создаётся:

* `hub_user_action`
* `sat_user_action_attr` с `action_type = reject`
* `link_suggestion_user_action`
* новая запись `sat_ai_suggestion_attr` со статусом `rejected`

Состояние файла не меняется.

---

## Сценарий 6. Пользователь выбрал “Create new file”

Создаётся:

* `hub_user_action`
* `sat_user_action_attr` с `action_type = export_new_file`
* `link_suggestion_user_action`
* новый `hub_file`
* `link_repository_file`
* `link_branch_file`
* `sat_file_attr`
* `sat_file_content_state`
* новая запись `sat_ai_suggestion_attr` со статусом `exported`

---

## Business Vault lite views

Ниже — действительно полезные представления.

## V_CURRENT_FILE_STATE

Текущая версия файла.

Показывает:

* file id
* path
* latest content hash
* deleted flag
* modified_at

---

## V_ACTIVE_BRANCH_FILES

Файлы активной ветки для дерева проекта.

---

## V_LATEST_AI_SUGGESTION_PER_FILE

Последнее AI-предложение на файл.

---

## V_PENDING_SUGGESTIONS

Все предложения, которые ждут accept/reject.

---

## V_FILE_ACTION_TIMELINE

Единая timeline по файлу:

* scan
* ai request
* suggestion created
* accepted
* rejected
* exported

---

## V_REQUEST_EXECUTION_LOG

Поиск по AI-вызовам:

* file
* command type
* model
* status
* duration
* error

---

## Примеры SQL views

```sql
CREATE VIEW v_pending_suggestions AS
SELECT
  s.hub_ai_suggestion_id,
  f.hub_file_id,
  fa.relative_path,
  sa.suggestion_status,
  sa.summary_text,
  sa.diff_text_ref,
  sa.output_text_ref,
  sa.created_at
FROM link_ai_request_suggestion lrs
JOIN link_ai_request_file lrf
  ON lrf.hub_ai_request_id = lrs.hub_ai_request_id
JOIN hub_file f
  ON f.hub_file_id = lrf.hub_file_id
JOIN sat_file_attr fa
  ON fa.hub_file_id = f.hub_file_id
JOIN sat_ai_suggestion_attr sa
  ON sa.hub_ai_suggestion_id = lrs.hub_ai_suggestion_id
WHERE fa.load_dts = (
  SELECT MAX(fa2.load_dts)
  FROM sat_file_attr fa2
  WHERE fa2.hub_file_id = fa.hub_file_id
)
AND sa.load_dts = (
  SELECT MAX(sa2.load_dts)
  FROM sat_ai_suggestion_attr sa2
  WHERE sa2.hub_ai_suggestion_id = sa.hub_ai_suggestion_id
)
AND sa.suggestion_status = 'pending';
```

---

## App-layer tables, которые нужны рядом с Vault

Это важно, потому что чистый Vault не должен тащить на себе весь runtime.

## ui_preferences

* `user_scope`
* `left_panel_width`
* `right_panel_width`
* `ai_panel_mode` -- docked / floating / collapsed
* `theme`
* `updated_at`

## app_settings

* `key`
* `value_json`
* `updated_at`

## filesystem_scan_state

* `repository_key`
* `last_scan_started_at`
* `last_scan_finished_at`
* `last_scan_status`
* `last_error`

## ai_job_queue

* `job_id`
* `ai_request_key`
* `job_status`
* `scheduled_at`
* `started_at`
* `finished_at`
* `retry_count`

Эти таблицы не надо насильно запихивать в Vault.

---

## Индексы, которые обязательны

Для SQLite MVP обязательно добавить индексы:

* по всем business keys в hubs;
* по FK в links;
* по `(hub_id, load_dts desc)` в satellites;
* по `request_status`, `suggestion_status`, `acted_at`, `created_at` для рабочих запросов UI.

Примеры:

```sql
CREATE INDEX idx_sat_file_attr_latest
  ON sat_file_attr (hub_file_id, load_dts DESC);

CREATE INDEX idx_sat_ai_request_attr_latest
  ON sat_ai_request_attr (hub_ai_request_id, load_dts DESC);

CREATE INDEX idx_sat_ai_suggestion_status
  ON sat_ai_suggestion_attr (suggestion_status, created_at DESC);

CREATE INDEX idx_link_ai_request_file_request
  ON link_ai_request_file (hub_ai_request_id);

CREATE INDEX idx_link_ai_request_file_file
  ON link_ai_request_file (hub_file_id);
```

---

## Правила загрузки данных

Чтобы система была стабильной, стоит зафиксировать простые правила ingestion.

### Rule 1

Сначала hub, потом link, потом satellite.

### Rule 2

Никогда не update-им старую satellite запись; только insert новой версии.

### Rule 3

Для текущего состояния всегда используем latest satellite по `MAX(load_dts)`.

### Rule 4

`record_source` обязателен:

* `frontend`
* `backend`
* `filesystem_scanner`
* `git_adapter`
* `ai_orchestrator`
* `system`

### Rule 5

Файловый контент, rendered prompts и diff blobs храним как refs на диск или blob-cache, а не как основной payload в SQLite.

---

## Почему эта модель подходит именно тебе

Потому что она поддерживает главный архитектурный принцип продукта:

**AI — это не отдельный чат, а слой действий над активным документом.**

В этой модели это отражено естественно:

* `AI Request` связан с `Document Session` и `File`;
* `AI Suggestion` — это артефакт действия над документом;
* `User Action` завершает цикл решения;
* история не теряется;
* accept/reject/export остаются прозрачными;
* backend может легко собрать timeline и текущее состояние.

---

## Что убрать, если хочется ещё проще

Если нужен совсем lean MVP, можно выкинуть один уровень сложности:

### Упрощение A

Не делать `HUB_DOCUMENT_SESSION`, а хранить session как обычный атрибут AI request.

### Упрощение B

Не делать `HUB_PROMPT_TEMPLATE`, если шаблонов мало и они versioned в коде.

### Упрощение C

Не делать `LINK_BRANCH_FILE`, если branch context и так всегда выводится через repository + current branch.

### Минимально жизнеспособный Vault для первого релиза

Тогда остаются:

* `HUB_REPOSITORY`
* `HUB_BRANCH`
* `HUB_FILE`
* `HUB_AI_REQUEST`
* `HUB_AI_SUGGESTION`
* `HUB_USER_ACTION`

И links:

* repository-branch
* repository-file
* ai_request-file
* ai_request-suggestion
* suggestion-user_action

Это уже рабочая модель.

---

## Моя рекомендация для MVP

Не идти в «полный Data Vault», а взять **прагматичный состав**:

### Обязательно

* hubs: repository, branch, file, ai_request, ai_suggestion, user_action
* links: repository_branch, repository_file, ai_request_file, ai_request_suggestion, suggestion_user_action
* satellites: file_attr, file_content_state, ai_request_attr, ai_request_context, ai_suggestion_attr, user_action_attr

### Опционально сразу

* workspace
* document_session
* prompt_template

Это даст:

* историю;
* хорошую трассируемость AI;
* нормальный accept/reject flow;
* простую реализацию в SQLite;
* возможность потом перейти в Postgres без переделки модели.

---

## Следующий логичный шаг после этой модели

После фиксации Data Vault базы стоит сделать 4 артефакта:

1. **ERD**

   * hubs / links / satellites
   * app tables отдельно

2. **API contract**

   * какие endpoints читают current state views;
   * какие создают AI request / action events.

3. **migration plan**

   * SQL migrations для SQLite;
   * seed для prompt templates/settings.

4. **engineering backlog**

   * scanner,
   * vault ingestion,
   * ai orchestration persistence,
   * diff acceptance flow,
   * serving views.

---

## Итог

Для этого MVP Data Vault нужен не ради «модной методологии», а ради 3 вещей:

* историчность;
* трассировка AI-действий;
* чистое разделение между бизнес-объектами и runtime/UI-состоянием.

Лучшее решение здесь — **Data Vault core + простые operational app tables рядом**.

Это будет и архитектурно чисто, и реально поднимется без лишней боли.
