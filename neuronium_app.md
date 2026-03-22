# Neuronium — Полная техническая спецификация
 
> Документ для рефакторинга. Описывает текущую реализацию всех компонентов системы.
 
---
 
## Оглавление
 
1. [Обзор системы](#1-обзор-системы)
2. [Инфраструктура и Docker](#2-инфраструктура-и-docker)
3. [База данных — модели и схема](#3-база-данных--модели-и-схема)
4. [Web API (FastAPI)](#4-web-api-fastapi)
5. [Аутентификация и авторизация](#5-аутентификация-и-авторизация)
6. [RAG-пайплайн](#6-rag-пайплайн)
7. [Агент (ReAct)](#7-агент-react)
8. [Chat Pipeline (SSE-стриминг)](#8-chat-pipeline-sse-стриминг)
9. [Загрузка и обработка файлов](#9-загрузка-и-обработка-файлов)
10. [GPU Worker](#10-gpu-worker)
11. [Media Worker (генерация изображений/видео)](#11-media-worker)
12. [Telegram-бот (app/)](#12-telegram-бот-app)
13. [Система навыков бота](#13-система-навыков-бота)
14. [Поисковый агент (LangGraph)](#14-поисковый-агент-langgraph)
15. [Agent Execution (MCP)](#15-agent-execution-mcp)
16. [Система уведомлений](#16-система-уведомлений)
17. [Confluence интеграция](#17-confluence-интеграция)
18. [Шеринг чатов](#18-шеринг-чатов)
19. [LLM провайдеры и Model Registry](#19-llm-провайдеры-и-model-registry)
20. [Настройки и конфигурация](#20-настройки-и-конфигурация)
21. [Зависимости и версии](#21-зависимости-и-версии)
22. [Диаграмма взаимодействия компонентов](#22-диаграмма-взаимодействия-компонентов)
 
---
 
## 1. Обзор системы
 
Neuronium — многослойная AI-система: Telegram-бот + Web API с поддержкой RAG, агентного рассуждения, обработки файлов, модульных навыков и интеграции с Confluence.
 
**Компоненты:**
 
| Компонент | Директория | Назначение |
|-----------|-----------|------------|
| Web API | `db/web_api/` | Публичный REST API (FastAPI, порт 8008) |
| Telegram-бот | `app/` | Бот на aiogram 3.x |
| GPU Worker | `gpu_worker/` | Воркер эмбеддингов (sentence-transformers, Redis Streams, Qdrant gRPC) |
| Ingest Worker | `db/web_api/services/ingestion/` | Парсинг файлов |
| Media Worker | `db/web_api/services/media/` | Генерация изображений/видео |
| Uploads Parse Worker | `db/web_api/services/uploads_parse_worker.py` | Парсинг chat-аттачей |
| БД модели | `db/model/` | SQLAlchemy ORM |
| Миграции | `db/migrations/` | Alembic |
| Legacy агенты | `src/agents/` | Старые агенты (не используются активно) |
 
**Точки входа:**
 
| Компонент | Файл | Команда |
|-----------|------|---------|
| Web API | `db/web_main.py` | `python -m db.web_main` |
| Telegram-бот | `main.py` | `python main.py` |
| GPU Worker | `gpu_worker/main.py` | `python -m gpu_worker.main` |
| File Ingest | `db/web_api/services/ingestion/file_worker.py` | `python -m db.web_api.services.ingestion.file_worker` |
| Media Worker | `db/web_api/services/media/worker.py` | `python -m db.web_api.services.media.worker` |
| Uploads Parse | `db/web_api/services/uploads_parse_worker.py` | `python -m db.web_api.services.uploads_parse_worker` |
 
---
 
## 2. Инфраструктура и Docker
 
### 2.1 Сервисы (docker-compose.yml, 383 строки)
 
| Сервис | Образ | Порт | Назначение |
|--------|-------|------|------------|
| db | postgres:15-alpine | 5432 | PostgreSQL (ltree, JSONB) |
| redis | redis:7-alpine | 6379 | Streams, кэш, сессии |
| qdrant | qdrant/qdrant:v1.14.1 | 6333 (REST), 6334 (gRPC) | Векторная БД для RAG |
| elasticsearch | elasticsearch:8.14.3 | 9200 | BM25 sparse search |
| minio | minio/minio:latest | 9000 (API), 9001 (console) | S3-совместимое хранилище |
| web-api | Custom (python:3.10-slim) | 8008 | Публичный REST API |
| db-api | Custom | internal | Внутренний API |
| bot | Custom | — | Telegram-бот |
| ingest-worker | Custom | — | Парсинг файлов |
| uploads-parse-worker | Custom | — | Парсинг chat-файлов |
| media-worker | Custom | — | Генерация медиа |
 
### 2.2 Сеть и volumes
 
- **Сеть:** `proxynet` (external, shared с Traefik)
- **Volumes:** `postgres_data` (external), `redis_data`, `minio_data`, `qdrant_storage`, `es_data`
- **Health checks:** на всех stateful-сервисах
 
### 2.3 Traefik
 
```
api.neuronium.ai → web-api:8008
s3.neuronium.ai → minio:9000
s3-console.neuronium.ai → minio:9001
```
 
Middleware chain: DDoS rate-limit → rate-limit → in-flight-req → security-headers → compress.
 
SSE-роуты (PathRegexp `^/api/web/v1/chats/.*/messages/stream$`) — без компрессии, с `Cache-Control: no-cache`, `Connection: keep-alive`.
 
### 2.4 Топология сети
 
```
Internet → Traefik (HTTPS) → Docker proxynet
  ├→ web-api:8008
  ├→ db-api (internal)
  ├→ bot (Telegram polling)
  ├→ db:5432
  ├→ redis:6379
  ├→ qdrant:6333/6334
  ├→ elasticsearch:9200
  └→ minio:9000/9001
 
GPU Worker (внешний, Runpod):
  ├→ Redis Streams (SSH tunnel → debian:6379)
  ├→ Qdrant gRPC (SSH tunnel → debian:6335)
  └→ web-api HTTP (для chunks + callbacks)
```
 
---
 
## 3. База данных — модели и схема
 
### 3.1 Все модели (`db/model/`)
 
**User** (`user_model.py`)
```
Таблица: user
├── tg_id: BigInteger [PK]
├── nickname: String(255)
├── first_name, last_name: String(255)
├── photo_url: String(1024)
├── language: String(10)
├── timestamp: DateTime
├── gmt_offset: Integer
└── relationships: messages (TelegramMessage), chats (Chat)
```
 
**Chat** (`chat_model.py`)
```
Таблица: chat
├── id: Integer [PK, autoincrement]
├── user_tg_id: BigInteger [FK → user.tg_id, indexed]
├── parent_id: Integer [FK → chat.id, self-ref, nullable]
├── title: String(200)
├── description: Text
├── system_prompt: Text
├── model: String(100)
├── temperature: Float
├── path: LtreeType [формат "1.42.99", GiST-индекс]
├── deleted_at: DateTime [soft delete, indexed]
├── active_revision_id: Integer [FK → chat_history_revision.id]
├── created_at, updated_at: DateTime [TimestampMixin]
└── relationships: user, parent, children, messages, file_links
```
 
**ChatMessage** (`chat_model.py`)
```
Таблица: chat_message
├── id: Integer [PK]
├── chat_id: Integer [FK → chat.id, indexed]
├── role: String(20) [user|assistant|system]
├── content: Text
├── client_message_id: String(128) [дедупликация]
├── sources: JSONB [RAG-метаданные]
├── deleted_at: DateTime [soft delete]
├── active_version_index: Integer [какая версия активна]
└── relationships: chat, versions (ChatMessageVersion)
```
 
**FileBlob** (`file_model.py`)
```
Таблица: file_blob
├── id: Integer [PK]
├── sha256: String(64) [unique index — дедупликация]
├── size_bytes: BigInteger
├── mime_type: String(200)
├── storage_backend: String(20) [s3|url|inline]
├── storage_key: Text [S3 key]
├── created_at: DateTime
└── relationships: links (ChatFileLink)
```
 
**ChatFileLink** (`file_model.py`)
```
Таблица: chat_file_link
├── id: Integer [PK]
├── chat_id: Integer [FK → chat.id, indexed]
├── file_blob_id: Integer [FK → file_blob.id, indexed]
├── original_filename: String(500)
├── parse_status: String(32) [pending → processing → ready | failed]
├── parse_error_code: String(64)
├── parse_error_reason: Text
├── embed_status: String(32) [pending → processing → ready | failed]
├── embed_error_reason: Text
├── created_at, updated_at: DateTime
└── relationships: blob, chat
```
 
**FileChunk** (`file_model.py`)
```
Таблица: file_chunk
├── id: Integer [PK]
├── chat_file_link_id: Integer [FK → chat_file_link.id, indexed]
├── chunk_index: Integer [indexed]
├── content: Text [2000 символов]
├── char_start, char_end: Integer
├── page: Integer [nullable, для PDF]
└── created_at: DateTime
```
 
**AuthSession** (`auth_session_model.py`)
```
Таблица: auth_sessions
├── id: Integer [PK]
├── user_tg_id: BigInteger [indexed]
├── jti: String(512) [unique — JWT ID]
├── expires_at: DateTime
└── revoked: Boolean
```
 
**ChatSharedMap** (`share_model.py`)
```
Таблица: chat_shared_map
├── id: Integer [PK]
├── target_chat_id: Integer [FK → chat.id, unique]
├── source_chat_id: Integer [FK → chat.id]
├── mode: String(16) [readonly]
└── created_at, updated_at: DateTime
```
 
**ChatHistoryRevision** (`chat_history_model.py`)
```
Таблица: chat_history_revision
├── id: Integer [PK]
├── chat_id: Integer [FK → chat.id, indexed]
├── title: String(200)
├── messages_json: JSONB [снимок истории]
├── message_count: Integer
└── created_at, updated_at: DateTime
```
 
**ChatMessageVersion** (`message_version_model.py`)
```
Таблица: chat_message_version
├── id: Integer [PK]
├── message_id: Integer [FK → chat_message.id, cascade delete]
├── version_index: Integer [1..N]
├── content: Text
├── created_by: BigInteger [tg_id]
└── created_at, updated_at: DateTime
```
 
**MediaGeneration** (`media_generation_model.py`)
```
Таблица: media_generation
├── id: Integer [PK]
├── chat_id: Integer [FK → chat.id]
├── user_tg_id: BigInteger
├── message_id: Integer [FK → chat_message.id]
├── type: String(16) [image|video]
├── provider: String(32) [openai|fal]
├── model_id: String(200)
├── user_prompt, effective_prompt: Text
├── params_json: JSONB
├── provider_payload_json: JSONB
├── source_attachment_id: Integer
├── source_kind: String(32)
├── status: String(20) [queued|processing|ready|failed]
├── error_code: String(64), error_message: Text
├── client_request_id: String(128), request_hash: String(64)
└── created_at, updated_at: DateTime
```
 
**ChatMessageAttachment** (`chat_message_attachment_model.py`)
```
Таблица: chat_message_attachment
├── id: Integer [PK]
├── chat_id: Integer [FK → chat.id]
├── message_id: Integer [FK → chat_message.id]
├── generation_id: Integer [FK → media_generation.id]
├── media_type: String(16) [image|video]
├── file_blob_id: Integer [FK → file_blob.id]
├── mime_type: String(200)
├── width, height: Integer
├── duration_ms: Integer
├── version_index: Integer [default 1]
├── is_active: Boolean
├── thumbnail_key: Text
└── created_at, updated_at: DateTime
```
 
**UploadSession** (`upload_session_model.py`)
```
Таблица: upload_session
├── id: String(64) [PK, UUID]
├── user_tg_id: BigInteger [FK → user.tg_id]
├── chat_id: Integer [FK → chat.id]
├── kind: String(32) [chat_image|media_source_image|chat_file]
├── object_key: Text [S3 key]
├── original_filename, mime_type: String
├── size_bytes: BigInteger, sha256: String(64)
├── status: String(16) [issued|finalized|cancelled|expired]
├── expires_at: DateTime
├── final_message_id, final_attachment_id: Integer
├── parse_status: String(16) [pending|queued|processing|ready|failed]
├── parse_error_code, parse_error_reason: String/Text
├── parsed_text: Text, parsed_truncated: Boolean
└── created_at, updated_at: DateTime
```
 
**Confluence модели** (`confluence_model.py`, `confluence_chat_model.py`)
```
Таблицы: confluence_connection, confluence_space_sub, confluence_page_map,
         atlassian_app_owner_credential, confluence_chat_space_sub, confluence_chat_page_map
```
 
**Прочие:** `SkillContext`, `SkillMessage`, `TelegramMessage`, `Alarm`, `NutritionModel`, `MealTimes`, `MealTrackingModel`, `SkillProfile`, `SkillNotification`, `ChatFileSourceMeta`
 
### 3.2 Миграции (Alembic)
 
Конфиг: `db/alembic.ini`. 20+ миграций. Команды:
```bash
alembic -c db/alembic.ini upgrade head
alembic -c db/alembic.ini revision --autogenerate -m "desc"
```
 
### 3.3 Repository Layer (`db/repository/`)
 
**chat_repository.py** — основной DAL:
```python
class ChatRepository:
    async def create_chat(session, user_tg_id, title, description, system_prompt, model_name, temperature, parent_id) -> Chat
    async def get_chat(session, user_tg_id, chat_id) -> Optional[Chat]
    async def list_chats(session, user_tg_id, parent_id, limit, cursor) -> (List, next_cursor)
    async def update_chat(session, chat, ...) -> Chat
    async def soft_delete_chat(session, chat) -> None  # sets deleted_at
    async def list_messages(session, user_tg_id, chat_id, limit, cursor)
    async def create_user_message(session, chat_id, content, client_message_id) -> ChatMessage
    async def create_assistant_message(session, chat_id, content, sources) -> ChatMessage
    async def soft_delete_message(session, chat_id, message_id) -> None
```
 
**user_repository.py:**
```python
class UserRepo:
    async def get_user_by_tg_id(tg_id, session) -> Optional[User]
    async def create_user(tg_id, nickname, language, session, ...) -> User
    async def update_user_gmt(tg_id, gmt_offset, session) -> User
```
 
### 3.4 Подключение к БД (`db/core/database.py`)
 
```python
class DatabaseConnection:
    # Async engine: create_async_engine(DATABASE_URI)
    # Pool: pool_size=20, max_overflow=40, pool_timeout=60, pool_recycle=1800
    # Session: sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
    async def get_async_session() -> AsyncGenerator[AsyncSession, None]
```
 
---
 
## 4. Web API (FastAPI)
 
### 4.1 Архитектура слоёв
 
```
routes/ (FastAPI роутеры) → services/ (бизнес-логика) → db/repository/ (DAL) → db/model/ (ORM)
```
 
- Фабрика приложения: `db/web_api/app.py`
- Агрегация роутов: `db/web_api/api/v1/router.py` → префикс `/api/web/v1`
- Настройки: `db/web_api/core/settings.py` (100+ env vars)
- Rate limiting: `db/web_api/core/rate_limits.py` (in-memory sliding window)
 
### 4.2 Все роуты (17 файлов)
 
| Файл | Префикс | Теги | Назначение |
|------|---------|------|------------|
| `auth.py` | `/auth` | Auth | Telegram OAuth, JWT выдача/ротация/отзыв |
| `me.py` | `/me` | Me | Профиль текущего пользователя |
| `chats.py` | `/chats` | Chats | CRUD чатов, дерево, SSE-стриминг `/messages/stream` |
| `files.py` | `/files`, `/chats/{id}/files` | Files | S3 presign, завершение загрузки, скачивание, удаление |
| `models.py` | `/models` | Models | Список LLM моделей |
| `rag_debug.py` | `/rag-debug` | RAG-Debug | Отладка retrieval |
| `sources.py` | `/chats/{id}/sources` | Sources | Сохранение текста как RAG-источник |
| `share.py` | `/chats/{id}/share` | Share | Создание share-токена, импорт |
| `chat_history.py` | `/chats/{id}/history` | ChatHistory | Снимки ревизий, восстановление |
| `message_versions.py` | `/chats/{id}/messages/{id}/versions` | MessageVersions | Альтернативные ответы |
| `ingest.py` | `/ingest` | Ingest | Коллбэки GPU-воркера |
| `file_events.py` | `/files` | FileEvents | SSE-события статуса файла |
| `upload_events.py` | `/uploads` | UploadEvents | SSE-события upload session |
| `media.py` | `/chats/{id}/media` | Media | Генерация изображений/видео |
| `attachments.py` | `/chats/{id}/attachments` | Attachments | Скачивание аттачей |
| `uploads.py` | `/chats/{id}/uploads` | Uploads | Draft-загрузки (presign + finalize) |
| `confluence.py` | `/confluence` | Confluence | OAuth 3LO, подписки, статус |
| `chat_sources_confluence.py` | `/chats/{id}/sources/confluence` | ChatSources-Confluence | Привязка Confluence к чату |
 
### 4.3 Ключевые эндпоинты — детали
 
#### AUTH
 
**POST /auth/telegram** — Telegram OAuth login
- Request: `{ id, first_name, last_name, username, photo_url, auth_date, hash }`
- Валидация HMAC виджета Telegram (возраст токена ≤ 86400 сек)
- Создание/обновление User
- Выдача access JWT (HS256, TTL: `ACCESS_TOKEN_TTL_MINUTES=60`)
- Выдача refresh JWT (unique jti, TTL: `REFRESH_TOKEN_TTL_DAYS=30`) в HttpOnly cookie
- Создание AuthSession (jti)
 
**POST /auth/refresh** — ротация refresh token
- Из cookie `refresh_token`
- Отзывает старый jti, выдаёт новую пару
 
#### CHATS
 
**POST /chats** — создание чата
- Request: `{ title?, description?, system_prompt?, model?, temperature?, parent_id? }`
- Вычисляет ltree path: `parent.path || text2ltree(id)` или `text2ltree(id)`
 
**GET /chats** — список чатов
- Query: `parent_id?, limit=50, cursor?`
- Keyset pagination: `(updated_at, id) DESC`
 
**GET /chats/{id}/children** — дочерние чаты (один уровень)
**GET /chats/{id}/descendants** — все потомки (плоский список через ltree `<@`)
**GET /chats/{id}/tree** — вложенное дерево с файлами и счётчиками
 
**PATCH /chats/{id}** — обновление (title, description, system_prompt, model, temperature)
**DELETE /chats/{id}** — soft-delete (sets deleted_at, рекурсивно для сообщений)
 
**POST /chats/{id}/messages/stream** — SSE-стриминг (основной эндпоинт, описан в секции 8)
 
#### FILES
 
**POST /files/presign** — S3 presigned PUT URL
- Request: `{ chat_id, filename, size, mime?, sha256? }`
- Response: `{ upload_token, url, method, headers, expires_sec, object_key }`
- Key format: `user/{tg_id}/chat/{chat_id}/{uuid}/{filename}`
- TTL: `S3_PRESIGN_EXPIRES_SEC=900`
 
**POST /chats/{id}/files/complete** — завершение загрузки
- Валидирует upload_token, HEAD S3
- Дедупликация FileBlob по SHA256
- Создаёт ChatFileLink, ставит в очередь парсинга
- Backpressure: 503 если `ingest_queue_len > INGEST_QUEUE_MAX_LEN`
 
**DELETE /files/{id}** — удаление (ChatFileLink, FileChunk, Qdrant vectors, ES docs)
 
**POST /files/import-link** — импорт URL
- Скачивает HTML, извлекает текст (trafilatura), чанкит, индексирует
 
#### UPLOADS (Draft-загрузки для чата)
 
**POST /chats/{id}/uploads/presign**
- Request: `{ kind: chat_image|media_source_image|chat_file, filename, size, mime }`
- Валидация: images max 4×20MB (40MB total), files max 5×50MB
- Создаёт UploadSession (status=issued)
 
**POST /chats/{id}/uploads/{upload_id}/finalize**
- Для `chat_file`: ставит в очередь `UPLOAD_PARSE_STREAM`
- Для `chat_image`: создаёт FileBlob + ChatMessageAttachment
 
**GET /chats/{id}/uploads/{upload_id}/status** — SSE-события статуса
 
#### SOURCES
 
**POST /chats/{id}/messages/{msg_id}/save-as-source**
- Создаёт subchat с наследованием system_prompt/model/temperature
- Сохраняет текст как inline-файл (FileBlob + ChatFileLink)
- Чанкит и индексирует немедленно (fastembed + Qdrant + ES)
 
#### SHARE
 
**POST /chats/{id}/share/create** → HMAC-SHA256 токен (stateless, ~44 base64url chars)
**POST /chats/share-import** → клонирование дерева чатов + файлов + Qdrant vectors
 
#### HISTORY & VERSIONS
 
**POST /chats/{id}/history/save** — снимок текущих сообщений
**POST /chats/{id}/history/{rev_id}/restore** — восстановление
**GET /chats/{id}/messages/{msg_id}/versions** — альтернативные ответы
**POST .../versions/{index}/activate** — переключение версии
 
### 4.4 Rate Limiting (`core/rate_limits.py`)
 
In-memory sliding window:
- `/auth/telegram`: 10 req/min
- `/auth/refresh`: 60 req/hour
- `/chats/.../messages/stream`: 60 req/min
- Other: 100 req/min
- Exempt: `/ingest/callback`
 
### 4.5 Startup/Shutdown (`app.py`)
 
```python
@app.on_event("startup"):
    - Preload fastembed (если RAG_EMBEDDINGS_PRELOAD)
    - Start Confluence poller
    - Start uploads GC
    - Start SSE Redis stream readers
 
@app.on_event("shutdown"):
    - Flush Langfuse traces
    - Close async S3 client
```
 
### 4.6 Обработка ошибок (`core/errors.py`)
 
```python
class LLMProviderError(Exception): ...
 
USER_MSG_INTERNAL_ERROR = "Произошла внутренняя ошибка. Попробуйте повторить запрос."
USER_MSG_LLM_UNAVAILABLE = "Сервис генерации временно недоступен."
USER_MSG_FILE_PARSE_FAILED = "Не удалось обработать файл."
USER_MSG_MEDIA_FAILED = "Не удалось сгенерировать медиа."
```
 
---
 
## 5. Аутентификация и авторизация
 
### 5.1 JWT-схема (`services/tokens.py`)
 
**Access Token** (короткоживущий, HS256):
```json
{ "sub": "<tg_id>", "exp": <ts>, "iat": <ts>, "type": "access" }
```
- TTL: `ACCESS_TOKEN_TTL_MINUTES` (default 60)
- Передаётся: `Authorization: Bearer <token>`
 
**Refresh Token** (долгоживущий, HS256):
```json
{ "sub": "<tg_id>", "exp": <ts>, "iat": <ts>, "type": "refresh", "jti": "<uuid>" }
```
- TTL: `REFRESH_TOKEN_TTL_DAYS` (default 30)
- Хранение: HttpOnly cookie `refresh_token` + AuthSession в БД (jti, expires_at, revoked)
 
**Service Token:**
- Статический Bearer token (env `SERVICE_TOKEN`)
- Для коллбэков GPU-воркера
- Проверка через `secrets.compare_digest()`
 
### 5.2 FastAPI-зависимости
 
```python
def get_current_user_tg_id(credentials) -> int:
    # Валидирует access token, возвращает tg_id
 
def get_user_tg_id_optional(credentials) -> int | None:
    # Для смешанных эндпоинтов
 
def is_service_request(authorization) -> bool:
    # Проверяет SERVICE_TOKEN
```
 
### 5.3 Контроль доступа
 
- Все операции с чатами: `chat.user_tg_id == current_user`
- Расшаренные чаты: read-only через ChatSharedMap
- Service эндпоинты: `is_service_request()` (GPU worker, admin)
 
---
 
## 6. RAG-пайплайн
 
### 6.1 Ingestion (два этапа)
 
```
Загрузка → S3 (FileBlob)
  → Redis Stream (ingest:file_jobs)
  → File Worker: парсинг → FileChunk (parse_status=ready)
  → Redis Stream (ingest:embed_jobs)
  → GPU Worker / Local fastembed: эмбеддинги → Qdrant + ES (embed_status=ready)
```
 
### 6.2 Парсинг файлов (`services/file_ingest.py`)
 
```python
extract_text(file_path: str, mime: str) -> str
split_into_chunks_with_offsets(text: str, chunk_size: int = 2000) -> List[dict]
```
 
**Поддерживаемые форматы:**
- PDF: PyMuPDF (fitz), fallback pypdf
- DOCX: python-docx
- XLSX: openpyxl
- Код/JSON/YAML: pretty-print + sanitize
- Plaintext: normalize whitespace
 
**Deny-list:** exe, dll, zip, rar, 7z, iso, dmg и др. бинарники/архивы
 
**Sniffing:** text fraction ≥ 0.85, null fraction < 0.01 (probe 4096 bytes)
 
**Санитизация:**
- Удаление PEM-ключей (`-----BEGIN...-----END`)
- Маскирование API keys, паролей, токенов (regex)
- Маскирование base64-блобов (120+ chars)
 
**Чанкинг:** фиксированные 2000 символов, перекрытие с соседями. Хранится в FileChunk с `char_start`, `char_end`, `page`.
 
### 6.3 Эмбеддинги (`services/embeddings.py`)
 
```python
class FastEmbedEmbedder:
    # fastembed ONNX/CPU
    # Модель: RAG_EMBEDDING_MODEL_NAME (default: paraphrase-multilingual-MiniLM-L12-v2)
    # Размер вектора: 384
    def embed_passages(texts: List[str]) -> np.ndarray  # shape: (N, 384)
    def embed_query(query: str) -> np.ndarray            # shape: (384,)
 
class _EmbeddingsCache:
    # SQLite-кэш: key → float32 BLOB
    # Key = hash(model_name, kind, max_seq_len, text_sha256)
    # Файл: RAG_EMBEDDINGS_CACHE_PATH (default: db/web_api/data/emb_cache.sqlite)
    def get_many(keys: List[str]) -> Dict[str, np.ndarray]
    def set_many(items: Dict[str, np.ndarray]) -> None
```
 
Настройки:
- `RAG_EMBEDDINGS_BATCH_SIZE=128`
- `RAG_EMBEDDINGS_QUANTIZE_DYNAMIC=true` (dynamic int8)
- `RAG_EMBEDDINGS_WARMUP=true`
- `RAG_NUM_THREADS=8`
 
### 6.4 Векторное хранилище — Qdrant (`services/rag_qdrant.py`)
 
```python
class QdrantRag:
    # Коллекция: rag_chunks, вектор 384, COSINE
    def ensure_collection(vector_size: int) -> None
    def upsert_file(chat_id, ancestor_ids, file_id, file_name, chunks: List[str]) -> int
    def retrieve(chat_id, query_text, k) -> List[RetrievedChunk]
    def retrieve_many(chat_ids, query_text, k) -> List[RetrievedChunk]
    def delete_file(file_id) -> None
    def clone_file_points(source_file_id, new_chat_id, new_ancestor_ids, new_file_id, new_file_name) -> int
```
 
**Payload каждой точки:**
```json
{
  "chat_id": int,
  "ancestor_ids": [int],
  "file_id": int,
  "file_name": "string",
  "chunk_index": int,
  "text": "string",
  "sha256": "string",
  "mime_type": "string",
  "page": int|null
}
```
 
**RetrievedChunk:**
```python
@dataclass
class RetrievedChunk:
    file_id: int
    file_name: str
    chunk_index: int
    score: float
    text: str
    vector: Optional[List[float]] = None
```
 
### 6.5 Sparse Search — Elasticsearch (`services/retrieval/bm25_es.py`)
 
- Индекс: `rag_chunks`
- BM25 full-text
- Операции: `index_chunks()`, `retrieve()`, `retrieve_many()`, `delete_file()`
 
### 6.6 Гибридный retrieval (`services/retrieval/hybrid.py`)
 
**Алгоритм: Reciprocal Rank Fusion (RRF)**
 
```python
def rrf_merge(dense, sparse, k, w_dense, w_sparse) -> List[RetrievedChunk]:
    # score = w_dense * (1/(K+rank_dense)) + w_sparse * (1/(K+rank_sparse))
    # ID документа: (file_id, chunk_index)
 
def retrieve_hybrid(chat_id, query_text, dense_k, sparse_k=None) -> List[RetrievedChunk]:
    # dense: Qdrant top dense_k
    # sparse: ES BM25 top sparse_k
    # merge: RRF
```
 
### 6.7 RAG Pipeline (`services/rag_pipeline.py`)
 
Многоэтапный retrieval:
 
```python
async def retrieve_with_pipeline(chat_id, user_query, model, history=None, use_rag=True)
    -> Tuple[List[RetrievedChunk], List[Dict]]
```
 
**Шаги:**
 
1. **Multi-Query** (опционально, `RAG_REFORMULATE_ENABLED=false` по умолчанию)
   - LLM генерирует 2-3 альтернативные формулировки
   - До `RAG_MULTIQUERY_NUM=3` вариантов
   - Учитывает последние `RAG_MULTIQUERY_HISTORY_LAST_N=6` сообщений
 
2. **Гибридный поиск** (для каждого запроса)
   - Dense: Qdrant `RAG_TOP_K=30`
   - Sparse: ES `RAG_SPARSE_TOP_K=150`
   - RRF fusion: `RAG_HYBRID_RRF_K=60`, веса `RAG_HYBRID_DENSE_WEIGHT=1.0`, `RAG_HYBRID_SPARSE_WEIGHT=1.0`
 
3. **Фильтрация по скору** — `RAG_SCORE_THRESHOLD=0.2`
 
4. **Фильтр по дереву чатов** — только чанки из целевого чата и потомков (ltree)
 
5. **Дедупликация** — unique по `(file_id, chunk_index)`
 
6. **MMR диверсификация** (`_mmr_select()`)
   - λ = `RAG_MMR_LAMBDA=0.5`
   - Top `RAG_CANDIDATES_PER_QUERY=20`
 
7. **Расширение соседей** — подгрузка `RAG_NEIGHBORS_EACH_SIDE=1` соседних чанков
 
8. **Реранкинг** (опционально) — cross-encoder если `RAG_RERANKER_MODEL_NAME` задана, max `RAG_RERANKER_MAX_PAIRS=100`
 
9. **Сборка контекста** — конкатенация до `RAG_MAX_CONTEXT_CHARS=10000`
 
---
 
## 7. Агент (ReAct)
 
### 7.1 Оркестратор (`services/agent/orchestrator.py`)
 
```python
class Agent:
    def __init__(self, model: str, max_steps: int = 6, allowed_actions: Set[str] = None, stream_thoughts: bool = True)
 
    async def run_iter(session, user_tg_id, chat_id, question, user_content=None) -> AsyncGenerator[Dict, None]:
        # Yields: { type: "thought"|"step"|"observation"|"error", ... }
```
 
**ReAct-цикл:**
1. System prompt + chat history + tools description
2. LLM с параметром `tools` (native function calling)
3. LLM возвращает: message + tool_calls (или просто message)
4. Для каждого tool_call → dispatch → result → append к history → продолжить
5. Max `AGENT_MAX_STEPS=6` итераций
6. Финальный LLM-вызов без tool_calls → текстовый ответ
 
### 7.2 Tool Registry (`services/agent/tool_registry.py`)
 
```python
@dataclass(frozen=True)
class ArgSchema:
    name: str
    type: str          # "integer"|"string"|"number"|"boolean"
    description: str
    required: bool = True
    default: Any = None
 
@dataclass(frozen=True)
class ToolDef:
    name: str
    description: str
    category: str      # "system"|"files"|"web"
    args_schema: tuple[ArgSchema, ...]
    handler: Callable   # (ctx: ToolContext, args: dict) -> dict
 
    def to_openai_function() -> Dict:
        # Конвертация в OpenAI function-calling schema
 
@dataclass
class ToolContext:
    session: AsyncSession
    user_tg_id: int
    chat_id: int
    model: str
 
class ToolRegistry:
    def register(tool: ToolDef) -> None
    def to_openai_tools(allowed: Set[str]) -> List[Dict]
    async def execute(name: str, args: Dict, ctx: ToolContext) -> Dict
```
 
### 7.3 Доступные инструменты
 
| Инструмент | Файл | Args | Назначение |
|------------|------|------|------------|
| `structure.list` | `tools/structure.py` | `chat_id, limit_children?, limit_files?` | Список подчатов и файлов |
| `files.read_chunks` | `tools/files_window.py` | `file_id, start?, limit?` | Чтение чанков с пагинацией |
| `sparse.search` | `tools/sparse.py` | `query, top_k?, mode?` | BM25-поиск через ES |
| `rag.search` | через registry | `query, top_k?` | Полный RAG-пайплайн (гибридный) |
| `web.search` | `tools/web_search.py` | `query, num_results?` | Tavily API |
| `web.fetch` | `tools/web_fetch.py` | `url` | Jina AI (извлечение контента) |
| `finish` | built-in | `answer` | Завершить цикл |
 
Все инструменты — **только чтение**, без мутаций.
 
### 7.4 Tool Policy (`services/agent/tool_policy.py`)
 
```python
def build_agent_tool_policy(user_tg_id, use_rag, use_web) -> Set[str]:
    # Возвращает разрешённые инструменты на основе флагов
 
def ensure_minimum_agent_actions(allowed: Set[str]) -> Set[str]:
    # Гарантирует минимальный набор (finish всегда доступен)
```
 
### 7.5 Промпты (`services/agent/prompts.py`)
 
```python
def build_system_prompt() -> str
def build_agent_environment_prompt() -> str
 
class ConversationHistory:
    # Multi-turn tracking: хранит assistant-ответы и tool-результаты
    # Sliding window для длинных разговоров
```
 
---
 
## 8. Chat Pipeline (SSE-стриминг)
 
### 8.1 Основной эндпоинт
 
`POST /chats/{chat_id}/messages/stream` (multipart/form-data)
 
**Request:**
- `content` (текст, required)
- `use_rag` (bool, default true)
- `client_message_id` (дедупликация)
- `files[]` (эфемерные, max 5×50MB)
 
### 8.2 Dataclasses (`services/chat_pipeline.py`)
 
```python
@dataclass
class ImageInput:
    url: str              # presigned S3 URL
    filename: str
    detail: str = "auto"  # OpenAI vision: auto|low|high
    message_id: Optional[int] = None
 
@dataclass
class UserMessage:
    text: str
    images: List[ImageInput] = []
    file_chunks: List[Dict] = []
    eph_files: List[Dict] = []
 
@dataclass
class PipelineRequest:
    session: AsyncSession
    user_tg_id: int
    chat_id: int
    chat: model.Chat
    content: str
    user_msg: model.ChatMessage
    client_message_id: Optional[str]
    user_message: UserMessage
    chat_system_prompt: str
    use_rag: bool
    eff_files_search: bool
    eff_web_search: bool
    include_reasoning: bool
    reasoning_throttle_ms: int
    token_throttle_ms: int
 
@dataclass
class PipelineResult:
    assistant_message_id: int
    collected_text: str
    qdrant_sources: List[Dict]
    timings: Dict
```
 
### 8.3 Последовательность SSE-событий
 
```
1. message.created     — сообщение пользователя сохранено
2. agent.thought.start — начало рассуждения (если REASONING_ENABLED=true)
3. agent.thought.delta — токены рассуждения (throttle: REASONING_THROTTLE_MS=80ms)
4. agent.thought.end   — конец рассуждения
5. agent.step          — вызов инструмента { index, action, args }
6. agent.observation   — результат инструмента (усечённый)
7. context             — RAG-источники { source_type, sources[] }
8. token               — потоковый вывод модели { delta } (throttle: TOKEN_THROTTLE_MS=0)
9. done                — завершение стрима
```
 
### 8.4 Поток обработки
 
1. **Подготовка** — парсинг multipart (текст, images, files), валидация
2. **System prompt** — chat-level system_prompt + capabilities block
3. **История** — `truncate_history()` по `CHAT_HISTORY_MAX_CHARS=80000`
4. **Сборка messages** — system + history + user message с images/file chunks
5. **Agent или Direct LLM** — если есть инструменты → ReAct цикл, иначе прямой completion
6. **RAG sources** — если agent вызвал rag.search → захват результатов
7. **Стриминг** — токены через SSE, throttle
8. **Сохранение** — assistant message в БД с sources JSONB
9. **Auto-title** — если chat.title is null и первое сообщение: LLM → 3-5 слов (timeout `AUTO_TITLE_TIMEOUT_MS=20000`)
 
### 8.5 SSE Hub (`services/sse_hub.py`)
 
```python
class EventHub:
    async def subscribe(topic, chat_id) -> asyncio.Queue
    async def publish(topic, chat_id, payload) -> None
    # Backpressure: drop oldest (SSE_DROP_OLDEST_LIMIT=200)
    # Disconnect slow clients (SSE_DISCONNECT_AFTER_DROPS=500)
    # Keep-alive: SSE_KEEP_ALIVE_SEC=15
    # Queue size: SSE_QUEUE_MAXSIZE=500
```
 
Redis Streams → фоновый reader → EventHub (in-process fanout) → SSE клиенты.
 
---
 
## 9. Загрузка и обработка файлов
 
### 9.1 Двухфазная загрузка (RAG-файлы)
 
```
1. POST /files/presign → S3 presigned PUT URL (TTL 15 мин)
2. Клиент → PUT файл в S3
3. POST /chats/{id}/files/complete:
   - Валидация upload_token + HEAD S3
   - Дедупликация FileBlob по SHA256
   - Создание ChatFileLink
   - Очередь парсинга: Redis Stream ingest:file_jobs
   - SSE-событие file_status
```
 
S3 key format: `user/{tg_id}/chat/{chat_id}/{uuid}/{filename}`
 
### 9.2 Draft-загрузки (chat images/files)
 
```
1. POST /chats/{id}/uploads/presign → UploadSession + presigned URL
   - kind: chat_image | media_source_image | chat_file
   - Object key: user/{tg_id}/chat/{chat_id}/drafts/{kind}/{upload_id}/{filename}
2. Клиент → PUT в S3
3. POST /chats/{id}/uploads/{id}/finalize:
   - chat_file → очередь uploads:parse_jobs (текст в parsed_text)
   - chat_image → FileBlob + ChatMessageAttachment
```
 
### 9.3 File Worker (`services/ingestion/file_worker.py`)
 
Redis consumer на `ingest:file_jobs`:
1. Скачивает FileBlob из S3
2. `extract_text()` — парсинг (PDF/DOCX/XLSX/code/text)
3. `split_into_chunks_with_offsets()` — чанки по 2000 символов
4. Сохраняет FileChunk в БД
5. Обновляет `parse_status=ready`
6. Ставит в очередь `ingest:embed_jobs` (для эмбеддингов)
 
### 9.4 Uploads Parse Worker (`services/uploads_parse_worker.py`)
 
Redis consumer на `uploads:parse_jobs`:
- Парсинг one-time chat аттачей
- Результат: `upload_session.parsed_text` (ephemeral, не RAG-индексированный)
- SSE-события через `events:upload_status`
 
### 9.5 Статусы файлов
 
```
parse_status: pending → processing → ready | failed
embed_status: pending → processing → ready | failed
```
 
Поля ошибок: `parse_error_code`, `parse_error_reason`, `embed_error_reason`
 
---
 
## 10. GPU Worker
 
### 10.1 Архитектура (`gpu_worker/`)
 
Docker: `nvidia/cuda:12.1.0-runtime-ubuntu22.04`, Python 3, sentence-transformers.
 
**Файлы:**
- `main.py` (109 строк) — event loop: `run_worker()` → `handle_job()`
- `config.py` — Pydantic settings
- `embedder.py` — `PassageEmbedder`: sentence-transformers, "passage:" prefix, normalized float32
- `streams.py` — `StreamConsumer`: XREADGROUP, ack/fail, dead letter queue `{stream}:dead`
- `qdrant_sink.py` — `QdrantSink`: gRPC upsert, deterministic UUID5 point IDs
- `models.py` — `EmbedJob`, `CallbackPayload`, `ChunksResponse`
- `http_client.py` — `WebApiClient`: fetch chunks + HMAC-signed callbacks
 
### 10.2 Поток обработки
 
```
1. Redis Streams consumer (ingest:embed_jobs, XREADGROUP, block=5s, count=1)
2. Fetch chunks paginated: GET /files/{file_id}/chunks (WebApiClient)
3. Encode: SentenceTransformer (batch_size=1024, normalize_embeddings=True)
4. Upsert to Qdrant (gRPC, prefer_grpc=True)
   - Point ID: uuid5(namespace_url, f"{file_id}:{chunk_index}:{blake2b(text)}")
   - Payload: {chat_id, ancestor_ids, file_id, file_name, chunk_index, text}
5. Callback: POST /api/web/v1/ingest/callback (HMAC-SHA256 signed, Bearer token)
6. Ack Redis message (или fail → dead letter)
```
 
### 10.3 Модель эмбеддингов
 
- `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`
- Dimension: 384
- Prefix: "passage:" (per e5 model spec)
 
---
 
## 11. Media Worker
 
### 11.1 Архитектура (`services/media/`)
 
- `worker.py` — Redis Streams consumer (`media:jobs`)
- `producer.py` — `enqueue_media_job()` → Redis
- `events.py` — SSE-события прогресса
- `types.py` — типы задач
- `providers/openai_images.py` — OpenAI DALL-E / gpt-image-1
- `providers/fal_images.py` — FAL FLUX
 
### 11.2 Поток
 
```
1. POST /chats/{id}/media/generate → MediaGeneration (status=queued)
2. Redis Stream media:jobs → worker
3. Worker:
   - OpenAI gpt-image-1: POST images/generations
   - FAL FLUX: fal-client submit
4. Результат → S3 (FileBlob) + ChatMessageAttachment
5. SSE-события: events:media → клиент
6. MediaGeneration.status = ready | failed
```
 
### 11.3 Поддерживаемые модели
 
**gpt-image-1** (OpenAI):
- Tasks: image_generation, image_edit
- Sizes: auto, 1024x1024, 1024x1536, 1536x1024
- Quality: auto, low, medium, high
- Formats: png, jpeg, webp
 
**FLUX** (FAL):
- Tasks: image_generation, image_edit
- Sizes: square_hd, square, portrait_4_3, portrait_16_9, landscape_4_3, landscape_16_9
- Steps: 1-50, seed, guidance_scale
 
---
 
## 12. Telegram-бот (app/)
 
### 12.1 Точка входа
 
```python
# main.py → app/bot.py (2222 строки)
async def main():
    await get_mcp_tools()       # Init MCP tools
    await setup_bot()           # Setup handlers & skills
    await dp.start_polling(bot) # Start polling
```
 
### 12.2 Глобальные инстансы (`bot_instance.py`)
 
```python
bot = Bot(token=BOT_TOKEN)
storage = MemoryStorage()  # FSM storage
dp = Dispatcher(storage=storage)
```
 
### 12.3 Middleware (`middleware_user_check.py`)
 
```python
class UserCheckMiddleware(BaseMiddleware):
    # Проверяет/создаёт пользователя в БД перед обработкой
    # Применяется к dp.message и dp.callback_query
```
 
### 12.4 Режимы работы
 
```python
MODE_GENERAL = "general_mode"   # Обычный чат
MODE_SEARCH = "search_mode"     # Веб-поиск
MODE_AGENT = "agent_mode"       # Мульти-задачный агент
```
 
### 12.5 FSM States (`bot.py`)
 
```python
class AgentStates(StatesGroup):
    creating_agent = State()
    editing_plan = State()
    running_agent = State()
```
 
### 12.6 Основные хендлеры (`bot.py`)
 
| Хендлер | Строка | Trigger | Назначение |
|---------|--------|---------|------------|
| `cmd_start()` | ~996 | `/start` | Главное меню |
| `cmd_test()` | ~900 | `/test` | Тестовые команды (для тестовых юзеров) |
| `search_callback()` | ~1014 | `callback_data="search"` | Обычный поиск |
| `deep_search_callback()` | ~1037 | `callback_data="deep_search"` | Глубокий поиск |
| `agents_callback()` | ~1061 | `callback_data="agents"` | Управление агентами |
| `back_to_main_callback()` | ~973 | `callback_data="back_to_menu"` | Навигация назад |
 
### 12.7 Inline-клавиатура главного меню
 
```python
[🖼️ Изображение] [🎬 Видео]
[🔍 Поиск в интернете] [🔬 Глубокий поиск]
[🛌 Мой сон] [🍎 Питание]
[🤖 Мои агенты]
```
 
### 12.8 FSM Context Keys
 
```python
"context_messages"    # [{"query": str, "response": str}] — история диалога
"last_search_query"   # для regeneration
"current_mode"        # general/search/agent
"agent_plan"          # JSON плана агента
"current_agent"       # ID агента
"agent_mapping"       # short_id -> filename
"skill_context_id"    # ID контекста навыка
"in_skill_context"    # bool
```
 
### 12.9 Конфигурация (`config.py`)
 
```python
BOT_TOKEN = os.getenv("BOT_TOKEN")
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
FAL_KEY = os.getenv("FAL_KEY") or os.getenv("FAL_API_KEY")
DB_API = f"http://{DB_API_HOST}:{DB_API_PORT}/api/v1"
 
# Режимы поиска
_DEFAULT_SEARCH_CONFIG = { MAX_SEARCH_RESULTS: 2, MAX_SEARCH_QUERIES: 2, MAX_SUBTASKS: 2, MAX_SOURCES: 5 }
_DEEP_SEARCH_CONFIG = { MAX_SEARCH_RESULTS: 5, MAX_SEARCH_QUERIES: 3, MAX_SUBTASKS: 5, MAX_SOURCES: 30 }
```
 
### 12.10 DB API Layer (`db_api/`)
 
Бот общается с db-api по HTTP:
 
**SkillDataAPI** (`skill_api.py`):
```python
class SkillDataAPI:
    def __init__(self, skill_name: str)  # base_url = DB_API
    async def get_user_profile(user_id) -> Optional[Dict]
    async def update_user_profile(user_id, data) -> bool
    async def get_user_notifications(user_id) -> List[Dict]
    async def create_notification(user_id, skill_name, type, frequency, time, ...) -> int
    async def update_notification(notification_id, **kwargs) -> bool
    async def delete_notification(notification_id) -> bool
```
 
**Другие API:** `user.py` (CRUD юзеров), `message.py` (треккинг сообщений), `skill_context.py` (контексты навыков), `alarm.py`, `nutrition.py`, `meal_times.py`, `meal_tracking.py`
 
### 12.11 Message Manager (`message_manager.py`)
 
```python
class MessageManager:
    async def save_messages(user_tg_id, message_ids, message_type) -> bool
    async def get_message_ids(user_tg_id, message_type) -> List[int]
    async def delete_messages(user_tg_id, message_type, delete_from_telegram=True) -> Dict
```
 
### 12.12 Skill Context Manager (`skill_context_manager.py`, ~850 строк)
 
```python
class SkillContextManager:
    active_contexts: Dict[int, dict]    # user_id -> context
    last_messages: Dict[int, int]       # user_id -> last_msg_id
 
    async def start_skill_context(user_id, skill_name, trigger_message, state) -> dict
    async def end_skill_context(user_id, state) -> bool
    async def add_message_to_context(user_id, role, content) -> bool
    async def get_context_history(user_id) -> List[Dict[str, str]]
    async def process_skill_message(user_id, message_text, message_type, file_path=None) -> str
        # Обработка сообщения внутри контекста навыка через OpenAI
```
 
### 12.13 Telegraph Service (`services/telegraph_service.py`)
 
```python
class TelegraphService:
    async def create_summary_from_search_response(response_text) -> Optional[str]  # 500 chars
    async def create_article_from_search_response(query, response_text, images, links) -> Optional[str]
```
 
---
 
## 13. Система навыков бота
 
### 13.1 BaseSkill (`skills/base_skill.py`)
 
```python
class BaseSkill(ABC):
    name: str = ""
    category: str = ""
    version: str = "1.0.0"
    description: str = ""
    enabled: bool = True
    _notification_services: Dict[str, Any] = {}
    _handlers_registered: bool = False
    _initialized: bool = False
    _cleanup_callbacks: List[Callable] = []
 
    @abstractmethod
    async def register_handlers(self, dp: Dispatcher) -> bool
    @abstractmethod
    async def initialize(self) -> bool
    async def cleanup(self) -> bool
    def get_metadata(self) -> Dict[str, Any]
    def add_notification_service(self, service_name, service)
    def get_notification_service(self, service_name)
    def add_cleanup_callback(self, callback)
    def get_menu_button(self) -> dict  # {"text", "callback_data", "category"}
```
 
### 13.2 SkillManager (`skills/manager.py`)
 
```python
class SkillManager:
    skills: Dict[str, BaseSkill] = {}
    menu_buttons: Dict[str, List[dict]] = {}  # category -> buttons
 
    async def initialize(self, dp: Dispatcher, bot_integration) -> bool:
        register_all_categories(self)
        for skill in self.skills.values():
            await skill.initialize(bot_integration)
            await skill.register_handlers(dp)
            self._register_menu_button(skill)
 
    def register_skill(self, skill: BaseSkill)
    def get_skill(self, skill_name) -> BaseSkill
    async def shutdown()
 
skill_manager = SkillManager()  # глобальный
```
 
### 13.3 Категории навыков
 
```
skills/categories/
├── health/
│   ├── sleep/          — управление сном и будильниками
│   ├── nutrition/      — планирование питания
│   └── training/       — фитнес-тренировки
└── general/
    ├── settings/       — настройки пользователя
    ├── images/         — генерация изображений (FAL/OpenAI)
    └── videos/         — генерация видео
```
 
### 13.4 Структура навыка (на примере Sleep)
 
```
sleep/
├── skill.py                — SleepSkill class, init, register_handlers
├── handlers/
│   ├── deps.py             — единственная точка DI (_db, _notification_service)
│   ├── menu.py             — Router, главное меню
│   ├── gmt.py              — настройка часового пояса
│   ├── alarms_add.py       — добавление будильников
│   ├── alarms_delete.py    — удаление
│   ├── navigation.py       — навигация
│   └── context.py          — контекстный чат
├── services/
│   └── notification_service.py — SleepNotificationService
├── states.py               — SleepStates(StatesGroup)
└── utils.py                — UI-утилиты
```
 
### 13.5 Паттерн инициализации навыка
 
```python
class SleepSkill(BaseSkill):
    async def initialize(self, bot_integration):
        self.db = SkillDataAPI("sleep")
        self.notification_service = SleepNotificationService(bot_integration, self.db)
        await self.notification_service.setup()
        from .handlers.deps import init_deps
        init_deps(self.db, self.notification_service)
 
    async def register_handlers(self, dp):
        from .handlers import menu, gmt, alarms_add, alarms_delete, navigation, context
        dp.include_router(menu.router)
        dp.include_router(gmt.router)
        # ...
```
 
### 13.6 Паттерн DI в хендлерах (`handlers/deps.py`)
 
```python
_db = None
_notification_service = None
 
def init_deps(db, notification_service):
    global _db, _notification_service
    _db = db
    _notification_service = notification_service
 
def get_db(): return _db
def get_notification_service(): return _notification_service
```
 
### 13.7 Паттерн регистрации хендлеров
 
```python
router = Router()
 
@router.callback_query(F.data == "skill_sleep")
async def sleep_menu(callback_query, state):
    db = get_db()
    # ...
 
# В skill.py: dp.include_router(menu.router)
```
 
### 13.8 Правила навыков
 
- Навык самодостаточен в своей папке. Нет зависимостей между навыками.
- `skill.py` — только компоновка (DI, router). Никаких хендлеров/клавиатур.
- `handlers/deps.py` — единственная точка DI.
- Все DB-операции через `SkillDataAPI`.
- Регистрация хендлеров через `dp.include_router()`, НЕ декораторами.
- Абсолютные импорты: `from app.skills.base_skill import BaseSkill`.
- Namespace callback_data: `skill_<name>`, `<name>_<action>`, `settings_<name>_*`.
- FSM-состояния в одном `StatesGroup` на навык.
 
### 13.9 Shared Profile (`skills/services/`)
 
- `shared_profile.py` — общие параметры (рост, вес, цели) между навыками
- `parameter_registry.py` — регистрация параметров
- `profiles_aggregator.py` — агрегация профилей
 
---
 
## 14. Поисковый агент (LangGraph)
 
### 14.1 Архитектура (`app/search_agent/`)
 
LangGraph state machine с 6 стадиями (criticizing отключена):
 
```
PLANNING → REFORMULATING → SEARCHING → PROCESSING → (CRITICIZING) → AGGREGATING → END
```
 
### 14.2 Граф (`search_graph.py`)
 
```python
def create_search_graph(status_callback=None):
    graph = StateGraph(AgentInput)
    graph.add_node(AgentState.PLANNING, planning)
    graph.add_node(AgentState.REFORMULATING, reformulating)
    graph.add_node(AgentState.SEARCHING, searching)
    graph.add_node(AgentState.PROCESSING, processing)
    graph.add_node(AgentState.AGGREGATING, aggregating)
    # edges: linear pipeline
    graph.set_entry_point(AgentState.PLANNING)
    return graph.compile()
```
 
### 14.3 Стадии (`graph_nodes.py`, ~600 строк)
 
**1. Planning** — LLM (gpt-4o-mini) декомпозирует запрос на подзадачи (SubTask[])
- Учитывает последние 5 сообщений контекста
- Output: `state.subtasks`
 
**2. Reformulating** — генерация альтернативных формулировок для каждой подзадачи
- До `MAX_SEARCH_QUERIES` вариантов
 
**3. Searching** — DuckDuckGo (DDGS) + парсинг URL
```python
search_engine = DDGS()
results = search_engine.text(query)
for result in results[:MAX_SEARCH_RESULTS]:
    content = web_parser.parse_url(result['href'])
    images = web_parser.extract_images(result['href'])
```
 
**4. Processing** — GPT-4o-mini извлекает ключевую информацию, агрегирует ответы подзадач
 
**5. Criticizing** — (отключена) критический анализ качества
 
**6. Aggregating** — финальный GPT-проход: когерентный ответ + встроенные ссылки
 
### 14.4 Типы данных (`agent_types.py`)
 
```python
class AgentState(str, Enum):
    PLANNING, REFORMULATING, SEARCHING, PROCESSING, CRITICIZING, AGGREGATING
 
class DialogMessage(BaseModel):
    query: str
    response: str
 
class SearchQuery(BaseModel):
    original_query: str
    reformulated_queries: List[str] = []
 
class SubTask(BaseModel):
    id: str
    description: str
    query: SearchQuery
    results: List[SearchResult] = []
    status: str = "pending"
 
class AgentInput(BaseModel):
    user_query: str
    context: List[DialogMessage] = []
    subtasks: List[SubTask] = []
    aggregated_response: Optional[str] = None
    references: List[str] = []
```
 
### 14.5 Web Parser (`web_parser.py`)
 
```python
class WebParser:
    def parse_url(url) -> Optional[str]:
        # BeautifulSoup, удаляет script/style/nav/header/footer
        # Извлекает блоки текста ≥ 70 chars
    def extract_images(url) -> List[Dict[str, str]]:
        # Фильтр: нет webp/svg/logo/icon/sprite/pixel
        # Приоритет: og:image, twitter:image
    def clean_url(url) -> str:
        # Удаляет tracking параметры
```
 
---
 
## 15. Agent Execution (MCP)
 
### 15.1 Архитектура (`app/agent_execution/`)
 
Multi-task LangGraph агент с MCP tool calling.
 
### 15.2 Граф (`agent_graph.py`)
 
```python
class AgentState(TypedDict):
    messages: Annotated[List[BaseMessage], operator.add]
    current_task: Optional[TaskDefinition]
    current_task_index: int
    is_complete: bool
    all_tasks: List[TaskDefinition]
 
mcp_servers = {
    "web_tools": {
        "command": "python",
        "args": ["-m", "app.mcp_servers.web_tools"],
        "transport": "stdio",
    },
}
 
# Nodes: agent_node (LLM) ↔ tools_node (MCP execution)
# Condition: tools_condition → "tools" | "end"
# Model: ChatOpenAI o3-mini with tool binding
```
 
### 15.3 Task Definition (`types.py`)
 
```python
class TaskDefinition(BaseModel):
    id: str
    name: str
    description: str
    requires_user_input: bool = True
    tools: List[str] = []
    goal: str = ""
 
class AgentState(BaseModel):
    plan_id: str
    user_id: str
    goal: str
    tasks: List[TaskDefinition]
    current_task_index: int = 0
    messages: List[BaseMessage] = []
    results: Dict[str, Any] = {}
```
 
### 15.4 GPT Service (`gpt_service.py`)
 
```python
class GPTService:
    async def chat(messages, model="o3-mini") -> str
    async def analyze_food_image(file_path) -> str  # GPT Vision
```
 
---
 
## 16. Система уведомлений
 
### 16.1 NotificationManager (`notifications/manager.py`)
 
```python
class NotificationManager:
    _bot_integration: NotificationBotIntegration
    _alarm_service: AlarmNotificationService
    _meal_service: MealNotificationService
    _weekly_nutrition_service: WeeklyNutritionService
 
    def initialize(self, bot: Bot) -> AlarmNotificationService
    def start() → scheduler.start()
    def stop() → scheduler.shutdown()
 
notification_manager = NotificationManager()  # глобальный
```
 
### 16.2 APScheduler (`notifications/scheduler.py`)
 
```python
class NotificationScheduler:
    scheduler = AsyncIOScheduler(
        jobstores={'default': MemoryJobStore()},
        executors={'default': AsyncIOExecutor()},
        job_defaults={'coalesce': False, 'max_instances': 3},
        timezone='UTC'
    )
 
    def add_alarm_notifications(user_id, alarm_id, alarm_time, weekday, gmt_offset):
        # CronTrigger с UTC-коррекцией
        # Два джоба: sleep_reminder (за 8ч) + alarm_callback
 
notification_scheduler = NotificationScheduler()  # глобальный
```
 
### 16.3 Сервисы уведомлений
 
- **AlarmNotificationService** — будильники и напоминания о сне
- **MealNotificationService** — напоминания о приёмах пищи
- **WeeklyNutritionService** — еженедельные отчёты по питанию
 
### 16.4 Bot Integration (`notifications/bot_integration.py`, `enhanced_bot_integration.py`)
 
```python
class NotificationBotIntegration:
    def __init__(self, bot: Bot)
    async def send_notification(user_id, text, reply_markup=None) -> bool
    async def send_alarm(user_id, alarm_data) -> bool
 
class EnhancedBotIntegration:
    async def sync_all_alarms()  # Загрузка всех будильников из БД при старте
```
 
---
 
## 17. Confluence интеграция
 
### 17.1 Модули (`services/confluence/`)
 
- `oauth.py` — OAuth 3LO flow, обмен кода на токен
- `client.py` — API Confluence (pages, spaces, CQL search)
- `poller.py` — фоновый sync (периодические CQL запросы)
- `sync.py` — upsert pages → chats + files → RAG index
- `privacy.py` — Personal Data Reporting API
- `parser.py` — HTML → text
- `utils.py` — утилиты
 
### 17.2 OAuth 3LO поток
 
```
1. GET /sources/confluence/oauth/authorize-url → redirect на Atlassian
2. Atlassian → callback ?code=...&state=...
3. Обмен code → access_token + refresh_token
4. Сохранение ConfluenceConnection (refresh_token_enc — зашифрованный)
5. Получение списка cloud-инстанций
```
 
### 17.3 Синхронизация
 
Фоновый поллер каждые `poll_interval_sec=900`:
1. CQL: `type=page AND lastModified >= :cursor`
2. Для каждой страницы:
   - Fetch HTML
   - Upsert ConfluencePageMap
   - Создание/обновление Chat + ChatFileLink
   - Чанкинг + RAG-индексация (Qdrant + ES)
 
### 17.4 Модели
 
```
ConfluenceConnection     — пользователь × cloud (OAuth, статус, last_error)
ConfluenceSpaceSub       — подписки на пространства (root_chat_id, space_chat_id)
ConfluencePageMap        — страница ↔ чат (page_id → page_chat_id, file_id)
ConfluenceChatSpaceSub   — привязка к чат-деревьям
ConfluenceChatPageMap    — маппинг для чат-контекстов
```
 
---
 
## 18. Шеринг чатов
 
### 18.1 Создание токена (`services/share_tokens.py`)
 
```python
def issue_share_token(owner_tg_id, root_chat_id) -> str:
    # Бинарный формат: [ver:1][owner:8][chat:8][nonce:5][mac:12]
    # Base64url: ~44 символов
    # HMAC-SHA256, stateless (без БД)
 
def verify_share_token(token) -> Tuple[int, int]:
    # → (owner_tg_id, root_chat_id)
```
 
### 18.2 Импорт (`services/share_service.py`)
 
```python
async def clone_subtree_ready_files(session, owner_tg_id, root_chat_id, target_user_tg_id) -> int:
    # 1. Список всех чатов в поддереве
    # 2. Клонирование структуры дерева
    # 3. Клонирование файлов с parse_status=ready AND embed_status=ready
    # 4. Переиспользование FileBlob (без повторного хранения)
    # 5. Клонирование Qdrant-векторов (clone_file_points — новые point ID)
    # 6. Создание ChatSharedMap (source → target, mode=readonly)
    # 7. Return new_root_id
```
 
---
 
## 19. LLM провайдеры и Model Registry
 
### 19.1 Архитектура (`services/llm/`)
 
- `registry.py` — ModelRegistry + parse_model_ref()
- `types.py` — LLMModelSpec, ChatCompletionResult, ToolCall, ChatMessage
- `openai_provider.py` — OpenAI async streaming
- `fal_openrouter_provider.py` — FAL/OpenRouter
- `provider_base.py` — базовый класс провайдера
- `exceptions.py` — LLMProviderError
- `json_utils.py` — утилиты парсинга JSON из LLM
 
### 19.2 LLMModelSpec (`types.py`)
 
```python
@dataclass(frozen=True)
class LLMModelSpec:
    provider: str              # "openai" | "fal"
    model_id: str
    display_name: Optional[str]
    context_window: Optional[int]
    max_output_tokens: Optional[int]
    supports_temperature: bool = True
    supports_streaming: bool = True
    supports_force_json: bool = False
    supports_image_input: bool = False
    supports_pdf_input: bool = False
    supports_image_output: bool = False
    supports_tools: bool = False
    supports_reasoning: bool = False
    supports_structured_output: bool = False
    media_tasks: FrozenSet[str] = frozenset()
    media_default_params: Dict = {}
    media_supported_params: Dict = {}
```
 
### 19.3 Model Registry (`registry.py`)
 
```python
class ModelRegistry:
    def upsert(spec: LLMModelSpec) -> None
    def get(provider, model_id) -> Optional[LLMModelSpec]
    def list() -> List[LLMModelSpec]
    def list_by_capability(capability) -> List[LLMModelSpec]
        # "chat", "image_input", "image_output", "tools", "reasoning", "structured_output"
        # Несколько через запятую: "chat,image_input" → AND
 
def parse_model_ref(model: str, default_provider="openai") -> ParsedModelRef:
    # Форматы:
    # "openai:gpt-5-nano" → (openai, gpt-5-nano)
    # "fal:google/gemini-3-flash" → (fal, google/gemini-3-flash)
    # "gpt-5-nano" → (openai, gpt-5-nano)
    # "google/gemini-3-flash" → (fal, ...) — эвристика по "/"
    # "flux" → алиас → (fal, flux)
```
 
### 19.4 Зарегистрированные модели
 
**OpenAI (direct):**
- gpt-5.2-chat-latest (400K ctx, 128K out)
- gpt-5-mini (400K ctx)
- gpt-5-nano (400K ctx) — **default**
- gpt-image-1 (image generation)
 
**Через FAL/OpenRouter:**
- Claude Opus 4.6, Sonnet 4.6, Haiku 4.5 (Anthropic)
- Gemini 3.1 Pro, 3 Flash (Google)
- DeepSeek V3.2, R1
- Qwen 3, Qwen 3 Thinking, Qwen 3 Coder Next
- Llama 4 Scout, Maverick (Meta)
- Grok Code Fast 1, Grok 4.1 Fast (xAI)
- MiniMax M2.5
- GLM 5 (Zhipu)
- FLUX (image generation)
 
### 19.5 Model Client (`services/model_client.py`)
 
```python
class ModelClient:
    def effective_model_ref(chat_model) -> str
        # null/default → LLM_DEFAULT_MODEL (openai:gpt-5-nano)
 
    async def chat_completion(messages, model, temperature, max_tokens, tools, call_site) -> str
    async def chat_completion_stream(messages, model, ...) -> AsyncGenerator[str, None]
```
 
### 19.6 Промптинг (`services/prompting_*.py`)
 
- `prompts.py` — системные промпты
- `prompting_router.py` — выбор промпта по контексту чата
- `prompting_packs.py` — `build_capabilities_block()`, `build_context_system_message()`, `truncate_history()`
- `prompting_types.py` — типы
 
---
 
## 20. Настройки и конфигурация
 
### 20.1 Web API Settings (`db/web_api/core/settings.py`, 300+ строк)
 
Все через pydantic BaseSettings + os.getenv():
 
**Сервер:**
- `WEBAPI_PORT=8001`, `JWT_SECRET`, `CORS_ALLOWED_ORIGINS`, `CORS_ALLOWED_ORIGIN_REGEX`
 
**JWT/Auth:**
- `ACCESS_TOKEN_TTL_MINUTES=60`, `REFRESH_TOKEN_TTL_DAYS=30`, `BOT_TOKEN`
 
**RAG Retrieval:**
- `RAG_ENABLED=true`, `RAG_SCORE_THRESHOLD=0.2`, `RAG_MMR_LAMBDA=0.5`
- `RAG_MULTIQUERY_NUM=3`, `RAG_MULTIQUERY_HISTORY_LAST_N=6`
- `RAG_REFORMULATE_ENABLED=false`, `RAG_CANDIDATES_PER_QUERY=20`
- `RAG_NEIGHBORS_EACH_SIDE=1`, `RAG_TOP_K=30`
- `RAG_MAX_CONTEXT_CHARS=10000`, `RAG_MAX_CHUNKS_PER_FILE=30`
 
**Эмбеддинги:**
- `RAG_EMBEDDINGS_PROVIDER=fastembed`
- `RAG_EMBEDDING_MODEL_NAME=sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`
- `RAG_EMBEDDINGS_BATCH_SIZE=128`, `RAG_NUM_THREADS=8`
- `RAG_EMBEDDINGS_QUANTIZE_DYNAMIC=true`, `RAG_EMBEDDINGS_WARMUP=true`
- `RAG_EMBEDDINGS_CACHE_ENABLED=true`, `RAG_EMBEDDINGS_CACHE_PATH=db/web_api/data/emb_cache.sqlite`
- `RAG_EMBEDDING_MAX_SEQ_LEN_QUERY=256`, `RAG_EMBEDDING_MAX_SEQ_LEN_PASSAGE=384`
 
**Sparse/Hybrid:**
- `RAG_SPARSE_ENABLED=true`, `RAG_SPARSE_PROVIDER=elasticsearch`, `RAG_SPARSE_TOP_K=150`
- `RAG_HYBRID_ENABLED=true`, `RAG_HYBRID_RRF_K=60`
- `RAG_HYBRID_SPARSE_WEIGHT=1.0`, `RAG_HYBRID_DENSE_WEIGHT=1.0`
- `RAG_ES_URL=http://localhost:9200`, `RAG_ES_INDEX=rag_chunks`
 
**Qdrant:**
- `QDRANT_URL=http://localhost:6333`, `QDRANT_API_KEY`, `QDRANT_COLLECTION=rag_chunks`
 
**S3/MinIO:**
- `S3_ENDPOINT`, `S3_PUBLIC_ENDPOINT`, `S3_REGION=us-east-1`, `S3_BUCKET=neuronium`
- `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_SECURE=true`, `S3_FORCE_PATH_STYLE=true`
- `S3_PRESIGN_EXPIRES_SEC=900`, `S3_MAX_POOL_CONNECTIONS=50`
 
**Redis:**
- `REDIS_URL=redis://redis:6379/0`, `REDIS_MAX_CONNECTIONS=50`, `REDIS_SOCKET_TIMEOUT_SEC=15`
- Streams: `INGEST_STREAM=ingest:embed_jobs`, `INGEST_FILE_STREAM=ingest:file_jobs`
- `FILE_EVENTS_STREAM=events:file_status`, `UPLOAD_PARSE_STREAM=uploads:parse_jobs`
- `UPLOAD_EVENTS_STREAM=events:upload_status`, `MEDIA_JOBS_STREAM=media:jobs`, `MEDIA_EVENTS_STREAM=events:media`
 
**Файлы:**
- Deny: exe, dll, zip, rar, 7z, iso, dmg...
- Allow exts: txt, md, csv, json, py, js, ts, go, rs...
- `FILE_SNIFF_PROBE_BYTES=4096`, `FILE_SNIFF_MIN_TEXT_FRACTION=0.85`
- `RAG_MAX_CHARS_PER_CHUNK=2000`, `RAG_MAX_FILE_SIZE_MB=5120`
 
**Chat Uploads:**
- Images: `CHAT_IMAGES_MAX_PER_MESSAGE=4`, `CHAT_IMAGES_MAX_SIZE_MB_EACH=20`, `CHAT_IMAGES_MAX_TOTAL_MB=40`
- Files: `CHAT_FILES_MAX_PER_MESSAGE=5`, `CHAT_FILES_MAX_SIZE_MB_EACH=50`
- `CHAT_FILES_MAX_CONTEXT_CHARS=10000`
- GC: `CHAT_UPLOADS_GC_ENABLED=true`, `CHAT_UPLOADS_GC_INTERVAL_SEC=300`
 
**Агент/Reasoning:**
- `REASONING_ENABLED=true`, `REASONING_THROTTLE_MS=80`, `TOKEN_THROTTLE_MS=0`
- `AGENT_MAX_STEPS=6`
 
**LLM:**
- `LLM_DEFAULT_PROVIDER=openai`, `LLM_DEFAULT_MODEL=openai:gpt-5-nano`
- `FAL_OPENROUTER_BASE_URL=https://fal.run/openrouter/router/openai/v1`
- `LLM_TRACE_PROMPTS=false`, `LLM_TRACE_MAX_CHARS=20000`
 
**Auto-title:**
- `AUTO_TITLE_ENABLED=true`, `AUTO_TITLE_MODEL=gpt-5-nano`, `AUTO_TITLE_TIMEOUT_MS=20000`
 
**SSE:**
- `SSE_QUEUE_MAXSIZE=500`, `SSE_KEEP_ALIVE_SEC=15`
- `SSE_DROP_OLDEST_LIMIT=200`, `SSE_DISCONNECT_AFTER_DROPS=500`
- `SSE_REPLAY_MAX_ENTRIES=1000`, `SSE_REDIS_XREAD_BLOCK_MS=5000`
 
**Media:**
- `MEDIA_JOBS_STREAM=media:jobs`, `MEDIA_MAX_CONCURRENT_TASKS=2`
- `MEDIA_DEFAULT_PROVIDER=openai`, `MEDIA_DEFAULT_MODEL_ID=gpt-image-1`
 
**Confluence:**
- `ATLASSIAN_OAUTH_CLIENT_ID`, `CLIENT_SECRET`, `REDIRECT_URL`
- `CONFLUENCE_TOKEN_ENC_KEY`
 
**Service:**
- `SERVICE_TOKEN` (Bearer для GPU worker)
- `INGEST_CALLBACKS_SECRET` (HMAC для коллбэков)
 
**Langfuse:**
- `LANGFUSE_ENABLED=false`, `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`
 
### 20.2 Bot Config (`app/config.py`)
 
- `BOT_TOKEN`, `OPENAI_API_KEY`, `FAL_KEY`
- `TELEGRAPH_ACCESS_TOKEN`, `TELEGRAPH_AUTHOR_NAME`, `TELEGRAPH_ENABLED`
- `TEST_USER_TG_NICKNAMES`
- `DB_API = http://{DB_API_HOST}:{DB_API_PORT}/api/v1`
 
---
 
## 21. Зависимости и версии
 
### 21.1 Core (`requirements/base.txt`)
 
SQLAlchemy 2.0.36, asyncpg 0.30.0, psycopg2 2.9.10, Pydantic 2.9.2, pydantic-settings 2.6.1, Alembic 1.14.0, python-dotenv 1.0.1
 
### 21.2 Web API (`requirements/web-api.txt`)
 
FastAPI 0.115.4, Uvicorn 0.32.0, Starlette 0.41.2, Redis ≥5.0.0, qdrant-client ≥1.7.0 <2.0.0, elasticsearch ≥8.13.0 <9.0.0, numpy, fastembed, OpenAI 1.54.3, fal-client, PyJWT 2.9.0, boto3, aiobotocore, beautifulsoup4, chardet, lxml, langfuse
 
### 21.3 Bot (`requirements/bot.txt`)
 
aiogram 3.14.0, aiofiles, APScheduler 3.10.4, httpx, openai 1.54.3, langchain, langchain-openai, langgraph, langchain-mcp-adapters, mcp, ddgs, tavily-python, beautifulsoup4, requests, telegraph, pydantic
 
### 21.4 GPU Worker (`gpu_worker/requirements.txt`)
 
sentence-transformers ≥2.6.0, torch ≥2.1.0, qdrant-client ≥1.9.0, redis ≥5.0.0, httpx ≥0.25.0, pydantic ≥2.6.0, numpy, hf_transfer
 
### 21.5 Ingest Worker (`requirements/ingest.txt`)
 
redis, httpx, chardet, beautifulsoup4, lxml, pypdf, python-docx, pandas, openpyxl, xlrd, pymupdf, docx2txt, razdel, boto3, aiobotocore, qdrant-client, numpy, fastembed
 
---
 
## 22. Диаграмма взаимодействия компонентов
 
```
┌──────────────────────────────────────────────────────────────────────────┐
│                        NEURONIUM SYSTEM FLOW                             │
└──────────────────────────────────────────────────────────────────────────┘
 
TELEGRAM BOT (app/)
├→ HTTP → db-api (internal)      │ SkillDataAPI — профили, уведомления
├→ HTTP → web-api                │ Chat API, LLM streaming
└→ MCP stdio → web_tools         │ Tool calling
 
WEB API (db/web_api/) — Main Hub
├→ PostgreSQL (async)            │ Все ORM-модели
├→ Redis Streams                 │ ingest:file_jobs, ingest:embed_jobs,
│                                │ media:jobs, uploads:parse_jobs,
│                                │ events:file_status, events:upload_status, events:media
├→ Qdrant (gRPC/HTTP)            │ rag_chunks (384-dim vectors)
├→ Elasticsearch                 │ rag_chunks (BM25)
├→ MinIO/S3                      │ Файлы, presigned URLs
├→ OpenAI API                    │ LLM streaming, images
├→ FAL/OpenRouter                │ Alternative LLMs (Claude, Gemini, etc.)
└→ Atlassian API                 │ Confluence OAuth + CQL
 
FILE INGEST WORKER
├→ Redis consumer (ingest:file_jobs)
├→ S3 download → parse → FileChunk
└→ Redis producer (ingest:embed_jobs)
 
GPU WORKER (external)
├→ Redis consumer (ingest:embed_jobs, XREADGROUP)
├→ HTTP → web-api (GET chunks)
├→ SentenceTransformer encode
├→ Qdrant gRPC upsert
└→ HTTP → web-api (HMAC callback)
 
MEDIA WORKER
├→ Redis consumer (media:jobs)
├→ OpenAI/FAL API → generate
├→ S3 upload result
└→ Redis events → SSE
 
UPLOADS PARSE WORKER
├→ Redis consumer (uploads:parse_jobs)
├→ Parse → text
└→ Redis events → SSE
```
 
### Таблица межкомпонентной коммуникации
 
| От | К | Протокол | Auth | Назначение |
|----|---|----------|------|------------|
| Bot → db-api | HTTP | N/A (internal) | User data, settings |
| Bot → web-api | HTTP | JWT | Chat API, LLM |
| Bot → MCP web_tools | stdio | N/A | Tool calling |
| web-api → PostgreSQL | psycopg2 | User/pass | Persistent storage |
| web-api → Redis | TCP | REDIS_PASSWORD | Streams, cache |
| web-api → Qdrant | gRPC/HTTP | API_KEY | Vector search |
| web-api → Elasticsearch | HTTP | N/A | BM25 search |
| web-api → MinIO/S3 | S3 API | Access key | File storage |
| web-api → OpenAI | HTTPS | OPENAI_API_KEY | LLM streaming |
| web-api → FAL | HTTPS | FAL_KEY | Alternative LLMs |
| gpu_worker → Redis | TCP | REDIS_PASSWORD | Streams consumer |
| gpu_worker → web-api | HTTPS | SERVICE_TOKEN | Chunks + callbacks |
| gpu_worker → Qdrant | gRPC | API_KEY | Vector upsert |
| ingest → Redis | TCP | REDIS_PASSWORD | Streams |
| ingest → PostgreSQL | psycopg2 | User/pass | FileChunk creation |
| media → Redis | TCP | REDIS_PASSWORD | Streams |
| media → OpenAI/FAL | HTTPS | API keys | Image generation |
 
---
 
*Документ сгенерирован автоматически на основе анализа кодовой базы. Дата: 2026-03-22.*
