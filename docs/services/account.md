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
| `GET /companies` | Доступные текущему пользователю компании |
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

Источники: [код и API Account](https://github.com/Alfa-future-hack-okoAI/Account-Service/tree/71315241bbf9b28ffdc15f0ca33ca04de7b3d0c3),
[контракты](../contracts.md), [авторизация](../security.md).
