# Account-Service

Account хранит пользователей, связи с компаниями и роль в каждой связи,
факты компании, платёжные агрегаты и документы. Проверяет доступ и формирует
снимок контекста для ML. Аутентификацию по телефону/паролю выполняет Keycloak.

## API

Внешний и внутренний префикс приложения: `/api/account-service`.
OpenAPI показывает короткие `paths` и этот префикс в `servers`.

| Метод и путь после префикса | Назначение |
|---|---|
| `GET /auth/users` | Поиск демопользователя; не возвращает пароль |
| `POST /auth/login` | Телефон/пароль → access и refresh token |
| `POST /auth/refresh` | Обновление токенов |
| `GET /users/me` | Текущий пользователь |
| `POST /companies` | Добавить компанию по ИНН из Чекко |
| `GET /companies` | Доступные компании из файлов и Чекко; источник, ИНН и доступность аналитики |
| `GET /companies/{company_id}` | Компания, платежи, контрагенты, ML-агрегаты |
| `PATCH /companies/{company_id}` | Дополнительный бизнес-профиль |
| `GET /companies/{company_id}/context` | Снимок контекста для персонализации |
| `GET /companies/{company_id}/payments/summary` | Платёжная сводка |
| `GET /companies/{company_id}/counterparties` | Контрагенты с пагинацией |
| `POST /companies/{company_id}/ai-preparations` | Создать асинхронную подготовку |
| `GET /companies/{company_id}/ai-preparations/{request_id}` | Состояние собственного задания |
| `PUT /companies/{company_id}/documents/{document_type}/{year}` | Загрузка/замена PDF |
| `GET /companies/{company_id}/documents/{document_type}/{year}` | Скачать PDF |
| `GET /companies/{company_id}/documents` | Список документов |

Полные тела, типы, обязательные поля и ограничения — [OpenAPI](../../reference/account-openapi.json).
`/health`, `/ready` и документация — отдельные служебные пути приложения;
в опубликованный прикладной OpenAPI включены не все служебные операции.

## Компании из файлов и Чекко

`GET /companies` сохраняет `items/total/page/page_size`. В элементах добавлены
`source`, `inn`, `ai_preparation_available`. Для исходного набора это
`fixture`, `null`, `true`; для Чекко — `checko`, строка ИНН, `false`.
UUID по-прежнему берётся из `id`, а `CHECKO_*` — внутренний external_id новой
компании. Названия и ИНН не используются для связывания с обезличенными U_*.

Добавление требует Bearer JWT:

```http
POST /api/account-service/companies
Content-Type: application/json
Authorization: Bearer ACCESS_TOKEN
```

```json
{
  "inn": "7816435129",
  "role": "финансовый директор",
  "business_profile": {"sales_model": "B2B", "tax_regime": "OSNO"}
}
```

Успех — `201`, повтор ИНН в своём списке — `409`. Реквизиты и последняя
доступная отчётность загружаются сервером и сохраняются в PostgreSQL.
Подробный GET и `context.company` дополнены `infoCompany`, `financials`,
`checko_fetched_at`. Имена полей старой интеграции сохранены. Без отчётности
`financials=null`; неизвестные показатели не подменяются нулями.

Компания принадлежит списку добавившего пользователя; тот же ИНН у другого
пользователя получает отдельный UUID и бизнес-профиль. Чужая компания — `404`.
Это пользовательское добавление, а не проверка юридических полномочий.
GET читает БД без повторного запроса Чекко. Автообновление снимка не реализовано.

Для Чекко `ml_context=null`, источником служит `checko`. Платежи набора U_*
не подставляются: нулевые суммы означают отсутствие загруженных операций.
Оба маршрута `ai-preparations` возвращают `409` до записи задания/outbox.
Создание и PATCH компании Чекко не публикуют события. Для компаний из файлов
ML-контракт и hash снимков остаются прежними.

Ключ Чекко хранится в runtime env Account (`CHECKO_API_KEY`), базовый URL —
`https://api.checko.ru/v2/`, таймаут запроса — `CHECKO_TIMEOUT_SECONDS` (20 секунд).
Без ключа новая операция даёт `503`, остальные ручки работают. Ошибки ответа,
идентичности компании или доступа провайдера — `502`; сеть/429/5xx — `503`;
неверный ИНН — `422`. Gateway должен учитывать два последовательных внешних
запроса и сохранять эти статусы, JWT и JSON без изменения тела.

## Контекст компании

`context` события содержит `company`, `payments`, `top_counterparties`,
`counterparties_total`, `data_warnings`, `sources`, `ml_context`.
Роль и автор находятся в envelope команды, а не задаются браузером.

Роли Account: собственник бизнеса, генеральный директор, финансовый директор,
главный бухгалтер, руководитель казначейства, другие руководители.
BI принимает соответствующие алиасы OWNER, CEO, CFO, CHIEF_ACCOUNTANT,
HEAD_OF_TREASURY, OTHER_EXECUTIVE. Для пользовательских запросов роль
автоматически берётся из текущей связи пользователя с компанией.

ML-агрегаты считаются по всем платежам, не только топу контрагентов:

- `revenue`, `group_revenue`, `group_size`, `group_revenue_complete`, `is_large`;
- `buyers` и `suppliers`: сумма, число уникальных контрагентов, `top1_share`,
  `known_share`, отраслевые доли `sections`;
- `excluded_from_ml`, `aggregation_version`.

`incoming_count` — число платежей, `buyers.count` — число разных покупателей.
Доли передаются в диапазоне 0…1, суммы Decimal — строками.
Техническая запись `U_428697` и самопереводы исключаются из ML-агрегатов;
выручка до 1000 включительно для ML считается неизвестной. Исходные значения
при этом сохранены. Названия и роли могут быть синтетическими — см. data_warnings.

## События и статусы

Account публикует `company.ai-preparation.requested.v1` через PostgreSQL outbox.
Принимает **только уведомления результата подготовки** `completed/failed`
и обновляет задание. Новостные события Account не потребляет.

HTTP-статусы задания: `pending`, `queued`, `ready`, `failed`.
Idempotency-Key — UUID; повтор собственного действия возвращает прежнее задание.
Новый ключ нужен для нового расчёта. Снимок контекста и его hash неизменяемы
в пределах задания.

## Документы

PDF хранится в MinIO, в PostgreSQL — его метаданные, хеш и object_key.
Типы: `balance_sheet`, `financial_results`, `changes_in_equity`, `cash_flows`,
`targeted_funds`. Default-лимит файла Account — 20 MiB; проверьте также лимит Gateway.
Наличие PDF в Account не означает, что текущий ML автоматически анализирует его:
описанный поток подготовки передаёт структурированный контекст.

Источники: [код и API Account](https://github.com/Alfa-future-hack-okoAI/Account-Service/tree/9e14c05f6bf2a4d82285fa73721d23fbdc5303a2),
[контракты](../contracts.md).
