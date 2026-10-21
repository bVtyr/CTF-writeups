# Lab 408 - MailNest - Broken Authentication via Unauthenticated Password Reset

Приложение: MailNest (Nestworks), сервис email-рассылок с админкой (System Config, SMTP, вебхуки и так далее).

Суть: на сайте есть скрытый, не выведенный в UI эндпоинт сброса пароля. Он принимает username и newPassword и меняет пароль без проверки личности, без старого пароля, без токена из письма, без капчи. Можно сбросить пароль хоть admin'у.

## Как нашёл

Сначала потыкал сам функционал сайта, в интерфейсе ничего подозрительного. Полез в JS-бандл страницы логина и там нашёл вызовы к ручкам, которых нет в UI:

- GET /api/v1/admin/config
- POST /api/v1/auth/force-reset-password

Первая при прямом обращении просто редиректила на логин, защищена нормально. Вторая вела себя иначе, отвечала ошибкой с описанием нужных полей, а не редиректом, значит запрос доходит до обработчика.

Первая проверка, пустой POST без тела, чтобы увидеть формат ошибки:

![Пустой запрос, сервер подсказывает нужные поля](images/lab408_01_empty_request.jpg)

400, "Missing required fields: username, newPassword". Поля известны.

## Подбор формата тела

Попытка 1, JSON-тело, но без Content-Type:

```json
{"username":"admin","newPassword":"gazgaz"}
```

Снова 400, та же ошибка, тело просто не распарсилось как JSON.

![JSON-тело без Content-Type, сервер его не видит](images/lab408_02_json_no_contenttype.jpg)

Попытка 2, form-urlencoded, как в старых PHP-формах:

```
username=admin&newPassword=gazgaz
```

Опять 400.

![Form-urlencoded тело тоже не подошло](images/lab408_03_form_urlencoded.jpg)

Попытка 3, тот же JSON, но с заголовком Content-Type: application/json. Дошло, что тело было верным с самого начала, просто сервер не понимал формат без явного content type. Добавил заголовок, и запрос прошёл.

![Content-Type: application/json, сброс пароля прошёл успешно](images/lab408_04_json_success.jpg)

200, success true, пароль admin заменён на gazgaz.

## Эксплуатация

Залогинился под admin/gazgaz, попал в админку MailNest, раздел System Config, доступный только админам. В блоке API & Security в поле System Secret Key открытым текстом лежал флаг.

![Админ-панель MailNest, System Secret Key = флаг](images/lab408_05_admin_panel_flag.jpg)

```
FLAG{2860e96f59ef9b36c1c390bc36fe16e1}
```
