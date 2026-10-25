# Окружения и версии

Проверено 26.10.2026. Адреса разных стендов не взаимозаменяемы: у них могут
отличаться пользователи, пароли, issuer JWT, новости и состояния анализа.

## Cloud.ru: серверный стенд

Базовый адрес API: `https://api.alfa-future-hack-okoai.ru`.
Точка входа — Caddy из Deployment. Корневой домен
`alfa-future-hack-okoai.ru` зарезервирован под фронтенд; его наличие в CORS
само по себе не означает, что фронтенд уже опубликован.

| Сервис | Базовый путь | Swagger |
|---|---|---|
| Account | `/api/account-service` | [Открыть](https://api.alfa-future-hack-okoai.ru/api/account-service/docs) |
| BI | `/api/business-impact-service` | [Открыть](https://api.alfa-future-hack-okoai.ru/api/business-impact-service/docs) |
| News | `/api/news-collector-service` | [Открыть](https://api.alfa-future-hack-okoai.ru/api/news-collector-service/docs) |
| ML | `/api/ml-service` | [Открыть](https://api.alfa-future-hack-okoai.ru/api/ml-service/docs) |

Issuer Keycloak:
`https://auth.alfa-future-hack-okoai.ru/realms/account-hack`.
Ожидаемый audience Account/BI: `account-api`.

| Компонент | Проверенная версия исходников | Статус |
|---|---|---|
| Account | `7131524` | Развёрнут |
| News | `cd43f69` | Развёрнут |
| ML | `a49ff94`, бизнес-логика `dev:7861963` | Развёрнут |
| BI | `c8c01a3` | Развёрнут; лента/сводка по компании, полный результат по request_id |
| Deployment | `b7bd515` | Применён на сервере |
| Go API-Gateway | `9c49289` | Реализован; не включён в текущий серверный Compose |
| Insight | `541337d` | Реализован; не включён в текущий серверный Compose |

Полные SHA и ссылки — [sources.json](../reference/sources.json). Runtime-образы
выбираются по digest в `Deployment/images/*.env`; имя ветки само по себе не
доказывает, что на сервере работает именно её последняя версия.

ML имеет открытый API без JWT по принятой конфигурации этого стенда. Account и
BI проверяют пользовательский JWT. Go Gateway не следует путать с действующим Caddy.

В ML runtime настроен `LLM_BASE_URL=https://api.alfahack-gateway26.ru/v1`.
Код ML по умолчанию содержит старый адрес Railway, поэтому при переносе
необходимо проверить окружение. Значение ключа LLM хранится только в секретах.

## Отдельный демостенд product

- [Лента](https://alfa-oko-demo.serveousercontent.com/feed).
- [Аналитика](https://alfa-oko-demo.serveousercontent.com/analytics).
- [Исходники product/polish](https://github.com/Alfa-future-hack-okoAI/product/tree/2b0efda81a0fcb35c72d25d0d7f0929cc0aeb889).

Фронтенд обслуживает Nginx. `/api/account-service/*` идёт в Account,
`/api/news/*` и `/api/ai/*` — в ML. BI не участвует в этом маршруте чтения.
Статические файлы работающего фронтенда подтверждают запросы `/api/news/feed`
и `/api/news/summary`. Домен используется для публикации через Serveo;
точный сервер, процесс туннеля и runtime-SHA этого стенда по репозиторию
не подтверждены: SSH-доступ к нему в рамках проверки отсутствовал.

`product/up.sh` собирает образы из исходников на машине запуска. Уже имеющиеся
клоны использует как есть, без автоматического pull. GitHub Actions workflows
в product при проверке не найдены. Это отдельный процесс от Cloud.ru CI/CD.

## Локальные адреса по примерам Compose

| Компонент | Адрес |
|---|---|
| Account | `http://localhost:18400/api/account-service` |
| BI | `http://localhost:18500` |
| News | `http://localhost:28000` |
| ML | `http://localhost:8090` |
| product frontend | `http://localhost:8080` |
| Go Gateway | `http://localhost:8088` |
| Insight | `http://localhost:8082` |

Порты могут быть переопределены `.env`. Адрес `localhost` внутри контейнера
указывает на этот контейнер; для межконтейнерных запросов используйте Docker DNS.
