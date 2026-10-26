# Данные для интерфейса

Есть два разных контракта: product читает ML через Nginx, серверный BI отдаёт
свои сохранённые проекции. HTTP-пути похожи, но структуры JSON различаются.

## Выбор и добавление компании

`GET /api/account-service/companies` возвращает прежний список. Показывайте
`name`, при наличии `inn`; различайте источники по `source=fixture|checko`.
Добавление по ИНН выполняется через `POST /api/account-service/companies`,
тело и ошибки описаны в [Account](services/account.md#компании-из-файлов-и-чекко).

Отключайте запуск аналитики при `ai_preparation_available=false` и объясняйте,
что для компаний Чекко он пока недоступен. Сервер также вернёт `409`.
Для таких компаний показывайте сохранённые реквизиты и финансовые показатели
из подробного GET Account; наличие записи Чекко не означает готовой аналитики BI.

## Серверный интерфейс через BI

| Экран | Запрос относительно `/api/business-impact-service` |
|---|---|
| Лента новостей компании | `GET /v1/feed?company_id=UUID&period=7d` |
| Сводка и важные карточки | `GET /v1/summary?company_id=UUID&period=7d` |
| Полный пакет одного задания | `GET /v1/analyses/REQUEST_UUID` |
| Карточка отдельно | `GET /v1/cards/CARD_ID` |
| Исходная обработанная новость | `GET /v1/news/NEWS_UUID` |

Для компании достаточно UUID и JWT: роль и автор определяются сервером.
`period` принимает `24h` или `1d`…`365d`. `feed` поддерживает page/page_size,
`scope=company|all`, `status=all|high_risk|opportunity|unaccounted|not_analyzed`,
`sort_by=rank|published_at`.

### Где заканчивается одна новость

В ответе BI **один элемент `items` — одна новость вместе с её оценкой**:

```text
ответ BI
├── company_id, role, period, analysis, counts, total, page, page_size
└── items
    ├── [0] первая новость
    │   ├── article: title, url, source, published_at, sentences…
    │   ├── card: полная карточка либо null
    │   ├── rank: 10
    │   ├── rank_raw: 2.214
    │   ├── status
    │   ├── display: исходный элемент ML feed
    │   └── highlight: исходное представление ML карточки либо null
    └── [1] вторая новость с таким же набором полей
```

`card`, `display` и `highlight` внутри первого элемента не являются следующими
новостями. Они частично повторяют данные и делают ответ большим.
Компактная версия BI feed обсуждалась, но пока не реализована.

| Что показывать | Поле BI |
|---|---|
| ID новости | `item.article.news_id` |
| Заголовок / ссылка / источник | `item.article.title`, `.url`, `.source` |
| Превью новости | `item.article.summary` |
| Дата | `item.article.published_at` |
| Число 1–10 в кружке | **`item.rank`** |
| Дробный ранг для сортировки | `item.rank_raw` |
| Метка влияния | `item.status` |
| Картинка и логотип при наличии | `item.display.imageUrl`, `.sourceLogo` |
| Подробности влияния | `item.card`; отдельная загрузка необязательна |
| Показ карточки из ML | `item.highlight`, если получен |

Пример адаптера для простого списка:

```javascript
const news = response.items.map(item => ({
  id: item.article.news_id,
  title: item.article.title,
  summary: item.article.summary,
  url: item.article.url,
  source: item.article.source,
  publishedAt: item.article.published_at,
  imageUrl: item.display?.imageUrl ?? null,
  sourceLogo: item.display?.sourceLogo ?? null,
  status: item.status,
  rank: item.rank,
  cardId: item.card?.card_id ?? null
}));
```

Это код клиента, не новый ответ BI. Даты форматируйте явно в нужном часовом
поясе. При `rank=null` скрывайте кружок; при `card=null` нет карточки для открытия.
Отсутствующий imageUrl требует заглушки. `counts` относятся ко всему набору
до пагинации; `items.length` — только текущая страница. Для summary `total`
означает число видимых карточек, а `counts.news` — число новостей.

### Аналитика компании

В BI summary: `display_company` — представление компании из ML, `counts.high_risk`,
`counts.opportunity`, `counts.news` — счётчики, `items[]` — подробные карточки.
`analysis.processing.finished_at` — время окончания выбранного задания.
`items[].highlight` может быть `null`; UI должен уметь использовать `card`.

Если нужен исходный сохранённый объект ML, прочитайте
`GET /v1/analyses/{request_id}` и поле `ml_summary`.
Поле `ml_summary` содержит `calculatedAt`, `company`, `counters`, `highlights`,
но относится к снимку импорта. ID задания возьмите из `analysis.request_id`
в ответе feed или summary. Если `analysis=null`, сохранённого результата
для выбранного пользователя и роли ещё нет. Отдельной компактной BI-ручки только для `ml_summary` нет.
Счётчики BI и ML нельзя автоматически считать одинаковыми: ML totalEvents
считает свои статьи периода, BI — новости выбранной выдачи компании.

## Фронтенд product: прямой контракт ML

| Страница | Запрос | Форма ответа |
|---|---|---|
| `/feed` | `GET /api/news/feed` | Массив коротких объектов новостей |
| `/analytics` | `GET /api/news/summary` | Объект `company`, `counters`, `highlights`, `calculatedAt`, `profile` |

В запросе: `companyId=U_*`, `userId=UUID`, `role`, `period`. Лента дополнительно
передаёт `status=all`, `sortBy=rank`. Эти имена не совпадают с query BI.

Короткий ML-элемент содержит `id`, `type`, `title`, `url`, `source`, `sourceLogo`,
`imageUrl`, `publishedAt`, `status`, `statusLabel`, `rank`, `rankRaw`, `matchScore`,
`cardId`, `hiddenByProfile`, `importance`, `storyId`, `position`.
В polish добавлен `pinned`; при collapse=true добавляется `sameEvent`.
В ленте нет полной карточки, предложений-источников и trace.

product форматирует только `publishedAt` и даты `sameEvent` в `ru-RU`,
`Europe/Moscow`. Для analytics форматирует `calculatedAt` и даты highlights.
Ранг приходит готовым: UI выбирает CSS-класс для 1–3 / 4–7 / 8–10 и печатает число.
Подпись статуса UI выбирает своим словарём, хотя сервер также отдаёт statusLabel.
`pinned=true` включает метку «важное»; `sameEvent.length` — число дополнительных
публикаций. ID новости передаётся в ML-чат как контекст выбранного события.

Прямое присоединение этого UI к BI требует адаптера данных и маршрутов.
BI не реализует product-ручки pin/hide/feedback и не передаёт их в ML.

Источники: [frontend handoff](https://github.com/Alfa-future-hack-okoAI/API-Gateway/blob/main/docs/frontend.md),
[ML HTTP API](https://github.com/Alfa-future-hack-okoAI/ml/blob/7d966a2fe3ece604da9d4b3645031f196abe9b3e/alfa_ml/api.py),
[Swagger BI](https://api.alfa-future-hack-okoai.ru/api/business-impact-service/docs).
