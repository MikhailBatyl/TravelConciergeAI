# AI-Консьерж путешественника ver2 + MCP 2GIS

## 1. Цель системы

Система проектируется как multi-agent AI travel concierge, который помогает пользователю подбирать места, отели, рестораны и маршруты на основе двух типов данных:

1. **Оперативные геоданные** — поиск локаций, адресов, времени работы и маршрутов через 2GIS.
2. **Семантические знания из отзывов** — рекомендации на основе RAG-поиска по пользовательским отзывам.

Ключевая идея архитектуры: разделить **контур подготовки данных** и **контур пользовательского inference**, чтобы обновление базы знаний не влияло на latency пользовательского сценария.

---

## 2. Архитектурный принцип

Решение строится как **двухконтурная система**:

### Контур A. Data Pipeline
Назначение:
- сбор отзывов о местах;
- очистка и нормализация данных;
- генерация эмбеддингов;
- загрузка данных в Qdrant;
- регулярное обновление базы знаний.

Оркестратор:
- **Apache Airflow**.

### Контур B. Inference Pipeline
Назначение:
- прием пользовательского запроса;
- маршрутизация запроса внутри multi-agent системы;
- обращение к внешним инструментам;
- выполнение RAG-поиска по отзывам;
- сбор итогового ответа и возврат пользователю.

Оркестратор:
- **n8n**.

---

## 3. High-Level Design

```text
                    ┌──────────────────────────────────────┐
                    │            User Channel              │
                    │ Telegram / Webhook / Chat UI         │
                    └──────────────────┬───────────────────┘
                                       │
                                       ▼
                     ┌───────────────────────────────────┐
                     │     n8n Supervisor Workflow       │
                     │ intent routing / orchestration    │
                     └───────────────┬───────────────────┘
                                     │
             ┌───────────────────────┼────────────────────────┐
             │                       │                        │
             ▼                       ▼                        ▼
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│ Geo Worker           │  │ Review RAG Worker    │  │ Memory / Session     │
│ 2GIS / MCP Tool      │  │ Qdrant retrieval     │  │ Redis or Postgres    │
└───────────┬──────────┘  └───────────┬──────────┘  └──────────────────────┘
            │                         │
            ▼                         ▼
   ┌─────────────────┐       ┌──────────────────────┐
   │ 2GIS API / MCP  │       │ Qdrant Vector Store  │
   └─────────────────┘       └───────────┬──────────┘
                                          │
                                          ▼
                               ┌──────────────────────┐
                               │ Airflow Data Pipeline│
                               │ extract / embed/load │
                               └───────────┬──────────┘
                                           │
                                           ▼
                               ┌──────────────────────┐
                               │ 2GIS Reviews Source  │
                               └──────────────────────┘
```

---

## 4. Multi-Agent архитектура

### 4.1 Supervisor

**Роль:** центральный оркестратор.

**Задачи:**
- принять запрос пользователя;
- определить, какие worker-агенты нужны для ответа;
- вызывать worker-воркфлоу как инструменты;
- агрегировать результаты;
- формировать итоговый JSON-ответ.

**Почему Supervisor нужен:**
- уменьшает перегрузку одного большого агента;
- повышает управляемость и читаемость архитектуры;
- позволяет масштабировать инструменты независимо;
- упрощает observability и трассировку.

### 4.2 Geo Worker

**Роль:** эксперт по геоданным.

**Задачи:**
- поиск мест по названию и категории;
- получение адреса, координат и времени работы;
- маршруты и nearby-search;
- получение оперативной информации из 2GIS.

**Инструменты:**
- MCP 2GIS Tool или HTTP Request / Custom Tool к API 2GIS.

### 4.3 Review RAG Worker

**Роль:** эксперт по пользовательскому опыту и отзывам.

**Задачи:**
- поиск релевантных отзывов в Qdrant;
- извлечение пользовательских сигналов: еда, чистота, сервис, шум, завтраки, цена/качество;
- возврат сжатого factual summary;
- поддержка объяснимой рекомендации.

**Инструменты:**
- Vector Store Tool;
- Qdrant Vector Store;
- embeddings model.

### 4.4 Memory Worker / Session Layer

**Роль:** хранение пользовательского контекста.

**Задачи:**
- удерживать session context;
- сохранять предпочтения пользователя;
- переиспользовать прошлые ограничения и пожелания;
- уменьшать повторные уточнения.

**Реализация:**
- MVP: n8n Memory + PostgreSQL/Redis;
- Production: долговременная память + user profile store.

---

## 5. Текущий MVP №6

Существующий workflow уже содержит базовый каркас single-agent решения:
- вход через Webhook;
- Normalize Input;
- AI Agent;
- OpenAI Chat Model;
- MCP Client для 2GIS;
- Simple Memory;
- Parse JSON;
- запись результата в PostgreSQL;
- возврат ответа через Respond to Webhook.

### Что в нем уже хорошо
- есть входной API-контур;
- есть агентный слой;
- есть интеграция MCP 2GIS;
- есть базовая память;
- есть персистентность через PostgreSQL;
- есть структурированный JSON output.

### Главные ограничения текущего MVP
- один агент перегружен всеми обязанностями;
- нет RAG-контура по отзывам;
- нет отделенного data pipeline;
- нет выделенного supervisor/worker orchestration;
- ограниченная observability;
- нет полноценного ingestion-контура отзывов.

---

## 6. Целевая эволюция MVP → Production

### MVP+ (ближайший этап)

1. Оставить n8n как основной orchestration layer.
2. Разделить текущий workflow на Supervisor и 2 worker-воркфлоу.
3. Добавить RAG worker с Qdrant.
4. Поднять Airflow DAG для обновления отзывов.
5. Добавить метрики и tracing через Langfuse.

### Production-ready этап

1. Вынести ingestion и enrichment в отдельный data pipeline.
2. Сделать устойчивую схему retries / dead-letter / alerting.
3. Добавить schema validation для входа и выхода воркеров.
4. Ввести metadata filtering в Qdrant.
5. Добавить quality evaluation и feedback loop.
6. При необходимости перейти от n8n-only orchestration к гибридной схеме n8n + LangGraph.

---

## 7. Data Pipeline в Airflow

### Цель

Airflow отвечает за наполнение и обновление knowledge base отзывов.

### Базовый DAG

```text
extract_reviews_from_2gis
    ↓
clean_and_normalize_reviews
    ↓
chunk_or_prepare_documents
    ↓
generate_embeddings
    ↓
upsert_to_qdrant
    ↓
write_sync_log
```

### Описание задач

#### extract_reviews_from_2gis
- вызов API или внешнего контура получения отзывов;
- загрузка новых отзывов по списку location_id;
- дедупликация по review_id.

#### clean_and_normalize_reviews
- очистка HTML/мусора;
- нормализация дат, рейтингов, языка;
- подготовка metadata.

#### chunk_or_prepare_documents
- если отзывы короткие, хранить один отзыв как один документ;
- если длинные, применять controlled chunking;
- сохранять связь с исходным местом.

#### generate_embeddings
- OpenAI Embeddings или локальный embedding service;
- batched processing;
- контроль размерности вектора и latency.

#### upsert_to_qdrant
- upsert points;
- payload: location_id, place_name, city, rating, source, review_date, tags;
- коллекция: `travel_reviews`.

#### write_sync_log
- логировать время запуска, число обработанных отзывов, ошибки и skipped records.

---

## 8. RAG-контур

### Цель

RAG нужен для ответов вида:
- «найди отель, где хвалят завтраки»;
- «какое место тихое для работы»;
- «где хороший сервис, но не слишком дорого».

### Логика retrieval

1. Пользовательский запрос попадает в Supervisor.
2. Supervisor делегирует часть задачи Review Worker.
3. Review Worker преобразует запрос в embedding.
4. Выполняется vector search в Qdrant.
5. При необходимости используется metadata filtering.
6. Worker возвращает не сырой результат, а краткое резюме с признаками качества.

### Почему это важно

LLM без RAG будет давать общие туристические советы. RAG по свежим отзывам делает рекомендации grounded, объяснимыми и ближе к реальному пользовательскому опыту.

---

## 9. Интеграция 2GIS

### Вариант 1. MCP 2GIS

Это приоритетный путь, если MCP-сервер стабильно доступен.

**Плюсы:**
- современный инструментальный протокол;
- удобная интеграция в AI Agent tool stack;
- хорошая демонстрация современного AI tooling подхода.

**Минусы:**
- зависимость от стабильности MCP-сервера;
- потребуется аккуратное описание схем инструментов.

### Вариант 2. HTTP Request Tool / Custom Tool

Если MCP-контур не готов, можно использовать прямые запросы к API 2GIS.

**Плюсы:**
- быстрее для MVP;
- проще отлаживать;
- больше контроля над payload и retry policy.

**Минусы:**
- меньше «вау-эффекта» для портфолио;
- потребуется руками проектировать tool schema.

---

## 10. Langfuse и observability

### Что отслеживаем
- latency по каждому workflow;
- количество tool calls;
- token usage;
- стоимость запросов;
- долю успешных JSON-ответов;
- долю fallback-сценариев;
- качество retrieval и relevance retrieved chunks.

### Где внедряем
- в n8n: tracing prompt → tool call → response;
- в Airflow: логирование DAG runs и ошибок ingestion;
- в Postgres/ClickHouse: бизнес-аналитика по пользовательским сценариям.

### Зачем это нужно
- видеть деградацию качества;
- понимать стоимость inference;
- быстро отлаживать сбои инструментов;
- показывать зрелый LLMOps-подход в портфолио.

---

## 11. Хранилища и инфраструктура

### Основные компоненты
- **n8n** — orchestration inference layer;
- **Airflow** — orchestration data layer;
- **Qdrant** — vector database;
- **PostgreSQL** — session storage, analytics, service tables;
- **Redis** — опционально для памяти, кэша и rate control;
- **Langfuse** — prompt management и tracing.

### Docker Compose контур

Планируемый compose-стек:
- n8n;
- airflow-webserver;
- airflow-scheduler;
- airflow-worker (если CeleryExecutor);
- postgres;
- redis;
- qdrant;
- langfuse;
- embedding service (опционально).

---

## 12. Репозиторий GitHub

```text
ai-travel-concierge/
│
├── .github/
├── n8n_workflows/
│   ├── supervisor_workflow.json
│   ├── geo_worker_workflow.json
│   ├── rag_worker_workflow.json
│   └── APPV-MVP-No6_AI-Concierge.json
│
├── airflow_dags/
│   ├── dag_2gis_reviews_rag.py
│   └── plugins/
│
├── infrastructure/
│   ├── docker-compose.yml
│   └── .env.example
│
├── docs/
│   ├── architecture_diagram.png
│   └── sequence_diagrams/
│
└── README.md
```

---

## 13. README: позиционирование для работодателя

README должен продавать проект как системный AI-продукт, а не просто как набор JSON-нод.

### Нужно подсветить
- бизнес-проблему;
- архитектурное разделение ingestion и inference;
- multi-agent паттерн supervisor/workers;
- RAG по свежим отзывам;
- интеграцию геоданных через 2GIS;
- observability через Langfuse;
- Dockerized deployment;
- production-minded подход к надежности.

### Формулировка ценности

Обычные travel-ассистенты дают общий ответ. Этот проект объединяет:
- реальные пользовательские отзывы;
- геоконтекст и маршруты;
- память о предпочтениях;
- агентную маршрутизацию задач;
- объяснимые рекомендации.

---

## 14. Рекомендуемая дорожная карта

### Этап 1. Stabilize MVP
- зафиксировать текущий workflow №6;
- вынести конфиг в `.env`;
- проверить JSON-контракт входа и выхода;
- добавить логирование ошибок.

### Этап 2. Split into workers
- создать Supervisor workflow;
- выделить Geo Worker;
- выделить Review RAG Worker;
- настроить вызов sub-workflows как tools.

### Этап 3. Build Airflow DAG
- реализовать extraction отзывов;
- добавить embeddings;
- загрузить данные в Qdrant;
- настроить расписание и sync logs.

### Этап 4. Add observability
- подключить Langfuse;
- добавить tracing и token metrics;
- логировать ошибки retrieval и tool failures.

### Этап 5. Portfolio packaging
- оформить README;
- добавить архитектурную диаграмму;
- приложить demo-сценарии;
- подготовить docker-compose для локального старта.

---

## 15. Основные технические риски

### 1. Переусложнение агента
Если не разделить роли между Supervisor и Workers, агент начнет путаться между картами, отзывами и памятью.

### 2. Слабый ingestion-контур
Если Airflow не обеспечивает качественное обновление базы, RAG будет быстро деградировать.

### 3. Непрозрачность ответов
Без Langfuse и метрик будет трудно понять, где проседает качество: retrieval, prompt, tool selection или сама модель.

### 4. Плохой metadata design
Если не сохранять полезные payload-поля в Qdrant, потом нельзя будет сделать точную фильтрацию по месту, городу, категории и рейтингу.

---

## 16. Итоговая целевая модель

Финальная система должна работать так:

1. Пользователь задает travel-вопрос.
2. Supervisor понимает, какие источники знаний нужны.
3. Geo Worker достает live-данные через 2GIS.
4. Review Worker достает grounding из отзывов через Qdrant.
5. Memory layer подмешивает пользовательские предпочтения.
6. Supervisor агрегирует результаты.
7. Пользователь получает персонализированный, объяснимый и актуальный ответ.

Именно такая архитектура выглядит убедительно и как MVP для собеседования, и как основа для дальнейшего production-развития.
