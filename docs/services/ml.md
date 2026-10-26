# ML-Service

ML разбирает новости, извлекает факты и источники, сопоставляет событие с компанией,
строит карточки и ранжирует их под профиль. HTTP API и два RabbitMQ consumer
работают в одном процессе. Состояние — SQLite; кеш моделей хранится отдельно.

## Версия серверного стенда

Развёрнут `main:a49ff94`; бизнес-логика совпадает с `dev:7861963`.
`polish:7d966a2` пока не развёрнут. Принятое решение — оставить dev-логику.

В runtime установлен `LLM_BASE_URL=https://api.alfahack-gateway26.ru/v1`.
Этот адрес переопределяет прежний default в коде. Ключ и параметры моделей
задаются в приватном `ml.env`; проверка новых токенов модели может расходовать бюджет.

## Два входных потока

| Вход | Результат |
|---|---|
| `news.collected.v1` | Общий разбор статьи; событие `news.article.processed.v1` |
| `company.ai-preparation.requested.v1` | Карточки под компанию/роль, сохранённый result; completed/failed |

На сервере `AUTO_CARDS=0`: новость разбирается сразу, а карточки создаются по
запросу компании. Команда Account уже содержит бизнес-контекст, включая агрегаты.
CSV остаются источником демонстрационных/резервных данных ML, а не заменой доступа
и связей пользователя из Account.

Последовательность pipeline: разбор и классификация → таблицы и шаблоны →
сопоставление с компанией → правила каналов влияния → текст карточки → проверка
утверждений/источников → пользовательское ранжирование.
Источники в `sentences` и `card.sources` позволяют объяснить отдельные выводы.
`generated_by` различает результат модели и правил. В replay/без доступной модели
часть полей может быть пустой; заполненность всех полей не гарантируется.

## HTTP для интеграции BI

| Endpoint ML | Назначение |
|---|---|
| `GET /v1/results/{result_id}?userId=UUID` | Результат подготовки, highlights и полные cards |
| `GET /v1/feed?company_id=UUID&user_id=UUID&role=ROLE&period=30d` | Новости и связи с компанией, ранги |
| `GET /v1/summary` | Company/counters/highlights |
| `GET /v1/companies/{company_id}/context` | Контекст/признаки и происхождение |
| `GET /v1/cards/{card_id}` | Полная карточка ML |
| `GET /v1/news/{news_id}` | Полная обработанная статья |

У результата query `userId`, у v1 feed/summary — `user_id`.
Для HTTP `/v1/prepare` есть синхронный аналог команды Account; штатный браузерный
сценарий запускает подготовку через Account/RabbitMQ.

Для product предусмотрены camelCase-алиасы `/api/news/feed`, `/api/news/summary`,
`/api/ai/chat` и POST `/api/news/summary/regenerate`. Полный [справочник API](../api.md).
Их prefix за Cloud.ru Caddy начинается с `/api/ml-service`.

## Ранг

Исходный ранг включает score карточки, веса/фокус профиля, уровень и закрепление.
Показ в UI: `rank = clamp(round(rankRaw * RANK_SCALE), 1, 10)`, default scale=5.
Это абсолютная шкала, а не место новости. `position` — место.
Если нет подходящей видимой карточки, `rank=null`.

## Что отличается в polish

- результаты и карточки подготовки переходят с UUID компании на `U_*`;
  finished-событие по-прежнему содержит UUID;
- роль получает отдельный `roleView` с выводом «для вас» и действиями;
- добавлены hide/unhide/pin/unpin для ленты, `pinned`, новый alias feedback;
- сводка оставляет одну карточку на сюжет;
- уточнены тексты о прошедших прогнозах и рекомендации руководителю;
- повтор одинакового чата ждёт уже запущенный расчёт.

Текущий HTTP-клиент BI проверяет UUID и отклонит результат polish с `U_*`.
Для перехода нужен адаптер идентичности BI, а также согласование
истории card_id и контекста. Одного изменения Docker-тега
для перехода недостаточно.

Источники: [deployed ML](https://github.com/Alfa-future-hack-okoAI/ml/tree/a49ff948dd64ce3298a94b6a0f65f5a17e595750),
[polish](https://github.com/Alfa-future-hack-okoAI/ml/tree/7d966a2fe3ece604da9d4b3645031f196abe9b3e).

Подробные алгоритмы и расширение ML: [pipeline и правила](https://github.com/Alfa-future-hack-okoAI/ml/blob/a49ff948dd64ce3298a94b6a0f65f5a17e595750/docs/PIPELINE.md),
[интеграция](https://github.com/Alfa-future-hack-okoAI/ml/blob/a49ff948dd64ce3298a94b6a0f65f5a17e595750/docs/INTEGRATION.md),
[оценка качества](https://github.com/Alfa-future-hack-okoAI/ml/blob/a49ff948dd64ce3298a94b6a0f65f5a17e595750/docs/QUALITY.md).
