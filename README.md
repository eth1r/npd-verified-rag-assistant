# Проверяемый RAG-ассистент для самозанятых

MVP-проект AI-ассистента по вопросам налога на профессиональный доход (НПД).

## Цель

Создать справочного ассистента, который отвечает на типовые вопросы начинающих самозанятых только на основе проверенной базы знаний и указывает источники.

## Технический стек MVP

- Интерфейс: Telegram-бот
- Оркестрация: n8n Cloud / self-hosted n8n
- LLM: OpenAI GPT-4.1 mini
- Embeddings: OpenAI `text-embedding-3-small`
- Vector DB: Qdrant
- База знаний: Markdown-файлы
- Реестр документов: `documents_registry.csv`

## Структура

```text
knowledge_base/
  raw_documents/          # active-документы для RAG
  reference_documents/    # справочные документы, не индексируются
  mini_wiki/              # будущий слой объяснений
  metadata/               # documents_registry.csv

prompts/                  # системные промпты
tests/                    # golden dataset
n8n/workflows/            # экспорт n8n workflow
logs/                     # changelog и эксплуатационные заметки
```

## Правило ingestion

В embeddings и Qdrant попадают только документы, у которых в `documents_registry.csv`:

```text
status = active
retrieval_eligible = true
```

Документы со статусами `reference_only`, `pending_review`, `excluded`, `outdated` не индексируются.

## Текущий состав базы

- 17 active raw-документов для RAG
- 1 reference-only документ: карта закона №422-ФЗ
- 1 pending_review документ
- 1 excluded документ

## Следующий этап

1. Создать Mini-Wiki.
2. Подготовить `system_prompt_v1.0.md`.
3. Собрать n8n ingestion workflow.
4. Загрузить active-документы в Qdrant.
5. Подготовить Golden Dataset.
