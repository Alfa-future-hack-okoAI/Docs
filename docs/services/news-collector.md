# News-Collector-Service

News-Collector получает новости ЦБ и Интерфакса, нормализует и хранит их до ML.
Стек: Python/FastAPI, PostgreSQL, SQLAlchemy/Alembic, RabbitMQ.

## Процессы

| Процесс | Обязанность |
|---|---|
| API | Чтение сохранённых исходных новостей |
| collector-cbr | RSS/страницы ЦБ → News PostgreSQL |
| collector-interfax | RSS/страницы Интерфакса → News PostgreSQL |
| outbox-worker | Подтверждённая публикация pending-outbox в RabbitMQ |

Сборщики не подключаются к RabbitMQ; отправитель не загружает сайты.
Новость и outbox сохраняются в одной транзакции.
По умолчанию источники опрашиваются каждые 300 секунд, отдельно друг от друга;
значения изменяются конфигурацией окружения.

## API и данные

```http
GET /api/news-collector-service/news?page=1&page_size=20
```

Ответ: `items`, `total`, `page`, `page_size`, `pages`.
page_size — 1…100. Сейчас этот GET не требует JWT.
Swagger доступен по `/api/news-collector-service/docs`.

News содержит `id`, `source`, `external_id`, `title`, `content`, `url`,
`published_at`, `collected_at`, `rss_description`, `content_status`, `content_error`.
Это исходная публикация; `rank`, карточки и выводы о компании здесь не вычисляются.

`content_status`: full/rss_only/unavailable/unknown. Ошибка загрузки страницы
может привести к сохранению RSS-описания. Наличие новости в БД не гарантирует
наличие полного текста или успешного завершения последующего ML.

## Событие

Exchange `news.events` (topic), routing key `news.collected.v1`, очередь ML
`news.collected`, vhost `news`. JSON envelope содержит event_id/event_type/version/
occurred_at и объект `news`. [Схема и пример](../contracts.md).

Уникальная пара `(source, external_id)` предотвращает повторную загрузку известной
новости. Исправления уже сохранённых публикаций сейчас не переиздаются как новая
редакция. Исходные двусмысленные и неполные тексты не следует считать проверенными
бизнес-фактами.

Outbox может повторно опубликовать событие при сбое между отправкой и записью
published_at, поэтому consumer должен учитывать event_id.

Источники: [код News](https://github.com/Alfa-future-hack-okoAI/News-Collector-Service/tree/cd43f6904e7aece70a777c10d7a835a002e49e12),
[OpenAPI](../../reference/news-collector-openapi.json).
