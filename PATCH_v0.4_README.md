# Patch v0.4 для базы знаний НПД

## Принцип patch v0.4

Patch v0.4 не добавляет точечные ответы под отдельные тесты. Он добавляет 4 тематических документа, которые шире покрывают реальные пользовательские зоны:

1. Аренда жилой квартиры и коммерческой недвижимости.
2. Хэндмейд, покупные материалы, заготовки и отличие собственной продукции от перепродажи.
3. Грузоперевозки, такси, транспортные услуги и ограничения вне НПД.
4. Оплата от ООО/ИП, личная карта, расчетный счет и роль чека.

## Файлы

- knowledge_base/raw_documents/npd_property_rent_v1.md
- knowledge_base/raw_documents/npd_handmade_own_products_v1.md
- knowledge_base/raw_documents/npd_transport_taxi_services_v1.md
- knowledge_base/raw_documents/npd_payment_methods_personal_card_v1.md
- knowledge_base/metadata/documents_registry_additions_v0.4.csv

## Как применить

1. Скопировать markdown-файлы в `knowledge_base/raw_documents/`.
2. Добавить 4 строки из `documents_registry_additions_v0.4.csv` в основной `knowledge_base/metadata/documents_registry.csv`.
3. Закоммитить raw-документы и обновленный основной registry.
4. Запустить ingestion workflow.
5. Проверить Qdrant по новым doc_id.
6. Повторно прогнать Evaluation Runner на реальных вопросах.

## Что НЕ делать

Не добавлять отдельный patch-registry как постоянный источник для ingestion, если workflow читает только основной `documents_registry.csv`.

Не добавлять служебные expectation-файлы в тесты. Patch v0.4 — это именно расширение базы знаний, а не подгонка тестов.
