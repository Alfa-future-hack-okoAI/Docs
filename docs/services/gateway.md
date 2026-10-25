# API Gateway

## Что работает на Cloud.ru

Сейчас внешний вход — **Caddy в Deployment**. Он завершает TLS и проксирует:

| Публичный префикс | Внутренний адрес | Изменение пути |
|---|---|---|
| `/api/account-service` | `account-api:8000` | Сохранить весь префикс |
| `/api/business-impact-service` | `bi-api:8000` | Снять только внешний префикс |
| `/api/news-collector-service` | `news-api:8000` | Снять внешний префикс |
| `/api/ml-service` | `ml:8090` | Снять внешний префикс |

Для Swagger BI/News/ML Uvicorn настроен с соответствующим root_path.
Account уже содержит свой префикс в маршрутах; добавление root_path поверх
него приведёт к дублированию пути.

Caddy передаёт Bearer и Idempotency-Key. Внешние `X-Service-Key`, `X-User-ID`,
`X-Role` удаляются. Пользовательские права проверяют Account и BI.
Внешний ML в текущем стенде опубликован без JWT по принятому решению.
Админка Keycloak не опубликована: auth-домен проксирует только realm/resources.

CORS текущего Caddy: `https://alfa-future-hack-okoai.ru`,
`http://localhost:5173`, `http://localhost:3000`. Лимит тела — 22MB по конфигурации;
Account дополнительно ограничивает сам PDF 20 MiB.

## Отдельный Go API-Gateway

Реализован в своём репозитории, но не включён в текущий Cloud.ru Compose.
Он маршрутизирует Account, BI и Insight:

| Префикс | Upstream | Rewrite |
|---|---|---|
| `/api/account-service` | `ACCOUNT_URL` | Без изменения |
| `/api/business-impact-service` | `BUSINESS_IMPACT_URL` | Удалить этот префикс |
| `/api/insight-service` | `INSIGHT_URL` | Удалить этот префикс |

Go Gateway не выдаёт и не проверяет JWT. Он сохраняет Authorization и удаляет
подменяемые служебные/forwarded-заголовки, не подставляет внутренний ключ BI.
Query, методы, JSON, PDF и потоковые ответы передаются downstream. Собственного
retry изменяющих запросов нет.

Служебные endpoints: `/healthz`, `/readyz`, `/docs`. Readiness обращается к
Account ready, BI ready, Insight readyz и не заменяет сквозную проверку авторизации.
Для BI/Insight Gateway отдаёт Swagger/ReDoc и переписывает OpenAPI servers;
Account schema проходит без изменения путей.

Default-параметры: listen `127.0.0.1:8088`, body 25 MiB, connect 5s,
Account/BI 60s, Insight 210s. CORS — точный список origins, wildcard запрещён.
Host-network Compose рассчитан на Linux; на Mac удобнее Go-процесс или явно
настроенные Docker-адреса.

Перенос этого Gateway в production требует отдельной задачи Deployment:
добавить Insight и его БД/секреты/health, выбрать образы и согласовать, кто
обслуживает News/ML и TLS. Текущие правила Caddy не следует удалять только
потому, что существует репозиторий Go Gateway.

Источники: [Caddy Deployment](https://github.com/Alfa-future-hack-okoAI/Deployment/blob/e32b30f1525eb686e028837b49bca8172c04933f/gateway/Caddyfile),
[Go Gateway](https://github.com/Alfa-future-hack-okoAI/API-Gateway/tree/9c49289f45233f0aa47cf16a20a720dd9b72a72c).
