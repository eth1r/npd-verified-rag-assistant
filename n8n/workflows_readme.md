# n8n Workflows README

Проект: NPD Verified RAG Assistant

## Статусы workflow

### Актуальные

```text
NPD Ingestion v1
NPD Evaluation Runner v0.3
NPD Telegram Polling Assistant
```

### Архивные

```text
NPD Retrieval Test v1
```

Архивные workflow сохранены только для истории разработки и воспроизводимости промежуточных этапов. Они не должны использоваться как основной контур проекта и могут не соответствовать текущей базе знаний, промптам и retrieval-логике.

## 1. NPD Ingestion v1 — актуальный служебный workflow

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

## 2. NPD Evaluation Runner v0.3 — актуальный evaluation workflow

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

RAW-поиск получает до 10 результатов. В контекст передаются первые 5 результатов и до 3 дополнительных чанков из наиболее релевантных документов. Mini-Wiki используется только как вспомогательный слой и не может быть источником.

### Проверка источников

- каждая пара `[doc_id / chunk_id]` должна присутствовать в текущем RAW-контексте;
- разрешены только документы `npd_*`;
- `wiki_*`, `glossary`, `fallback_rules` и `negative_scenarios` запрещены как источники;
- при отсутствии валидного источника ответ заменяется строгим fallback.

### Результат финального прогона

```text
50 вопросов
94% успешных ответов
100% корректных fallback для внебазовых и рискованных запросов
100% содержательных ответов с проверяемыми источниками
```

## 3. NPD Telegram Polling Assistant — актуальный пользовательский workflow

Назначение: основной пользовательский интерфейс MVP в Telegram.

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
- программная проверка источников;
- строгий fallback;
- safety guard;
- память короткого диалога;
- команда `/reset`;
- дневной лимит запросов;
- polling вместо webhook.

## 4. NPD Retrieval Test v1 — АРХИВ

Статус: **архивный, не использовать для финальной оценки и пользовательского контура**.

Назначение на этапе разработки: вручную проверить ранний retrieval/generation-контур на одном тестовом вопросе.

Причины архивного статуса:

- рассчитан на одиночную ручную проверку;
- предшествует финальному разделению RAW и Mini-Wiki;
- не отражает текущую source-verification логику;
- не используется для финального прогона 50 вопросов;
- заменён `NPD Evaluation Runner v0.3` и финальным Telegram-workflow.

Подробная политика по архивным workflow находится в:

```text
n8n/workflows/archive/README.md
```

## Safety guard

Safety guard блокирует запросы, связанные со скрытием доходов, уклонением от налогов, занижением дохода, фиктивными чеками и обходом ограничений НПД.

## Verified RAG rules

- фактические ответы формируются только по `RAW_CONTEXT`;
- Mini-Wiki не используется как доказательство;
- внешние знания и неподтверждённые выводы запрещены;
- каждый фактический ответ должен содержать проверяемый RAW-источник;
- для fallback и safety-отказов источники запрещены;
- источники дополнительно проверяются программной логикой.

## Требуемые credentials

После импорта актуальных workflow необходимо настроить:

- OpenAI HTTP Header Auth;
- Qdrant HTTP Header Auth;
- Telegram Bot API token;
- Google Sheets OAuth2 для Evaluation Runner.

Подробности:

```text
n8n/credentials_required.md
n8n/qdrant_setup.md
```
