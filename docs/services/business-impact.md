# Business-Impact-Service

BI — хранилище готовых новостей, контекстов, карточек и результатов ML.
Использует FastAPI, PostgreSQL, SQLAlchemy/Alembic и RabbitMQ.
Сам не пересчитывает риск и не запускает LLM.

## Как попадают данные

1. Собственная очередь в vhost news получает `news.article.processed.v1`.
2. Собственная очередь в vhost account получает requested и completed/failed.
3. Worker сопоставляет части по request_id, создаёт durable-задание импорта.
4. При completed читает полные данные по HTTP ML; при failed сохраняет ошибку.
5. Результат, статьи, карточки и `synced` сохраняются транзакционно.

Уведомление completed содержит только result_id и версии. Событие processed
содержит статью и ID карточек; это не полный пакет задания.

## Основные запросы

Внешний префикс — `/api/business-impact-service`, внутренние пути — `/v1/...`.

| Запрос | Ответ |
|---|---|
| `GET /v1/news` | Каталог обработанных новостей, независимо от компании |
| `GET /v1/news/{news_id}` | Полная новость и предложения-источники |
| `GET /v1/feed?company_id=UUID` | Новости компании, включая элементы без карточки |
| `GET /v1/summary?company_id=UUID` | Компания, счётчики, до 10 подробных карточек |
| `GET /v1/cards/{card_id}` | Полная карточка влияния |
| `GET /v1/analyses/{request_id}` | Полный результат конкретного задания |
| `GET /v1/analyses/{request_id}/sync` | Состояние HTTP-импорта |
| `POST /v1/analyses/{request_id}/sync/retry` | Возобновить неудавшийся импорт без новой генерации |
| `GET /v1/companies/{company_id}/profile` | Снимок Account и текущие сохранённые ML-признаки |
| `GET /v1/companies/{company_id}/contexts/{context_version}` | Точный снимок для объяснения старой карточки |

Для article/card `revision=N` выбирает точную ревизию BI; без него — последнюю.
Полные схемы — [OpenAPI](../../reference/business-impact-openapi.json).
Передача `company_id=U_*` вместо UUID даст ошибку валидации.

## Последний анализ и история

Для feed, summary и profile выбор ограничен компанией, ролью и автором.
Для Bearer автор — JWT sub, роль проверяется через Account. Запрос другого пользователя не получает
чужой персональный результат как запасной.
Сортировка: started_at, finished_at, request_id по убыванию. Поэтому старое
долго считавшееся задание не заменяет новое по одному лишь времени доставки.
Выбирается последний сохранённый исход, включая partial/failed; это не
фильтр только успешных результатов.

Полный пакет одного задания возвращает `GET /v1/analyses/{request_id}`.
Поле `analysis` в feed/summary содержит только метаданные выбранного задания,
включая `request_id`. Лента дополнительно объединяет сохранённые связи разных
запусков той же компании/роли/автора и версии контекста. Пустой последующий
запуск не удаляет эти связи.

## Структуры ответа

`CompanyView` у feed/summary: `company_id`, `role`, `period`, `analysis`,
`scope`, `company`, `display_company`, `items`, `total`, `page`, `page_size`,
`counts`, `outcomes`.

`items[]`: `article`, `card`, `status`, `rank`, `rank_raw`, `hidden_by_profile`,
`display`, `highlight`, `analysis_id`, `context_version`.
Это подробная структура; компактная лента пока не выпущена.

Нативный импорт имеет `source="ml-http-import"`, `source_event`, `imported_at`,
контекст, `articles`, `cards`, `feed_items`, исходы и `ml_result`, `ml_summary`,
`ml_company`. Его статус: ready/partial/failed.
`processing_counts_scope=visible_result_cards` означает, что счётчики обработки
описывают карточки результата, а не точное число внутренних LLM-попыток.

## Надёжность и доступ

Повторы входных событий дедуплицируются. Quorum-очереди имеют retry/dead.
HTTP-загрузка использует lease и до 12 попыток по текущей конфигурации; после
исчерпания — blocked. Last_error хранит класс ошибки без тела модели и секретов.
BI сверяет компанию, роль, контекст, result_id и версии. Несовпадение не следует
обходить ослаблением авторизации.

При JWT проверяются подпись/claims и текущий доступ через Account. Внутренний
Service-Key предназначен для доверенного server-to-server канала; Caddy его
удаляет на внешнем входе.

Источники: [BI](https://github.com/Alfa-future-hack-okoAI/Business-Impact-Service/tree/c8c01a3bb069eb2f89691822c1116ccfd4b92f34),
[структура интерфейса](../frontend.md), [диагностика](../operations.md).
