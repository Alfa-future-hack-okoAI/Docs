# ОкоAI — документация проекта
Команда - 'око эйай'

Контрольные коммиты:

- Account-Service —
- News-Collector-Service —
- Business-Impact-Service —
- ml —
- Insight-Service —
- API-Gateway —
- Deployment —
- Product —
- frontend —
- Docs —

Все репозитории, кроме Docs закрыты, мы пригласили эксперта для проверки остальных репозиториев. 
Проект развернут на сервере, просьба предоставить доступ к ключу на время проверки. 

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

Все используемые CSV собраны в каталоге [data/csv](data/csv). Это снимок
файлов Account-Service от 26 октября 2026 года: исходные наборы организаторов
и сгенерированные демонстрационные справочники. ML-Service использует
`df_Uid_data_hack.csv`, `df_Uid_payment_hack.csv` и `companies_names.csv`. В News-Collector, BI,
Insight и Gateway отдельные входные CSV не обнаружены.

### Исходные данные организаторов

Эти файлы скопированы без изменения содержимого. Кодировка — UTF-8,
разделитель — точка с запятой (`;`). Количество строк указано без заголовка.

| CSV-файл | Строк | Содержимое и использование |
|---|---:|---|
| [df_Fid_Uid_link_hack.csv](data/csv/source/df_Fid_Uid_link_hack.csv) | 12 514 | Связи пользователей `SYNTHETIC_F` с компаниями `SYNTHETIC_U`, город и регион. Account использует при загрузке пользователей и доступных компаний. |
| [df_Uid_data_hack.csv](data/csv/source/df_Uid_data_hack.csv) | 85 525 | Исходные профили компаний: ОКВЭД, выручка, группа, регион, банковские продукты и другие признаки. Используется Account и ML. |
| [df_Uid_payment_hack.csv](data/csv/source/df_Uid_payment_hack.csv) | 1 380 585 | Платёжные связи: получатель, плательщик, сумма и количество платежей. Account и ML используют для агрегатов и связей с контрагентами. |

### Сгенерированные демонстрационные данные

Эти файлы готовит Account-Service на основе исходного набора. Кодировка — UTF-8,
разделитель — запятая (`,`). Имена, телефоны, должности и названия компаний
синтетические; они не являются подтверждёнными реквизитами.

| CSV-файл | Строк | Содержимое и использование |
|---|---:|---|
| [users_profiles.csv](data/csv/generated/users_profiles.csv) | 11 120 | Демонстрационные пользователи: внешний ID, телефон, имя, фамилия, основная роль и основная компания. Используется Account при подготовке и загрузке пользователей. |
| [user_company_roles.csv](data/csv/generated/user_company_roles.csv) | 12 514 | Роль пользователя в каждой компании. Account использует для членства и персонализации; это отдельная роль для каждой связи. |
| [companies_names.csv](data/csv/generated/companies_names.csv) | 204 708 | Синтетические отображаемые названия компаний по внешнему ID. Используется Account и ML. |
| [users_credentials.redacted.csv](data/csv/generated/users_credentials.redacted.csv) | 11 120 | Очищенная копия `users_credentials.csv`: ID пользователя, телефон и ID Keycloak сохранены, значения столбца `password` удалены. |

Оригинальный `users_credentials.csv` содержит действующие пароли и остаётся
в приватных подготовленных данных Account-Service. Очищенная копия нужна
только для просмотра структуры и связей: она не подходит для входа или
загрузки паролей в Keycloak. Для локального запуска используйте штатную
подготовку данных Account, описанную в [локальной разработке](docs/local-development.md).

CSV не содержат компаний, добавленных через Чекко: они загружаются через API
и сохраняются в БД Account. Выгрузки новостей, карточек и истории диалогов
также не являются этими исходными CSV.

Перечень столбцов, размеры файлов и контрольные суммы SHA-256 приведены
в [manifest.json](data/csv/manifest.json). Большие CSV удобнее скачивать
через кнопку **Raw / Download raw file** на странице файла GitHub.

## Контракты и примеры

- [Версии исходников и происхождение](reference/README.md), [sources.json](reference/sources.json).
- [OpenAPI и индекс endpoint](reference/http-api.md).
- [События, схемы и примеры](docs/contracts.md).
- [Словарь ID и оценок](docs/architecture.md#идентификаторы-и-оценки).
