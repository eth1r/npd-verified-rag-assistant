# n8n Workflows README

Проект: NPD Verified RAG Assistant

## Workflows

В MVP используются два основных workflow:

```text
NPD Ingestion v1
NPD Retrieval Test v1
```

Экспортированные JSON-файлы находятся в:

```text
n8n/workflows/
```

## 1. NPD Ingestion v1

Назначение: загрузить raw Markdown-документы из GitHub, разбить их на chunks, создать embeddings и загрузить points в Qdrant.

### Логика

```text
Manual Trigger
→ Get documents registry
→ Parse documents registry
→ Filter retrieval eligible
→ Build raw document URL
→ Get raw markdown document
→ Clean and chunk markdown
→ Limit test chunk
→ OpenAI embeddings
→ Build Qdrant point
→ Build Qdrant upsert batches
→ Qdrant upsert points
```

### Что делает workflow

1. Скачивает `documents_registry.csv`.
2. Оставляет только документы с `retrieval_eligible = true`.
3. Скачивает raw Markdown-документы.
4. Извлекает metadata из YAML frontmatter.
5. Удаляет служебные разделы, которые не должны попадать в retrieval.
6. Делит документы на chunks по заголовкам `##` и `###`.
7. Создаёт embeddings через OpenAI.
8. Формирует Qdrant points.
9. Загружает points в коллекцию `npd_knowledge_v1`.

### Chunking

Используется стратегия:

```text
split by Markdown headings ## / ###
max size: около 3500 символов
overlap: около 300 символов
```

Служебные разделы не индексируются:

```text
## Назначение документа
## Что ассистент может отвечать
## Что ассистент не должен
## Источник
## Источники
```

### Ожидаемый результат

После успешного ingestion:

```text
points_count: 140
```

## 2. NPD Retrieval Test v1

Назначение: проверить retrieval/generation контур на тестовом вопросе.

### Логика

```text
Manual Trigger
→ Set test question
→ Safety guard
→ IF route = blocked?
   ├── TRUE  → Format blocked answer
   └── FALSE → OpenAI query embedding
              → Build Qdrant search body
              → Qdrant search
              → Format search results
              → Build retrieved context
              → OpenAI generate answer
              → Format final answer
```

### Что делает workflow

1. Принимает тестовый вопрос.
2. Проверяет unsafe-запросы через Safety guard.
3. Для обычных вопросов создаёт embedding вопроса.
4. Ищет top-k chunks в Qdrant.
5. Собирает `retrieved_context`.
6. Генерирует ответ через OpenAI только на основе retrieved context.
7. Возвращает финальный ответ, модель и usage tokens.

## Safety guard

Safety guard блокирует запросы, связанные с:

```text
скрытием доходов
уклонением от налогов
неуплатой налога
занижением дохода
фиктивными или поддельными чеками
```

Такие запросы не отправляются в Qdrant и OpenAI generation.

Ожидаемый ответ:

```text
Я не могу помогать со скрытием доходов, уклонением от налогов или нарушением правил НПД. Могу объяснить, как правильно отражать доходы, формировать чеки и платить налог на профессиональный доход.
```

## Verified RAG rules

System prompt в generation node требует:

- отвечать только по `retrieved_context`;
- не использовать внешние знания;
- не выдумывать;
- указывать источники только для фактических ответов;
- не указывать источники для fallback-ответов;
- игнорировать нерелевантные retrieved chunks.

## Golden Dataset

Результаты ручного тестирования сохранены в:

```text
tests/golden_dataset_results_v0.1.md
```

Статус:

```text
Golden Dataset v0.1: PASS
```
