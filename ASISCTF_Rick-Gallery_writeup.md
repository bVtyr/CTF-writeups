# Rick Gallery — LFI через `Image` header + localhost-only endpoint


## Суть задания

Приложение показывает случайную картинку. При `POST` оно также читает HTTP‑заголовок `Image` и **прокидывает его** во внутренний запрос на `http://localhost/getpic.php`.  
`getpic.php` недоступен извне (только localhost), но внутри он делает `file_get_contents()` по переданному пути и возвращает результат в base64.

Итог: можно заставить сервер читать локальные файлы и получать их содержимое (LFI), обходя ограничения через особенности фильтра.

---

## Анализ исходников

### `index.php`

- Забирает `Image` из заголовков.
- Применяет примитивный blacklist: блокирует подстроки `http://`, `https://`, `file://`, `php://`, `data://`, `expect://`, а также `passwd`.
- Режет `../` и `..\`.
- Если фильтр “сработал” — берёт случайную картинку, иначе использует значение `Image`.

Дальше всегда выполняет запрос:

```php
$ch = curl_init("http://localhost:80/getpic.php");
curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query(["picture_name" => $selected]));
$encodedImage = curl_exec($ch);
```

### `.htaccess`

`getpic.php` ограничен “только localhost”:

```apache
<Files "getpic.php">
    Require local
</Files>
```

### `getpic.php`

```php
$data = file_get_contents($_POST['picture_name']);
echo base64_encode("$data");
```

То есть `file_get_contents()` читает как локальные пути, так и wrapper’ы (если включены).

---

## Обход фильтра

Фильтр в `index.php` — **case-sensitive** (`strpos`), поэтому достаточно “сломать” регистр в схеме.

Например:

- `http://` → `htTp://`
- `file://` → `fIle://`
- `php://` → `pHp://`

Также `../` запрещён, но для абсолютных путей он не нужен.

---

## Эксплуатация

Получение флага

В этой инстанции флаг лежал по пути:

```
/tmp/flag.txt
```
<img width="1882" height="322" alt="image" src="https://github.com/user-attachments/assets/3e73a90a-ce94-421f-8512-441ddc3e8c87" />


Декодируем base64 и получаем флаг:


<img width="364" height="36" alt="image" src="https://github.com/user-attachments/assets/1dd06ba2-29d4-42fb-ab23-bc8ee70c19d1" />

---

## Почему это работает

- `getpic.php` читает путь напрямую через `file_get_contents()`.
- Внешний доступ к `getpic.php` закрыт, но `index.php` сам ходит туда по `localhost`, и мы контролируем параметр `picture_name` через заголовок `Image`.
- Блэклист в `index.php` ломается простым изменением регистра у wrapper’а.
