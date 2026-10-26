# ОкоAI — документация проекта
Команда - 'око эйай'

Контрольные коммиты:

- Account-Service (`main`) — [fd870fbc5d4801f0a264da47121a8c9589f8a8d3](https://github.com/Alfa-future-hack-okoAI/Account-Service/commit/fd870fbc5d4801f0a264da47121a8c9589f8a8d3)
- News-Collector-Service (`main`) — [cd43f6904e7aece70a777c10d7a835a002e49e12](https://github.com/Alfa-future-hack-okoAI/News-Collector-Service/commit/cd43f6904e7aece70a777c10d7a835a002e49e12)
- Business-Impact-Service (`main`) — [6936cd7a91902b811c8494d2bc4007c05eb9589e](https://github.com/Alfa-future-hack-okoAI/Business-Impact-Service/commit/6936cd7a91902b811c8494d2bc4007c05eb9589e)
- ml (`main`) — [d4f3783529a76dd553cbfdb24a34703d0e4797a5](https://github.com/Alfa-future-hack-okoAI/ml/commit/d4f3783529a76dd553cbfdb24a34703d0e4797a5)
- Insight-Service (`main`) — [77a4ac49efbc12478c7304e03d5fa72f7d13301d](https://github.com/Alfa-future-hack-okoAI/Insight-Service/commit/77a4ac49efbc12478c7304e03d5fa72f7d13301d)
- API-Gateway (`main`) — [bb692a3991b51184954fb8bc5fb543becdc6ed70](https://github.com/Alfa-future-hack-okoAI/API-Gateway/commit/bb692a3991b51184954fb8bc5fb543becdc6ed70)
- Deployment (`main`) — [207735ab83aa5b464494a213d76d7ec5a352d20a](https://github.com/Alfa-future-hack-okoAI/Deployment/commit/207735ab83aa5b464494a213d76d7ec5a352d20a)
- Product (`polish`) — [9609d42e7295060d6e8432bd83416a563fd354bd](https://github.com/Alfa-future-hack-okoAI/product/commit/9609d42e7295060d6e8432bd83416a563fd354bd)
- frontend (`dev`) — [839a40b3db31316ac682596d6fcfc592df3a9add](https://github.com/Alfa-future-hack-okoAI/frontend/commit/839a40b3db31316ac682596d6fcfc592df3a9add)
- Docs (`main`, до обновления списка) — [ee708c147eeea78d215c7e1f423b62586b235d2f](https://github.com/Alfa-future-hack-okoAI/Docs/commit/ee708c147eeea78d215c7e1f423b62586b235d2f)

Все репозитории, кроме Docs закрыты, мы пригласили эксперта для проверки остальных репозиториев. 
Проект развернут на сервере, просьба предоставить доступ к ключу на время проверки. 
Демосборка для демонстрации работы находиться в Product, локальный запуск.


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
| [frontend](https://github.com/Alfa-future-hack-okoAI/frontend), [product](https://github.com/Alfa-future-hack-okoAI/product) | Интерфейс и отдельная демосборка для демонстрации работы | [Два сценария фронтенда](docs/frontend.md) |
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
