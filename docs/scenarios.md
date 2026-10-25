# Пользовательский сценарий

Ниже — действующий Cloud.ru-стенд. База:
`https://api.alfa-future-hack-okoai.ru`.
Значения `COMPANY_UUID`, `REQUEST_UUID`, `CARD_ID`, `ACCESS_TOKEN` — подстановки
из ваших ответов. Пароли не включены в документацию.

## 1. Вход

Демопользователя можно найти через
`GET /api/account-service/auth/users?search=F_...`.
Номер и пароль предоставляет владелец окружения из его устойчивого файла
`users_credentials.csv`. Каталог пользователей пароль не возвращает.

```http
POST /api/account-service/auth/login
Content-Type: application/json

{"phone_number":"PHONE_FROM_YOUR_ENVIRONMENT","password":"PASSWORD_FROM_YOUR_ENVIRONMENT"}
```

Сохраните `access_token`, `refresh_token` и `expires_in`. Для остальных запросов:

```http
Authorization: Bearer ACCESS_TOKEN
```

В Swagger введите в Bearer-поле только сам access token. Внешний BI ServiceKey
оставьте пустым: публичный Caddy удаляет этот внутренний заголовок.

## 2. Выбрать компанию

```http
GET /api/account-service/companies?page=1&page_size=20
```

Возьмите `items[].id` — UUID доступной компании. Внешний `U_*` находится в
`external_id` и не заменяет UUID в BI.

```http
GET /api/account-service/companies/COMPANY_UUID/context
```

Этот GET читает контекст и не запускает ML. Дополнительный бизнес-профиль
изменяется через `PATCH /companies/{company_id}`; схема — в Swagger Account.

## 3. Запустить подготовку

```http
POST /api/account-service/companies/COMPANY_UUID/ai-preparations
Authorization: Bearer ACCESS_TOKEN
Idempotency-Key: NEW_UUID
Content-Type: application/json

{}
```

Роль, автор и контекст вычисляются сервером. Тело запроса не принимает
`max_news`, `role` или произвольный контекст. Новое действие получает новый
UUID-ключ. Повтор того же HTTP-действия использует прежний ключ и тело.

- `202` — новое задание; сохраните `request_id` и `Location`.
- `200` — повтор того же ключа, возвращено прежнее задание.
- `409` — ключ уже связан с другой компанией в области пользователя.

## 4. Проверить завершение и импорт

```http
GET /api/account-service/companies/COMPANY_UUID/ai-preparations/REQUEST_UUID
```

`pending` — создано, `queued` — публикация подтверждена, `ready` — ML сообщил
об успехе, `failed` — есть ошибка обработки. `result.result_id` — ID результата ML.

```http
GET /api/business-impact-service/v1/analyses/REQUEST_UUID/sync
```

Дождитесь `state=synced`. `ready` в Account недостаточно. Состояния BI и причины
404/retry/blocked описаны в [диагностике](operations.md).

## 5. Показать интерфейс без сохранения request_id

```http
GET /api/business-impact-service/v1/feed?company_id=COMPANY_UUID&period=7d
```

Левая лента: `items[].article.title`, `.url`, `.source`, `.published_at`;
статус — `items[].status`, число в кружке — **`items[].rank`**.

```http
GET /api/business-impact-service/v1/summary?company_id=COMPANY_UUID&period=7d
```

Сводка: `display_company`, `counts`, до десяти подробных элементов `items`.

```http
GET /api/business-impact-service/v1/analyses/REQUEST_UUID
```

Этот запрос нужен, если требуется полный пакет одного задания: контекст,
статьи, карточки, исходы и исходные ответы ML. Возьмите `REQUEST_UUID` из
`analysis.request_id` в ответе feed или summary; хранить его на фронтенде заранее
не требуется. При `analysis=null` сохранённого анализа ещё нет.

Feed и summary выбирают последнее сохранённое задание текущего пользователя и
роли. Пока новый результат не импортирован, доступна предыдущая выдача. После
импорта `analysis.request_id` указывает на новый запуск. Проверяйте
`analysis.status`: возможны `ready`, `partial` и `failed`.

Расписание «раз в 9 часов» пока не реализовано. GET не запускает расчёт.

## 6. Открыть карточку и источники

Из ленты возьмите `items[].card.card_id`; `card=null` означает, что оценки нет.

```http
GET /api/business-impact-service/v1/cards/CARD_ID
```

Карточка содержит блоки вывода, действия, позицию банка, `sources` и `trace`.
Точную новость можно прочитать по `news_id` и `news_revision`, снимок компании —
по `company_id` и `context_version`. Не собирайте `card_id` из строк вручную.

## 7. Чат

Контракт Insight уже реализован в репозитории, но его серверный маршрут пока
не подключён. Локальный сценарий создания диалога и отправки сообщения —
на [странице Insight](services/insight.md). Чат product идёт непосредственно в ML
и не является подтверждением развёрнутого Insight.
