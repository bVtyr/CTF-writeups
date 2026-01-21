# KnightCloud — доступ к Premium Analytics без оплаты (KCTF)

**Задача:** KnightCloud (490 pts)

## Суть
У KnightCloud есть «внутренний» эндпоинт миграции подписки, который меняет уровень доступа пользователя (tier). Он оказался доступен из публичного приложения и позволил обычному пользователю получить **Premium**. После смены tier стала доступна **Advanced Analytics**, где и отображается флаг.

## Затронутые эндпоинты
- `/api/internal/v1/migrate/user-tier` — изменение subscription tier пользователя.
- `/api/premium/analytics` — премиум-аналитика (возвращает данные и флаг).

## Доказательства
1. Вызов «internal» эндпоинта вернул `200 OK` и подтвердил смену tier на `premium`.
2. На странице `/dashboard` появился `Subscription Status: Premium`, а в блоке **Advanced Analytics** отображается флаг.

## Скриншоты
- `screenshots/request-response-user-tier.png`
- `screenshots/premium-dashboard-flag.png`

## Флаг
`KCTF{pr1v1l3g3_3sc4l4t10n_1s_fun}`
