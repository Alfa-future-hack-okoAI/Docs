# Справочник HTTP API

Снимки Account, BI, News и ML получены с серверного стенда 26.10.2026. Insight извлечён из исходного кода и пока не развёрнут в Cloud.ru.

Полные схемы описывают тела, обязательные поля и типы. Таблицы ниже — индекс, а не замена схемам.

## Account Service

Внешний префикс: `/api/account-service`. [OpenAPI JSON](account-openapi.json).

| Метод | Путь внутри сервиса | Назначение |
|---|---|---|
| `GET` | `/auth/users` | Выбрать подготовленного пользователя для входа |
| `POST` | `/auth/login` | Войти по номеру телефона и паролю |
| `POST` | `/auth/refresh` | Обновить токены авторизации |
| `GET` | `/users/me` | Получить профиль текущего пользователя |
| `GET` | `/companies` | Компании, доступные текущему пользователю |
| `GET` | `/companies/{company_id}` | Полная карточка компании с платежами и топом контрагентов |
| `PATCH` | `/companies/{company_id}` | Изменить дополнительный бизнес-профиль |
| `GET` | `/companies/{company_id}/context` | Бизнес-контекст для персонализации новостей |
| `GET` | `/companies/{company_id}/payments/summary` | Получить сводку платежей компании |
| `GET` | `/companies/{company_id}/counterparties` | Получить контрагентов компании |
| `PUT` | `/companies/{company_id}/documents/{document_type}/{year}` | Загрузить или заменить PDF за отчётный год |
| `GET` | `/companies/{company_id}/documents/{document_type}/{year}` | Скачать PDF своей компании через API |
| `GET` | `/companies/{company_id}/documents` | Список документов своей компании |
| `POST` | `/companies/{company_id}/ai-preparations` | Запросить подготовку сводки и инсайтов после выбора компании |
| `GET` | `/companies/{company_id}/ai-preparations/{request_id}` | Узнать состояние своего запроса подготовки аналитики |

## Business-Impact-Service

Внешний префикс: `/api/business-impact-service`. [OpenAPI JSON](business-impact-openapi.json).

| Метод | Путь внутри сервиса | Назначение |
|---|---|---|
| `GET` | `/v1/news` | Обработанные новости без привязки к компании |
| `GET` | `/v1/news/{news_id}` | Полная новость, включая sentences |
| `GET` | `/v1/cards/{card_id}` | Полная карточка влияния на компанию |
| `GET` | `/v1/analyses/{request_id}` | Полный итог задания ML |
| `GET` | `/v1/companies/{company_id}/contexts/{context_version}` | Исходный снимок Account-Service |
| `GET` | `/v1/feed` | Новости компании, включая новости без карточки |
| `GET` | `/v1/summary` | Счётчики и до десяти подробных карточек |
| `GET` | `/v1/analyses/{request_id}/sync` | Состояние загрузки результата из ML |
| `POST` | `/v1/analyses/{request_id}/sync/retry` | Повторить неудавшуюся загрузку из ML без повторной генерации |
| `GET` | `/v1/companies/{company_id}/profile` | Компания: исходный контекст и рассчитанные ML признаки с источниками |
| `GET` | `/health` | Проверка процесса |
| `GET` | `/ready` | Проверка готовности |

## News-Collector-Service

Внешний префикс: `/api/news-collector-service`. [OpenAPI JSON](news-collector-openapi.json).

| Метод | Путь внутри сервиса | Назначение |
|---|---|---|
| `GET` | `/news` | Список исходных новостей |

## Альфа Око — ML-сервис

Внешний префикс: `/api/ml-service`. [OpenAPI JSON](ml-openapi.json).

| Метод | Путь внутри сервиса | Назначение |
|---|---|---|
| `POST` | `/v1/feedback` | Оценка карточки пользователем: полезно / не полезно и причина (идёт в /v1/quality) |
| `GET` | `/v1/quality` | Мониторинг качества: проверки карточек, позиция банка, время, расход LLM, офлайн-оценка ленты |
| `GET` | `/health` | Проверка процесса |
| `POST` | `/v1/news` | Загрузить любую новость вручную и сразу получить разбор и карточки |
| `GET` | `/v1/news/{news_id}` | Разобранная новость: шаблоны, таблицы с цитатами, предложения |
| `GET` | `/v1/feed` | Лента новостей компании с меткой: высокий риск / возможность / не учтено / не анализировалась |
| `GET` | `/v1/summary` | Сводка: счётчики и карточки «самое важное» (форма GET /api/news/summary) |
| `GET` | `/v1/cards/{card_id}` | Полная карточка: утверждения, источники до строки, каналы, трассировка |
| `GET` | `/v1/cards/{card_id}/data` | Все данные, по которым сделаны выводы карточки: признаки компании, таблица новости, правила |
| `GET` | `/v1/cards/{card_id}/history` | Прежние версии карточки |
| `POST` | `/v1/cards/build` | Построить карточку для пары событие × компания |
| `POST` | `/v1/cards/{card_id}/rebuild` | Перестроить карточку с уточнением пользователя (новая версия, старая — в истории) |
| `POST` | `/v1/prepare` | То же, что событие company.ai-preparation.requested, синхронно: ответное событие + результат |
| `GET` | `/v1/results/{result_id}` | Результат подготовки по result_id из completed: карточки под профиль пользователя |
| `POST` | `/v1/chat` | Чат: вопрос с цитатами, изменение профиля, данных компании, карточки, шаблонов (форма POST /api/ai/chat) |
| `GET` | `/api/news/feed` | = GET /v1/feed в форме front-api.md: companyId, period, status, sortBy |
| `GET` | `/api/news/summary` | = GET /v1/summary в форме front-api.md |
| `POST` | `/api/news/summary/regenerate` | Достроить карточки по главным новостям периода и вернуть сводку (как GET /api/news/summary) |
| `POST` | `/api/ai/chat` | = POST /v1/chat в форме front-api.md |
| `GET` | `/v1/profiles/presets` | Предустановленные профили |
| `GET` | `/v1/profiles/{profile_id}` | Получить профиль |
| `PATCH` | `/v1/profiles/{profile_id}` | Изменить профиль операциями: weight/mute/unmute/focus/block_order/min_level/pin/hide |
| `GET` | `/v1/templates` | Получить шаблоны |
| `POST` | `/v1/templates` | Добавить или заменить шаблон (YAML). Проверяются схема, выражения и имена признаков |
| `GET` | `/v1/features` | Реестр признаков компании: имя, формула, исходные поля, кто добавил |
| `POST` | `/v1/features` | Добавить признак (YAML). Проверяются формула и имена полей |
| `GET` | `/v1/drafts/{draft_id}` | Черновик шаблона или признака из чата |
| `POST` | `/v1/drafts/{draft_id}/apply` | Подтвердить и применить черновик |
| `GET` | `/v1/companies/demo` | Демонстрационные компании |
| `GET` | `/v1/companies/{company_id}/context` | Таблица признаков компании: значение, формула, исходные поля, происхождение; сырые поля; история правок |
| `POST` | `/v1/companies/{company_id}/overrides` | Уточнить данные компании: set_company_field / add_company_note / revert. Исходные данные не меняются |

## Insight-Service

Внешний префикс: `/api/insight-service`. [OpenAPI JSON](insight-openapi.json).

| Метод | Путь внутри сервиса | Назначение |
|---|---|---|
| `GET` | `/healthz` | Проверка процесса |
| `GET` | `/readyz` | Проверка готовности зависимостей |
| `GET` | `/v1/feed` | Получить ленту компании из BI |
| `GET` | `/v1/cards/{card_id}` | Получить полную карточку BI |
| `GET` | `/v1/companies/{company_uuid}/contexts/{context_version}` | Получить точный снимок контекста из BI |
| `POST` | `/v1/conversations` | Создать собственный разговор |
| `GET` | `/v1/conversations` | Список собственных разговоров |
| `GET` | `/v1/conversations/{conversation_id}` | Метаданные собственного разговора |
| `GET` | `/v1/conversations/{conversation_id}/history` | Страница истории разговора |
| `POST` | `/v1/chat` | Отправить вопрос по компании и необязательной карточке |
| `POST` | `/v1/cards/{card_id}/reanalyze` | Перестроить авторизованную карточку в ML |

## Как читать пути

Добавляйте внешний префикс ровно один раз. Например, Account `/auth/login` превращается в `/api/account-service/auth/login`, а BI `/v1/feed` — в `/api/business-impact-service/v1/feed`.

У Account префикс реально есть в маршрутах приложения, хотя его OpenAPI показывает короткие `paths` вместе с `servers`. Для остальных сервисов префикс снимает Gateway.

Публикация Insight по указанному префиксу описывает контракт Go Gateway; действующий Caddy этого маршрута пока не имеет.
