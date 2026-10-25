# Авторизация и секреты

## Пользовательский вход

```text
Браузер → Account /auth/login → Keycloak
Браузер ← access_token + refresh_token
Браузер → Gateway → Account / BI с Authorization: Bearer access_token
```

Account/BI проверяют RS256-подпись по настроенному JWKS, срок действия,
issuer, audience и нужные claims. BI дополнительно обращается в Account с тем
же Bearer и проверяет текущий доступ к компании и роль. Данные user_id/role
из браузерных служебных заголовков не являются источником полномочий.

Production issuer:
`https://auth.alfa-future-hack-okoai.ru/realms/account-hack`.
Audience: `account-api`. Внутренний URL загрузки JWKS может быть Docker-адресом
Keycloak; это не меняет публичный issuer токена.

## Срок жизни и обновление

Обычная исходная настройка access token — 5 минут. Для проверки стенда временно
установлен client override `account-login` на **35 минут** (5 + 30), сохранено
исходное отсутствие override. Это операционное состояние на дату документации,
а не обязательство API всегда выдавать токен на 35 минут.

Клиент использует expires_in из login/refresh и обновляет токен:

```http
POST /api/account-service/auth/refresh
Content-Type: application/json

{"refresh_token":"REFRESH_TOKEN"}
```

Изменение lifespan действует на новые токены; exp уже выданного JWT не меняется.
Возврат настройки — удалить временный client attribute `access.token.lifespan`
в Keycloak после проверки текущего значения. Тогда снова действует realm default.
Перезапуск Account для такой настройки не требуется.

## Внутренний Service-Key BI

Header `X-Service-Key`, значение — приватный SERVICE_API_KEY BI.
Это доверенный межсервисный доступ, без ограничений конечного пользователя.
Ключ не должен попадать в frontend bundle, примеры Docs или браузерный proxy.

Публичный Caddy и Go Gateway удаляют входной X-Service-Key. Поэтому внешний
Swagger BI используйте с Bearer. Для внутренних сервисов предусмотрен отдельный
Docker-сетевой путь. Если указан Authorization, BI выбирает пользовательский
режим; невалидный JWT не заменяется корректным Service-Key.

## ML, Insight и Gateway

- ML серверного стенда опубликован без JWT. Не трактуйте его userId как
  доказательство авторизации; это параметр выбора профиля.
- product/polish защищает пути ML через Nginx auth_request в Account users/me.
- Insight делегирует авторизацию BI и затем использует подтверждённую identity;
  собственной JWKS-проверки в нём нет.
- Go Gateway маршрутизирует Bearer, но сам его не проверяет.

Не объединяйте эти различные механизмы в утверждение «все сервисы проверяют JWT».

## Где находятся пароли и ключи

Account PostgreSQL хранит телефон, идентичность и связи, без поля password.
Keycloak хранит credentials в собственной БД в форме хешей, а не обратимо
зашифрованного пользовательского пароля.
У демонстрационного bootstrap есть отдельное исключение: сгенерированный
users_credentials.csv содержит исходные демопароли для выдачи тестировщикам.
Его нельзя пересоздавать при каждом релизе или публиковать в Git.

В Cloud.ru приватные env и generated-данные находятся в
`/home/deploy/okoai/shared`. Значения ключей — только у владельцев окружения.
GitHub App private key и deploy SSH key — в Actions Secrets; package Read —
настройка GHCR. Публичные JSON-schema и image digest секретами не являются.

Эта страница описывает реализованные границы, а не сертификат безопасности
стенда. Текущее открытое ML API и демонстрационные credentials — явные особенности
окружения, которые нужно учитывать перед внешним production-применением.
