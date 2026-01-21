# KnightCloud — доступ к Premium Analytics без оплаты (KCTF)

**Задача:** KnightCloud 

## Суть
У KnightCloud есть «внутренний» эндпоинт миграции подписки(найден в .js файле), который меняет уровень доступа пользователя (tier). Он оказался доступен из публичного приложения и позволил обычному пользователю получить **Premium**. После смены tier стала доступна **Advanced Analytics**, где и отображается флаг.

## Затронутые эндпоинты
- `/api/internal/v1/migrate/user-tier` — изменение subscription tier пользователя.
- `/api/premium/analytics` — премиум-аналитика (возвращает данные и флаг).

## Решение
1. Вызов «internal» эндпоинта вернул `200 OK` и подтвердил смену tier на `premium`.
 <img width="1666" height="665" alt="image" src="https://github.com/user-attachments/assets/9048d4b9-d998-46da-a64c-ecf4be61f74c" />

2. На странице `/dashboard` появился `Subscription Status: Premium`, а в блоке **Advanced Analytics** отображается флаг.
3. <img width="1744" height="959" alt="image" src="https://github.com/user-attachments/assets/7bd48b92-0936-470c-bb13-681c5cf30dfd" />



## Флаг
`KCTF{pr1v1l3g3_3sc4l4t10n_1s_fun}`
