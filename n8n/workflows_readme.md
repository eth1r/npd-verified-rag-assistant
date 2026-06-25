# n8n Workflows README

Проект: NPD Verified RAG Assistant

## Назначение

В каталоге `n8n/workflows/` хранятся экспортированные workflow проекта. Они разделены по назначению: загрузка базы знаний, ручная проверка retrieval-контура, пакетная оценка качества и пользовательский Telegram-интерфейс.

## Актуальные workflow

```text
NPD Ingestion v1
NPD Retrieval Test v1
NPD Evaluation Runner v0.3
NPD Telegram Polling Assistant
```

## 1. NPD Ingestion v1

Назначение: загрузить документы базы знаний, разбить их на чанки, создать embeddings и записать points в Qdrant.

### Логика

```text
Manual Trigger
→ Get documents registry
→ Parse documents registry
→ Filter retrieval eligible
→ Build raw document URL
→ Get raw markdown document
→ Clean and chunk markdown
→ OpenAI embeddings
→ Build Qdrant point
→ Build Qdrant upsert batches
→ Qdrant upsert points
```

### Что делает workflow

1. Загружает реестр документов.
2. Оставляет документы со статусом `active` и `retrieval_eligible = true`.
3. Скачивает Markdown-документы.
4. Извлекает metadata из YAML frontmatter.
5. Удаляет служебные разделы, которые не должны попадать в retrieval.
6. Делит документы на чанки по структуре Markdown.
7. Создаёт embeddings через OpenAI `text-embedding-3-small`.
8. Формирует Qdrant points.
9. Загружает данные в коллекцию `npd_knowledge_v1`.

### Актуальное состояние коллекции

```text
36 документов
258 чанков
184 RAW-чанка
74 Mini-Wiki-чанка
```

Состав базы:

- 25 RAW-документов — доказательная база;
- 8 тематических Mini-Wiki-документов — вспомогательный слой;
- 3 служебных документа — `glossary`, `fallback_rules`, `negative_scenarios`.

## 2. NPD Retrieval Test v1

Назначение: вручную проверить базовый retrieval/generation-контур на одном тестовом вопросе.

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

Этот workflow использовался на раннем этапе разработки для локальной проверки одного вопроса. Он сохранён как промежуточная версия и не является основным пользовательским контуром.

## 3. NPD Evaluation Runner v0.3

Файл:

```text
n8n/workflows/NPD_Evaluation_Runner_v0.3_sanitized.json
```

Назначение: пакетное тестирование RAG-системы на контрольном наборе из 50 вопросов.

### Основная логика

```text
Manual Trigger
→ Read golden dataset
→ Limit test questions: 50
→ Attach run id
→ Safety guard
→ IF blocked?
   ├── TRUE  → Format blocked answer
   └── FALSE → OpenAI query embedding
              → Qdrant RAW search
              → Qdrant Mini-Wiki search
              → Build RAW context
              → Build Mini-Wiki context
              → Merge
              → OpenAI generate answer
              → Programmatic source verification
→ Prepare answer for judge
→ OpenAI judge answer
→ Format judge result
→ Append row in Google Sheets
```

### RAW и Mini-Wiki

Workflow использует два независимых retrieval-контура:

- `RAW_CONTEXT` — единственная доказательная база;
- `MINI_WIKI_CONTEXT` — вспомогательный слой для понимания темы.

Mini-Wiki исключается из RAW-поиска через фильтр `source_type = mini_wiki` и не может использоваться как источник.

### Retrieval

RAW-поиск получает до 10 результатов. В контекст передаются:

- первые 5 результатов;
- до 3 дополнительных чанков из наиболее релевантных документов;
- не более 8 RAW-чанков.

Mini-Wiki-поиск получает до 3 вспомогательных чанков.

### Проверка источников

После генерации ответа выполняется программная проверка:

- каждая пара `[doc_id / chunk_id]` должна присутствовать в текущем RAW-контексте;
- разрешены только документы `npd_*`;
- `wiki_*`, `glossary`, `fallback_rules` и `negative_scenarios` запрещены как источники;
- при отсутствии валидного источника ответ заменяется строгим fallback.

После этого отдельная модель-судья оценивает смысловую корректность ответа и соответствие источников.

### Результат тестирования

Контрольный прогон:

```text
50 вопросов
94% успешных ответов
100% корректных fallback для внебазовых и рискованных запросов
100% содержательных ответов с проверяемыми источниками
```

Файл в репозитории очищен от API-ключей, credential bindings, ID Google-таблицы и внутренних идентификаторов экземпляра n8n. После импорта необходимо заново настроить credentials и Google Sheets.

## 4. NPD Telegram Polling Assistant

Назначение: пользовательский интерфейс проекта в Telegram.

### Основная логика

```text
Schedule / Polling
→ Telegram getUpdates
→ Deduplication
→ Service commands
→ Daily limit
→ Dialogue memory
→ Safety guard
→ RAW retrieval
→ Mini-Wiki retrieval
→ Answer generation
→ Source verification
→ Telegram response
```

### Возможности

- ответы на вопросы по НПД на основе базы знаний;
- отдельные RAW- и Mini-Wiki-контуры;
- проверка источников;
- строгий fallback при отсутствии подтверждения;
- блокировка рискованных запросов;
- память короткого диалога;
- команда `/reset`;
- дневной лимит запросов;
- polling вместо webhook.

Telegram-workflow является основным пользовательским контуром MVP.

## Safety guard

Safety guard блокирует запросы, связанные с:

```text
скрытием доходов
уклонением от налогов
занижением дохода
фиктивными и поддельными чеками
обходом ограничений НПД
```

Такие запросы не передаются в генерационный RAG-контур и получают безопасный отказ.

## Verified RAG rules

Основные правила системы:

- фактические ответы формируются только по `RAW_CONTEXT`;
- Mini-Wiki не используется как доказательство;
- внешние знания и неподтверждённые выводы запрещены;
- каждый фактический ответ должен содержать проверяемый RAW-источник;
- для fallback и safety-отказов источники запрещены;
- источники дополнительно проверяются программной логикой;
- при ошибке проверки ответ заменяется строгим fallback.

## Требуемые credentials

После импорта workflow необходимо настроить:

- OpenAI HTTP Header Auth;
- Qdrant HTTP Header Auth;
- Telegram Bot API token;
- Google Sheets OAuth2 для Evaluation Runner.

Подробности находятся в:

```text
n8n/credentials_required.md
n8n/qdrant_setup.md
```
