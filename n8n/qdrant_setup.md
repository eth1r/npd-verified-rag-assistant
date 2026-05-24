# Qdrant Setup for NPD MVP

Проект: NPD Verified RAG Assistant  
Назначение: описание локального Qdrant для MVP.

## Используемая коллекция

```text
npd_knowledge_v1
```

Параметры коллекции:

```text
Vector size: 1536
Distance: Cosine
on_disk_payload: true
```

Размер вектора `1536` соответствует модели embeddings:

```text
text-embedding-3-small
```

## Docker-сеть

n8n и Qdrant должны находиться в одной Docker-сети, чтобы n8n мог обращаться к Qdrant по имени:

```text
http://qdrant:6333
```

Проверить сети:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Networks}}"
```

Проверить контейнеры в сети:

```bash
docker network inspect n8n_default --format '{{range .Containers}}{{.Name}} {{println}}{{end}}'
```

Ожидаемо в одной сети должны быть:

```text
n8n
qdrant
```

Если n8n не видит `qdrant`, подключить n8n к сети:

```bash
docker network connect --alias n8n n8n_default n8n 2>/dev/null || true
```

Проверить DNS из контейнера n8n:

```bash
docker exec n8n node -e "const dns=require('dns'); dns.lookup('qdrant',(err,address)=>{ if(err){console.error(err); process.exit(1)} console.log(address) })"
```

## Проверка доступности Qdrant из n8n

```bash
QKEY=$(grep QDRANT_API_KEY /opt/apps/qdrant/.env | cut -d= -f2)

docker exec -e QDRANT_API_KEY="$QKEY" n8n node -e "fetch('http://qdrant:6333/collections',{headers:{'api-key':process.env.QDRANT_API_KEY}}).then(r=>r.text()).then(console.log).catch(console.error)"
```

Ожидаемый ответ:

```json
{"result":{"collections":[{"name":"npd_knowledge_v1"}]},"status":"ok"}
```

## Создание коллекции

Если коллекции нет, создать её:

```bash
docker run --rm \
  --network n8n_default \
  --env-file /opt/apps/qdrant/.env \
  --entrypoint sh \
  curlimages/curl:8.8.0 \
  -c '
curl -s -X PUT \
  -H "api-key: $QDRANT_API_KEY" \
  -H "Content-Type: application/json" \
  http://qdrant:6333/collections/npd_knowledge_v1 \
  -d "{\"vectors\":{\"size\":1536,\"distance\":\"Cosine\"},\"on_disk_payload\":true}"
'
```

## Проверка количества points

```bash
docker run --rm \
  --network n8n_default \
  --env-file /opt/apps/qdrant/.env \
  --entrypoint sh \
  curlimages/curl:8.8.0 \
  -c 'curl -s -H "api-key: $QDRANT_API_KEY" http://qdrant:6333/collections/npd_knowledge_v1'
```

Ожидаемо после ingestion:

```text
points_count: 140
```

## Очистка и пересоздание коллекции

Использовать осторожно. Команда удаляет коллекцию и создаёт заново:

```bash
docker run --rm \
  --network n8n_default \
  --env-file /opt/apps/qdrant/.env \
  --entrypoint sh \
  curlimages/curl:8.8.0 \
  -c '
curl -s -X DELETE \
  -H "api-key: $QDRANT_API_KEY" \
  http://qdrant:6333/collections/npd_knowledge_v1

curl -s -X PUT \
  -H "api-key: $QDRANT_API_KEY" \
  -H "Content-Type: application/json" \
  http://qdrant:6333/collections/npd_knowledge_v1 \
  -d "{\"vectors\":{\"size\":1536,\"distance\":\"Cosine\"},\"on_disk_payload\":true}"
'
```

## Частая ошибка

Ошибка:

```text
getaddrinfo ENOTFOUND qdrant
```

означает, что n8n не может разрешить hostname `qdrant`.

Причина обычно в том, что контейнер n8n был пересоздан и потерял подключение к Docker-сети, где находится Qdrant.

Решение: снова подключить n8n к сети с Qdrant.
