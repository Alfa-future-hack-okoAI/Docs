# Окружения

Серверный стенд, локальный запуск и демосборка product используют разные
адреса и могут иметь разные данные пользователей и компаний.

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

Account, News, ML и BI развёрнуты на сервере. Go API-Gateway и Insight
реализованы в отдельных репозиториях, но пока не подключены к серверному Compose.
Версии образов задаются в `Deployment/images/*.env`.

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
Домен демостенда опубликован через Serveo. Его конфигурация и данные
независимы от серверного стенда Cloud.ru.

`product/up.sh` собирает образы из исходников на машине запуска. Уже имеющиеся
клоны использует как есть, без автоматического pull. Автоматический выпуск
через GitHub Actions в product не настроен. Это отдельный процесс от Cloud.ru CI/CD.

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
