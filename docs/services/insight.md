# Insight-Service

**Статус:** Go-сервис реализован в репозитории, но не подключён к текущему
Cloud.ru Deployment и публичному Caddy. Ниже — действующий контракт кода,
а не обещание доступности `/api/insight-service` на серверном домене.

Insight предоставляет авторизованное чтение BI, адаптер ML-чата/перерасчёта и
устойчивую историю разговоров в собственной PostgreSQL.

## Авторизация и границы

Принимается один Bearer-заголовок. Сам Insight не проверяет подпись JWT через
JWKS и не обращается напрямую к Account/Keycloak. Доступ и verified identity
он получает через защищённый BI с исходным Bearer. BI проверяет JWT и текущую
связь пользователя с компанией.

Владелец разговора: проверенный subject JWT + UUID компании + текущая роль BI.
Чужой/неизвестный conversation возвращает 404. Читать историю без повторной
проверки доступа нельзя.

## API

Go Gateway снимает внешний префикс `/api/insight-service`.

| Внутренний endpoint | Назначение |
|---|---|
| `GET /v1/feed?company_id=UUID` | Сохранить полный ответ BI, без удаления полей |
| `GET /v1/cards/{card_id}?revision=N` | Карточка BI |
| `GET /v1/companies/{company_uuid}/contexts/{context_version}` | Точный снимок BI |
| `POST /v1/conversations` | Создать разговор |
| `GET /v1/conversations?company_id=UUID` | Список разговоров владельца |
| `GET /v1/conversations/{id}?company_id=UUID` | Метаданные разговора |
| `GET /v1/conversations/{id}/history?company_id=UUID` | Страницы истории |
| `POST /v1/chat` | Текущий вопрос, необязательная карточка/разговор → ML |
| `POST /v1/cards/{card_id}/reanalyze` | Перестроить карточку через ML |

Все POST требуют JSON и UUID `Idempotency-Key`. Повтор с тем же владельцем,
точным путём и телом воспроизводит сохранённый ответ. Другие параметры с тем
же ключом — 409. Ключи сохраняются в PostgreSQL и не истекают автоматически.

Пример локального запроса после настройки зависимостей:

```http
POST /api/insight-service/v1/chat
Authorization: Bearer ACCESS_TOKEN
Idempotency-Key: NEW_UUID
Content-Type: application/json

{"company_id":"COMPANY_UUID","message":"Почему эта новость важна?","card_id":"CARD_ID","revision":1}
```

При переданном `conversation_id` реплика добавляется в собственный разговор;
без него создаётся новый. Ответ дополняется `conversation_id` и `provenance`.
Обязательные поля и пагинация — [OpenAPI из кода](../../reference/insight-openapi.json).

## Что важно для интеграции

- Сохранённая история **не передаётся в ML**: каждый вызов отправляет текущий
  вопрос и проверенный контекст, а не многошаговую provider-сессию.
- При неопределённом исходе сохраняется outcome_unknown; автоматического повтора
  изменяющего запроса в ML нет. Distributed exactly-once не заявляется.
- Проверки версии карточки/контекста выполняются до вызова и не атомарны с ML;
  `exact_snapshot_guaranteed=false`.
- Reanalyze меняет ML, но не синхронизирует новую карточку обратно в BI.
  BI revision остаётся прежней; последующий запрос может получить 409 stale reference.
- Адаптер рассчитан на UUID-ветку ML dev, не на смену company_id в polish.
- Отдельного Insight `/summary` или `/companies/{id}/analysis` в этом снимке нет.

Локально сервис слушает 8082; Compose поднимает собственный PostgreSQL.
`/healthz` проверяет процесс; `/readyz` — PostgreSQL и bounded-пробы BI/ML.

Источник: [Insight 541337d](https://github.com/Alfa-future-hack-okoAI/Insight-Service/tree/541337d23ff0cab3c1f7e84702c28723e093d921).
