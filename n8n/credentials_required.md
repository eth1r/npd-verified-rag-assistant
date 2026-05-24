# n8n Credentials Required

Проект: NPD Verified RAG Assistant  
Назначение: список credentials, необходимых для запуска MVP workflow в n8n.

## Важно

В репозитории не хранятся API-ключи, токены, пароли, proxy-данные и другие секреты.

Все секреты должны храниться только в n8n Credentials или в `.env` на сервере.

## Required credentials

### 1. OpenAI Header Auth - NPD MVP

Используется в workflow:

- `NPD Ingestion v1`
- `NPD Retrieval Test v1`

Тип credential в n8n:

```text
Generic Credential Type → Header Auth
```

Поля:

```text
Header Name: Authorization
Header Value: Bearer <OPENAI_API_KEY>
```

Где используется:

- создание embeddings через `POST https://api.openai.com/v1/embeddings`;
- генерация ответа через `POST https://api.openai.com/v1/chat/completions`.

Используемые модели:

```text
Embeddings: text-embedding-3-small
Answer model: gpt-4.1-mini
```

---

### 2. Qdrant Header Auth - NPD MVP

Используется в workflow:

- `NPD Ingestion v1`
- `NPD Retrieval Test v1`

Тип credential в n8n:

```text
Generic Credential Type → Header Auth
```

Поля:

```text
Header Name: api-key
Header Value: <QDRANT_API_KEY>
```

Где используется:

- загрузка points в Qdrant;
- поиск по коллекции Qdrant.

Qdrant URL внутри Docker-сети:

```text
http://qdrant:6333
```

Коллекция:

```text
npd_knowledge_v1
```

---

## Proxy

Если OpenAI API вызывается через proxy, proxy указывается в настройках HTTP Request node:

```text
Options → Proxy
```

Формат:

```text
http://LOGIN:PASSWORD@PROXY_IP:PROXY_PORT
```

Proxy-данные не должны попадать в экспортированные workflow JSON.

---

## Проверка перед коммитом workflow

Перед добавлением workflow JSON в GitHub проверить отсутствие секретов:

```cmd
findstr /S /I "sk- api-key Bearer proxy password QDRANT_API_KEY OPENAI_API_KEY" n8n\workflows\*.json
```

Если команда ничего не выводит или показывает только названия credential без реальных ключей — workflow можно коммитить.

Если найден реальный API key, proxy password или `Bearer sk-...`, файл нельзя коммитить до очистки.
