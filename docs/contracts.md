# Межсервисные контракты

Действующий поток: News → ML, Account → ML, ML → Account и BI,
затем BI → HTTP ML для получения полного пакета.
В Cloud.ru это **один RabbitMQ с двумя vhost**, а не обязательные два сервера.
В локальных Compose могут использоваться разные RabbitMQ-контейнеры.

## Маршрутизация

| Vhost | Exchange / тип | Routing key | Очередь / получатель |
|---|---|---|---|
| news | `news.events` / topic | `news.collected.v1` | `news.collected` → ML |
| news | `news.events` / topic | `news.article.processed.v1` | `business-impact.ml-results` → BI |
| news | `news.events` / topic | `news.article.processed.v1` | `news.article.processed` — дополнительная очередь ML topology |
| account | `account-service.ai` / direct | `company.ai-preparation.requested.v1` | `ml-service.company-ai-preparations` → ML |
| account | тот же exchange | requested/completed/failed `.v1` | `business-impact.ai-preparations` → BI |
| account | тот же exchange | completed/failed `.v1` | `account-service.ai-results` → Account |

Очереди consumer независимы: BI не читает очередь ML или Account.
Две подписки на одну очередь делят сообщения между consumer, а не создают копии.
Bind на exchange создаёт отдельный поток для каждой очереди.

Account queues — durable/quorum, dead-letter exchange `account-service.ai.failed`.
BI имеет отдельные `.retry` и `.dead` очереди. Точные имена/аргументы задаются
конфигурацией и [topology BI](https://github.com/Alfa-future-hack-okoAI/Business-Impact-Service/blob/c8c01a3bb069eb2f89691822c1116ccfd4b92f34/src/modules/impacts/infrastructure/broker/topology.py).
Дополнительные legacy-binding `ml.results` не означают, что текущий ML
публикует `company.analysis.finished.v1` или processed v2: нативный путь — v1 ниже.

## 1. Собранная новость

Producer: News-Collector. Routing key: `news.collected.v1`.
Envelope: `event_id`, `event_type="news.collected"`, `version=1`, `occurred_at`, `news`.
В news — UUID, source/external_id, title/content/url, даты, качество загрузки.

event_id идентифицирует доставку события, news.id — публикацию. Не заменяйте
их строкой URL. Стабильность исходной публикации определяется source/external_id.

## 2. Запрос подготовки компании

Producer: Account. Routing key: `company.ai-preparation.requested.v1`.
Подготовка доступна только для компаний из файлов. HTTP-поля Чекко, источник
и признак доступности аналитики не включаются в этот ML-контракт.

| Поле | Смысл |
|---|---|
| `event_id=request_id` | UUID созданного Account задания |
| `event_type`, `version`, `occurred_at` | Вид команды, версия 1, дата с timezone |
| `company_id` | UUID компании Account |
| `requested_by`, `role` | UUID пользователя, роль в компании |
| `context_version` | SHA-256 полного context |
| `selection` | lookback_hours и max_news; default 168 и 5 |
| `context` | company, payments, top_counterparties, counterparties_total, data_warnings, sources, ml_context |

`company_id` совпадает с `context.company.id` и `context.payments.company_id`.
В новых командах есть ml_context; исторический снимок может его не содержать.
Суммы и доли Decimal — строки, неизвестное значение — null. Не подменяйте null нулём.

Канонический hash:

```python
"sha256:" + hashlib.sha256(json.dumps(
    context, ensure_ascii=False, sort_keys=True, separators=(",", ":")
).encode("utf-8")).hexdigest()
```

Нельзя менять context после вычисления hash, добавлять к нему роль или
переименовывать поля перед сохранением исходного снимка.

## 3. Завершение подготовки

Producer: ML. Routing keys: `company.ai-preparation.completed.v1` и
`company.ai-preparation.failed.v1`. В body event_type не содержит суффикс `.v1`;
версия задаётся отдельным числом.

Общие поля: event_id, event_type, version, occurred_at, request_id, company_id,
role, context_version. event_id уведомления отличается от request_id.

Успех:

```json
{
  "result": {
    "result_id": "00000000-0000-4000-8000-000000000001",
    "news_revision": "n1:example",
    "pipeline_version": "ml-0.3+example",
    "generated_at": "2026-10-26T09:05:00Z"
  },
  "error": null
}
```

Это фрагмент envelope. Полных новостей и карточек в completed нет.
При failed: `result=null`, `error={code, message, retryable}`.

AMQP для Account-потока: persistent JSON UTF-8, message_id=event_id,
correlation_id=request_id; type — CompanyAiPreparationRequested/Completed/Failed.
Согласуйте транспортные metadata с body: Account/BI могут отклонить противоречие.

## 4. Обработанная новость

Producer: ML. Routing key: `news.article.processed.v1`.

Тело: event_id, event_type=`news.article.processed`, version=1, article, cards.
article содержит summary, importance, классификацию, templates и sentences.
`cards` — **массив строковых ID**, а не полных объектов. При AUTO_CARDS=0 он
обычно пуст; карточки по команде Account получаются другим потоком.

## 5. Как BI забирает полные данные

После requested и completed BI читает:

1. `/v1/results/{result_id}?userId={requested_by}`.
2. `/v1/feed` с UUID компании, user_id, role, period=30d, scope=company, collapse=false.
3. `/v1/summary` с теми же компанией/автором/ролью/периодом.
4. `/v1/companies/{company_id}/context`.
5. Недостающие `/v1/cards/{card_id}` и все нужные `/v1/news/{news_id}`.

BI проверяет result_id/company_id/role/context_version/news_revision/pipeline_version,
связи card↔news и актуальность контекста. Raw HTTP-ответы сохраняются вместе с
нормализованными данными. BI сохраняет состояние объектов на момент импорта;
при параллельном обновлении ML версии объектов могут различаться.

## Повторная доставка и совместимость

- У News и Account outbox допускает повтор доставки; consumer дедуплицирует event_id.
- BI ждёт обе части команды/результата в любом порядке; результат не связывается
  только по company_id, иначе параллельные задания смешаются.
- Старые публикации до появления binding не воспроизводятся автоматически.
- У ML news-потока есть окно между отметкой события done и публикацией processed;
  полноценного отдельного transactional outbox на этом участке нет. Проверка
  «сообщение исчезло из очереди» не доказывает, что BI получил статью.
- `polish` меняет company_id в HTTP на U_*, но не в finished. Схема envelope v1
  при этом совместима, а текущий HTTP-import BI — нет.

## Где найти точные контракты

- [События Account-Service](https://github.com/Alfa-future-hack-okoAI/Account-Service/tree/main/contracts).
- [Контракты и примеры BI](https://github.com/Alfa-future-hack-okoAI/Business-Impact-Service/tree/main/contracts).
- [Интеграция ML](https://github.com/Alfa-future-hack-okoAI/ml/blob/dev/docs/INTEGRATION.md).
