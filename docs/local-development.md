# Локальная разработка

Локальные Compose сервисов и серверный Deployment решают разные задачи.
Не копируйте production.env в локальный проект. Секреты берутся из собственного
окружения; значения .env.example — только образец конфигурации.

## Требования и каталоги

Docker с Compose v2, Git. Для запуска Python с хоста нужны версии из pyproject
сервиса; Account/News/BI используют Poetry, ML — uv. Для Go Gateway/Insight
нужен Go версии из go.mod. Node и зависимости фронтенда определены его Dockerfile
и package-lock.json.

Удобно держать репозитории рядом:

```text
workspace/
  Account-Service/
  News-Collector-Service/
  Business-Impact-Service/
  ml/
  API-Gateway/
  Insight-Service/
  Deployment/
  Docs/
```

Переходы между каталогами ниже выполняются отдельно из workspace.

## Account

Подготовьте исходные CSV в fixtures/source и локальный .env по примеру сервиса.
Generated-данные и users_credentials.csv сохраняются между перезапусками.

```bash
cd Account-Service
cp .env.example .env
docker compose build api
docker compose up -d
docker compose ps
```

cp нужен только при первом запуске. LOCAL_UID/LOCAL_GID и пути фикстур задаются
по README Account. Отдельная предварительная сборка API избегает конкурирующей
сборки одного account-service image несколькими сервисами Compose.
Дождитесь ready и импорта пользователей Keycloak; первый запуск дольше обычного.

## News и ML

```bash
cd News-Collector-Service
cp .env.example .env
docker compose up -d --build
```

ML должен видеть News RabbitMQ и Account RabbitMQ. Для процесса на хосте
используйте опубликованные loopback-порты; для контейнера — Docker DNS в общей
сети. Отдельные Compose-сети без явного подключения друг друга не видят.

В ML .env настройте DATA_DIR на CSV организаторов, LLM_BASE_URL на рабочий
`https://api.alfahack-gateway26.ru/v1` и приватный LLM_API_KEY, либо выберите
документированный replay-режим. Account-события требуют ACCOUNT_RABBITMQ_ENABLED.
Точные команды запуска и остальные параметры — в README ML выбранной ветки.

Для быстрого отдельного демо можно использовать product/up.sh: он поднимает
News, Account, ML и встроенный фронтенд. Он не добавляет BI или Insight и не
является альтернативой настройке их интеграции.

## BI

В .env задайте случайный SERVICE_API_KEY, ML_BASE_URL, ACCOUNT_BASE_URL,
Keycloak public/internal URLs и два AMQP URL. Внутриконтейнерные адреса должны
быть доступны из контейнера, а не только с Mac.

```bash
cd Business-Impact-Service
cp .env.example .env
docker compose up -d --build
```

Compose применяет миграции, запускает API и ml-results-worker. Поднимите BI до
новой подготовки Account, чтобы его очереди успели получить обе части события.
Сценарий проверки — [scenarios.md](scenarios.md), локальные base URL —
[environments.md](environments.md).

## Insight и Go Gateway

Insight Compose поднимает собственный PostgreSQL и сервис. BI_URL/ML_URL
из его .env.example должны указывать на реальные локальные upstream.
Проверяйте имена переменных в актуальном README Insight: DATABASE_URL обязателен.

Go Gateway требует ACCOUNT_URL, BUSINESS_IMPACT_URL, INSIGHT_URL.
Его Compose использует host network и рассчитан на Linux. На Mac можно запустить
`go run .` с upstream `http://localhost:18400`, `http://localhost:18500`,
`http://localhost:8082`, либо явно настроить Docker-сеть и адреса.

Не подключайте Go Gateway к отсутствующему Insight и не считайте 503 readiness
в этом случае ошибкой работающего Account.

## Проверки

| Репозиторий | Где смотреть команды и prerequisites |
|---|---|
| Account | [README и тесты](https://github.com/Alfa-future-hack-okoAI/Account-Service/tree/main/tests) |
| News | [CI workflow](https://github.com/Alfa-future-hack-okoAI/News-Collector-Service/tree/main/.github/workflows) |
| BI | `poetry run python dev/run_integration_tests.py -q`; изолированные DB/vhost |
| ML | `uv run pytest`; CI использует синтетические CSV и временное состояние |
| Gateway | `go test ./...`, `go test -race ./...`, `go build ./...` |
| Insight | `go test ./...`; PostgreSQL-тестам нужен отдельный TEST_DATABASE_URL |
| Deployment | `python3 -m unittest discover -s tests -v`; валидация manifest/Compose |

Интеграционные проверки запускаются на отдельных тестовых DB/vhost.
Не направляйте синтетические события в рабочие очереди ML и не используйте
production-секреты вместо локальных тестовых значений.
