# Источники и машинные контракты

Срез документации: **26.10.2026, Europe/Moscow**.

## Происхождение

[sources.json](sources.json) фиксирует полные SHA исходников. Для серверных
сервисов они сверены с image revision labels работающих контейнеров и manifest
Deployment. Account/BI/News/ML OpenAPI скачаны с соответствующих публичных
`/openapi.json`. Схема Insight извлечена из openAPIDocument его исходного кода;
это не проверка развёрнутого Insight.

Ни один новый платный LLM-запрос для подготовки документации не выполнялся.
Приложения, БД, runtime env и пользовательские credentials при этом не менялись.

## HTTP

- [Индекс всех методов и путей](http-api.md).
- [Account OpenAPI](account-openapi.json).
- [Business-Impact OpenAPI](business-impact-openapi.json).
- [News-Collector OpenAPI](news-collector-openapi.json).
- [ML OpenAPI текущего сервера](ml-openapi.json).
- [Insight OpenAPI из кода](insight-openapi.json).

Версия OpenAPI info.version и source commit — разные понятия. Совпавшие
info.version двух образов не доказывают совпадение бизнес-логики.

## События

| Схема | Владелец / источник |
|---|---|
| [news.collected.v1](events/news.collected.v1.schema.json) | Producer News; совместимая копия контракта Account |
| [company.ai-preparation.requested.v1](events/company.ai-preparation.requested.v1.schema.json) | Account |
| [company.ai-preparation.finished.v1](events/company.ai-preparation.finished.v1.schema.json) | Согласованный Account/ML finished |
| [ai-preparation-status.v1](events/ai-preparation-status.v1.schema.json) | HTTP status Account |
| [news.article.processed.v1](events/news.article.processed.v1.schema.json) | Нативный ML output; схема модели consumer BI |

Полные [примеры событий](../examples/events/) синтетические либо происходят
из контрактных fixtures. ID, хеш и даты finished-примеров согласованы с
примером requested. Они нужны для понимания формата и локальных тестов;
не публикуйте их в рабочие очереди как новые реальные события.

Schema проверяет структуру, но не все межполевые инварианты. Например,
context_version требует дополнительного пересчёта hash; card source_ids —
разрешения в источники; компания — совпадения с авторизованным снимком.

## Обновление

Обновляйте снимок только вместе с указанием producer revision и проверкой
совместимости consumer. Runtime-секреты, пользовательские CSV и bearer-токены
не являются частью reference. Для текущего runtime используйте живой Swagger,
а для воспроизводимости интеграции — зафиксированный source SHA.
