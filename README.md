# ОкоAI — документация проекта
Команда «око эйай».

ОкоAI связывает новости с бизнес-контекстом компании и показывает риски,
возможности, источники и рекомендуемые действия.

Исходники сервисов доступны в закрытых репозиториях по приглашению.

## С чего начать

| Что нужно сделать | Что прочитать |
|---|---|
| Понять состав проекта и поток данных | [Архитектура](docs/architecture.md) |
| Найти адреса работающих сервисов | [Окружения](docs/environments.md) |
| Войти, выбрать компанию и проверить анализ | [Пользовательский сценарий](docs/scenarios.md) |
| Подключить фронтенд и разобраться в JSON | [Данные для интерфейса](docs/frontend.md) |
| Найти endpoint и схему ответа | [Справочник HTTP API](docs/api.md) |
| Реализовать отправку/приём RabbitMQ | [Межсервисные контракты](docs/contracts.md) |
| Разобраться с базами и версиями данных | [Модель хранения](docs/data-model.md) |
| Настроить выпуск и обновление сервера | [Деплой и CI/CD](docs/deployment.md) |
| Запустить сервисы локально | [Локальная разработка](docs/local-development.md) |
| Скачать исходные CSV и демонстрационные справочники | [Файлы данных](#csv-файлы-проекта) |
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
отдельную схему `product`.

## CSV-файлы проекта

Все используемые CSV собраны в каталоге [data/csv](data/csv).

### Исходные данные организаторов

| CSV-файл | Строк | Содержимое и использование |
|---|---:|---|
| [df_Fid_Uid_link_hack.csv](data/csv/source/df_Fid_Uid_link_hack.csv) | 12 514 | Связи пользователей `SYNTHETIC_F` с компаниями `SYNTHETIC_U`, город и регион. Account использует при загрузке пользователей и доступных компаний. |
| [df_Uid_data_hack.csv](data/csv/source/df_Uid_data_hack.csv) | 85 525 | Исходные профили компаний: ОКВЭД, выручка, группа, регион, банковские продукты и другие признаки. Используется Account и ML. |
| [df_Uid_payment_hack.csv](data/csv/source/df_Uid_payment_hack.csv) | 1 380 585 | Платёжные связи: получатель, плательщик, сумма и количество платежей. Account и ML используют для агрегатов и связей с контрагентами. |

### Сгенерированные демонстрационные данные

| CSV-файл | Строк | Содержимое и использование |
|---|---:|---|
| [users_profiles.csv](data/csv/generated/users_profiles.csv) | 11 120 | Демонстрационные пользователи: внешний ID, телефон, имя, фамилия, основная роль и основная компания. Используется Account при подготовке и загрузке пользователей. |
| [user_company_roles.csv](data/csv/generated/user_company_roles.csv) | 12 514 | Роль пользователя в каждой компании. Account использует для членства и персонализации; это отдельная роль для каждой связи. |
| [companies_names.csv](data/csv/generated/companies_names.csv) | 204 708 | Синтетические отображаемые названия компаний по внешнему ID. Используется Account и ML. |
| [users_credentials.redacted.csv](data/csv/generated/users_credentials.redacted.csv) | 11 120 | Очищенная копия `users_credentials.csv`: ID пользователя, телефон и ID Keycloak сохранены, значения столбца `password` удалены. |

В `users_credentials.redacted.csv` пароли удалены. Этот файл показывает
структуру данных и не подходит для входа в систему.

Компании, добавленные через Чекко, хранятся в БД Account и не входят в эти CSV.
Большие CSV можно скачать через **Raw / Download raw file** на GitHub.

## API и взаимодействие

- [Справочник HTTP API](docs/api.md).
- [События, схемы и примеры](docs/contracts.md).
- [Словарь ID и оценок](docs/architecture.md#идентификаторы-и-оценки).
