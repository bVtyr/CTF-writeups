# Lab 408 - MailNest - Broken Authentication via Unauthenticated Password Reset

Приложение: MailNest (Nestworks), сервис email-рассылок с админкой (System Config, SMTP, вебхуки и так далее).

Суть: на сайте есть скрытый, не выведенный в UI эндпоинт сброса пароля. Он принимает username и newPassword и меняет пароль без проверки личности, без старого пароля, без токена из письма, без капчи. Можно сбросить пароль хоть admin'у.

## Как нашёл

Сначала потыкал сам функционал сайта, в интерфейсе ничего подозрительного. Полез в JS-бандл страницы логина и там нашёл вызовы к ручкам, которых нет в UI:

- GET /api/v1/admin/config
- POST /api/v1/auth/force-reset-password

Первая при прямом обращении просто редиректила на логин, защищена нормально. Вторая вела себя иначе, отвечала ошибкой с описанием нужных полей, а не редиректом, значит запрос доходит до обработчика.

Первая проверка, пустой POST без тела, чтобы увидеть формат ошибки:

<img width="1667" height="827" alt="image" src="https://github.com/user-attachments/assets/28278761-9a2c-4482-b676-47a3ce6c0298" />


400, "Missing required fields: username, newPassword". Поля известны.

## Подбор формата тела

Попытка 1, JSON-тело, но без Content-Type:

```json
{"username":"admin","newPassword":"gazgaz"}
```

Снова 400, та же ошибка, тело просто не распарсилось как JSON.

<img width="1574" height="689" alt="image" src="https://github.com/user-attachments/assets/d04fd1a6-f280-4486-a7eb-a8aa9119fc4b" />


Попытка 2, form-urlencoded, как в старых PHP-формах:

```
username=admin&newPassword=gazgaz
```

Опять 400.

<img width="1558" height="661" alt="image" src="https://github.com/user-attachments/assets/d11e2013-5932-4a9a-a0a8-46e9cf4799a8" />


Попытка 3, тот же JSON, но с заголовком Content-Type: application/json. Дошло, что тело было верным с самого начала, просто сервер не понимал формат без явного content type. Добавил заголовок, и запрос прошёл.

<img width="1607" height="711" alt="image" src="https://github.com/user-attachments/assets/5bb80555-111c-4f0c-a2e7-6a50b120745e" />


200, success true, пароль admin заменён на gazgaz.

## Эксплуатация

Залогинился под admin/gazgaz, попал в админку MailNest, раздел System Config, доступный только админам. В блоке API & Security в поле System Secret Key открытым текстом лежал флаг.

<img width="1893" height="904" alt="image" src="https://github.com/user-attachments/assets/ac2f55ce-2991-4bab-8d37-e73ad4d9b99d" />


```
FLAG{2860e96f59ef9b36c1c390bc36fe16e1}
```
