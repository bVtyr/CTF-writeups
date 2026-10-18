# Murder (Securinets CTF Quals 2026)

Дали файл `parkour.zip`, который якобы достали с компа жертвы. Надо найти страну рождения убийцы, дату и время убийства и координаты места.

Формат: `Securinets{country_DD/MM/YYYY-HH:MM:SS_lat,lon}`, координаты режем до 4 знаков.

## Файл

Распаковал через 7z, внутри игра на Godot. Достал проект через gdre. В скриптах ничего интересного, полезное нашлось только в `menu.tscn`:

```
text = "created by waterdup45"
```

## itch.io

https://waterdup45.itch.io/parkour



## X

https://x.com/chiggens67



В видео за окном игры открыт Chrome, и во вкладке Reddit пользователя `Street-Compote-5562`

## Reddit

https://www.reddit.com/user/Street-Compote-5562/


### Время

На странице видно только день регистрации, так что точное время создания аккаунта взял через API самого реддита 

```
created_utc = 1791996413 = 14/10/2026 16:46:53 UTC
+33 часа                 = 16/10/2026 01:46:53 UTC
```

### Место

![street view](images/streetview.png)

по геосинт тулзам получил хинт что мальта и поискал насосы через OSM и полистал панорамы на севере острова, нашлось в Меллихе:

https://www.google.com/maps/@35.9622757,14.3678539,3a,90y,45h,105t/data=!3m7!1e1!3m5!1svgis7PuM3W-byoQHatRSGA

После обрезки получается `35.9622,14.3678`.

## Страна

Твит про спидраны намекает на speedrun.com:

https://www.speedrun.com/users/chiggens67



В X в "About this account" стоит Tunisia(откуда кста сами орги поэтому логично), но с ней флаг не заходил

## Флаг

```
Securinets{Brazil_16/10/2026-01:46:53_35.9622,14.3678}
```
