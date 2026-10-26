# Эксплуатация и диагностика

## Последовательность проверки

1. Проверить доступность API и ready.
2. Войти в Account, проверить users/me и доступ к компании.
3. Создать задание с новым UUID Idempotency-Key.
4. Дождаться Account ready или изучить failed/error.
5. Проверить BI sync именно этого request_id.
6. После synced проверить результат по request_id, ленту компании и отдельную карточку.

Readiness подтверждает доступность необходимых зависимостей, но не означает,
что конкретное задание обработалось и прошло проверку бизнес-контракта.

## Два разных статуса

| Account status | Что означает |
|---|---|
| pending | Задание сохранено, публикация ещё не подтверждена |
| queued | Outbox подтвердил публикацию, ответа ещё нет |
| ready | ML прислал completed |
| failed | ML прислал failed; смотрите error |

| BI sync state | Что означает |
|---|---|
| waiting | Пока есть только requested или только finished |
| pending | Обе части есть, worker ещё не начал импорт |
| processing | Данные загружаются и проверяются |
| retry | Попытка не удалась, повтор запланирован |
| synced | Результат записан в BI; отдельно проверьте analysis.status |
| blocked | Исчерпаны попытки, нужна диагностика |

При текущем default до 12 попыток HTTP-импорта. Состояние хранится в БД и
переживает рестарт worker; бесконечный перезапуск контейнера не устраняет
ошибку контракта.

## Частые симптомы

| Симптом | Что проверить |
|---|---|
| 401 Account/BI | Срок JWT, issuer/audience, подпись; обновить через refresh/login |
| Service-Key не работает в публичном BI | Caddy удаляет header; используйте Bearer |
| 403 | Текущая связь user↔company, роль и автор задания; запрос выполняет тот же пользователь |
| Account pending долго | outbox-worker, next_attempt_at, RabbitMQ publisher confirms |
| Account queued долго | Consumer ML в vhost account, зависимость LLM, ошибки ML |
| Account ready, BI 404 | sync; ready Account не подтверждает импорт BI |
| BI waiting | Отсутствующая часть события; время создания binding и отдельная очередь BI |
| BI retry/blocked ValidationError | Формат статьи/карточки/контекста не прошёл Pydantic; точный путь поля нужен из контролируемого воспроизведения |
| MLContractError company_id mismatch | Сверить UUID и U_*, branch/version ML, result_id/context_version |
| Новость есть, rank=null | Для выбранного профиля нет видимой карточки; это не оценка 0 |
| unaccounted | Карточка есть, но статус ML «не учтено»; не то же самое, что not_analyzed |
| Пустая лента | Доступ, автор, роль, period, timestamp статей, наличие импортированного анализа |
| 502/503 Gateway | Upstream DNS/порт, readiness, timeout; сравнить прямой внутренний запрос |
| GHCR read_package | Deployment Actions Read на конкретный приватный пакет |
| TLS timeout LLM | Отличать DNS/TCP/TLS от HTTP 401; API-ключ применяется после TLS |

## Как разбирать ошибку импорта

`GET /v1/analyses/{request_id}/sync` возвращает last_error как класс ошибки.
Это намеренно не полный ответ LLM и не бизнес-контекст. По одной строке
ValidationError нельзя установить виноватое поле или обвинить Keycloak/БД.

Сопоставьте requested/finished и HTTP-read ML с теми же ID, ролью и контекстом.
В контролируемой среде получите Pydantic errors с loc/type без входных текстов
и секретов. Убедитесь, что воспроизведение не коммитит частичный результат.
После исправления причины:

```http
POST /api/business-impact-service/v1/analyses/REQUEST_UUID/sync/retry
Authorization: Bearer ACCESS_TOKEN
```

Это повтор готового импорта, не новая генерация ML. Если ML сам вернул failed,
нужна отдельная диагностика и новый запрос Account с новым ключом.

## Наблюдение на сервере

После штатного SSH-входа как deploy доступны read-only команды:

```bash
docker ps --format '{{.Names}} {{.Image}} {{.Status}}'
docker logs --tail 100 okoai-account-outbox-1
docker logs --tail 100 okoai-account-ai-results-1
docker logs --tail 100 okoai-bi-ml-results-1
docker logs --tail 100 okoai-ml-1
```

Не публикуйте полный docker inspect/env или приватный ml.env: там credentials.
Для проверки версии извлекайте только image/revision labels.
RabbitMQ management и внутренние БД доступны через согласованный приватный
доступ, а не публичные API-маршруты. Чтение Get messages с ack удаляет сообщение
из рабочей очереди; для наблюдения используйте counts/consumers и логи.

## Данные и обновления

Generated-пароли должны сохраняться между релизами. Если старый CSV перестал
подходить, сравните окружение/пользователя и bootstrap-процедуру; не перегенерируйте
массово credentials как первый способ диагностики.
Резервируйте PostgreSQL, MinIO и ML state перед миграциями данных. Image rollback
не восстанавливает БД и не отменяет изменённые карточки или PDF.
