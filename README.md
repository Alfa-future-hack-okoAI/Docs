# ОкоAI — документация проекта

ОкоAI связывает новости с бизнес-контекстом компании: собирает публикации,
обрабатывает их в ML, сохраняет выводы и показывает пользователю риски,
возможности, источники и рекомендуемые действия.

Документация сверена **26 октября 2026 года, Europe/Moscow** с кодом сервисов,
публичными OpenAPI и версиями контейнеров Cloud.ru. Она описывает текущую
реализацию; незавершённые интеграции отмечены отдельно.

## С чего начать

| Что нужно сделать | Что прочитать |
|---|---|
| Понять состав проекта и поток данных | [Архитектура](docs/architecture.md) |
| Найти работающие адреса и нужную версию | [Окружения и версии](docs/environments.md) |
| Войти, выбрать компанию и проверить анализ | [Пользовательский сценарий](docs/scenarios.md) |
| Подключить фронтенд и разобраться в JSON | [Данные для интерфейса](docs/frontend.md) |
| Найти endpoint и схему ответа | [Справочник HTTP API](reference/http-api.md) |
| Реализовать отправку/приём RabbitMQ | [Межсервисные контракты](docs/contracts.md) |
| Разобраться с базами и версиями данных | [Модель хранения](docs/data-model.md) |
| Подключить JWT и проверку доступа | [Авторизация](docs/security.md) |
| Настроить выпуск и обновление сервера | [Деплой и CI/CD](docs/deployment.md) |
| Запустить сервисы локально | [Локальная разработка](docs/local-development.md) |
| Разобрать ready, retry, blocked, 401 или 404 | [Эксплуатация и диагностика](docs/operations.md) |
| Проверить ограничения и будущие изменения | [Ограничения](docs/known-limitations.md) |

## Сервисы и репозитории

| Компонент | Ответственность | Документация |
|---|---|---|
| [Account-Service](https://github.com/Alfa-future-hack-okoAI/Account-Service) | Пользователи, доступ к компаниям, бизнес-контекст, документы, запуск подготовки | [Account](docs/services/account.md) |
| [News-Collector-Service](https://github.com/Alfa-future-hack-okoAI/News-Collector-Service) | Сбор ЦБ/Интерфакса, хранение исходных новостей, публикация через outbox | [News-Collector](docs/services/news-collector.md) |
| [ml](https://github.com/Alfa-future-hack-okoAI/ml) | Разбор новостей, сопоставление с компанией, карточки, ранги и ML-чат | [ML](docs/services/ml.md) |
| [Business-Impact-Service](https://github.com/Alfa-future-hack-okoAI/Business-Impact-Service) | Импорт и хранение готовых данных ML, API аналитики | [BI](docs/services/business-impact.md) |
| [Insight-Service](https://github.com/Alfa-future-hack-okoAI/Insight-Service) | Авторизованный адаптер чата/перерасчёта, история диалогов | [Insight](docs/services/insight.md) |
| [API-Gateway](https://github.com/Alfa-future-hack-okoAI/API-Gateway) | Go reverse proxy для Account, BI и Insight | [Gateway](docs/services/gateway.md) |
| [frontend](https://github.com/Alfa-future-hack-okoAI/frontend), [product](https://github.com/Alfa-future-hack-okoAI/product) | Интерфейс и отдельная интеграционная демосборка | [Два сценария фронтенда](docs/frontend.md) |
| [Deployment](https://github.com/Alfa-future-hack-okoAI/Deployment) | Compose, Caddy, версии образов, выкладка Cloud.ru | [CI/CD](docs/deployment.md) |

## Работающий серверный стенд

- [Swagger Account](https://api.alfa-future-hack-okoai.ru/api/account-service/docs).
- [Swagger Business-Impact](https://api.alfa-future-hack-okoai.ru/api/business-impact-service/docs).
- [Swagger News-Collector](https://api.alfa-future-hack-okoai.ru/api/news-collector-service/docs).
- [Swagger ML](https://api.alfa-future-hack-okoai.ru/api/ml-service/docs).

На Cloud.ru сейчас работают эти четыре сервиса за **Caddy из Deployment**.
Репозитории Go API-Gateway и Insight содержат реализацию, но пока не включены
в этот серверный Compose. Демостенд `alfa-oko-demo.serveousercontent.com` использует
отдельную схему `product`, в которой фронтенд читает ML напрямую через Nginx.

В серверном ML развёрнут `main` с бизнес-логикой `dev`; `polish` не развёрнут.
Переход на него требует согласования идентификаторов с BI.

## Контракты и примеры

- [Версии исходников и происхождение](reference/README.md), [sources.json](reference/sources.json).
- [OpenAPI и индекс endpoint](reference/http-api.md).
- [События, схемы и примеры](docs/contracts.md).
- [Словарь ID и оценок](docs/architecture.md#идентификаторы-и-оценки).

Примеры событий взяты из контрактных фикстур или явно синтетические. Пароли,
JWT, ключи LLM, Service-Key, `.env` и выгрузки пользователей в Docs не хранятся.

## Как поддерживать документацию

При изменении контракта обновляйте страницу сервиса, межсервисный сценарий,
снимок схемы и пример. Форматом события владеет producer. Сначала проверяйте
совместимость consumer, затем выпускайте новую версию. Снимки в `reference/`
зафиксированы на дату проверки и не заменяют живой Swagger.
