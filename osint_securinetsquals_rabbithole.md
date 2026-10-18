# RabbitHole (Securinets quals CTF)

Надо собрать флаг из 9 вопросов

## 1. Композитор Outer Wilds

https://en.wikipedia.org/wiki/Andrew_Prahlow

<img width="1458" height="812" alt="image" src="https://github.com/user-attachments/assets/65e88d8a-7d8d-4ee6-8b5f-6d562d3b4ca1" />


Ответ: `andrewprahlow`

## 2. Приемная мать персонажа

Prahlow писал музыку к короткометражке GODLIKE про игрока в League of Legends, он указан в описании ролика на YouTube. В первую минуту играют за Riven.

<img width="1456" height="814" alt="image" src="https://github.com/user-attachments/assets/4c4ed7cb-e156-42f7-87db-8f4fe33c5fec" />


По лору у Riven приемные родители Asa и Shava Konte: https://wiki.leagueoflegends.com/en-us/Universe:Riven

Ответ: `shavakonte`

## 3. Место университетского концерта

Прошлый год это 2025. 17 мая 2025 Prahlow играл сет по Outer Wilds с UCLA Game Music Ensemble на концерте Faraway Fantasies

Тут подвох В твите он сам пишет royce hall, но с этим флаг не заходил. На Eventbrite площадкой указан Ackerman Grand Ballroom, и на видео с концерта обычный бальный зал с синими шторами, на royce hall не поход

Ответ: `ackermangrandballroom`

## 4. Лучший фрукт


<img width="1406" height="840" alt="image" src="https://github.com/user-attachments/assets/0a22a80f-4f19-4179-92bd-49ee83524107" />

Ответ: `kumquat`

## 5. Production Manager Assistant

В биографии у Prahlow есть короткометражка для Покемона. Это POKÉTOON PT18 "Bonding with Kangaskhan", стафф есть на Bulbapedia: https://bulbapedia.bulbagarden.net/wiki/PT18
<img width="1456" height="827" alt="image" src="https://github.com/user-attachments/assets/3ad1e1fc-4016-47d2-b36c-71160000d9d0" />


Ответ: `junaoki`

Это Aoki Jun (青木純), глава SPACE NEKO COMPANY. Дальше все вопросы про него.

## 6. Где и когда родился

Тут трабл в том что разные сайты писали по разному, где-то ноябрь 80го, а где-то 81го
В японской Википедии написано 1981年 沖縄県生まれ, то есть Окинава 1981


Ответ: `okinawa1981`

## 7. Куда перенес студию в 2024

Там же в истории компании в 2024 студия переехала в Мусасино (東京都武蔵野市)

Ответ: `musashino`

## 8. Группа крови

В исходнике его блога https://ameblo.jp/aokijun есть поле `bloodtype_text`:

<img width="1548" height="784" alt="image" src="https://github.com/user-attachments/assets/5b6d0db5-ba45-49b7-b544-259639e03b0e" />


Ответ: `A`

## 9. Пост про разбитый телефон

Пост 携帯死亡, где он уронил телефон на платформе станции Икэбукуро: https://ameblo.jp/aokijun/entry-10014009422.html

<img width="1512" height="786" alt="image" src="https://github.com/user-attachments/assets/077a2198-1ed4-4607-b044-3992dbbfb722" />


На странице стоит 2005-12-10 22:30, а в `datePublished` в коде 2005-12-11T02:30 по японскому времени. Заходит 11 число.

Ответ: `2005-12-11`

## Флаг

```
Securinets{andrewprahlow_shavakonte_ackermangrandballroom_kumquat_junaoki_okinawa1981_musashino_A_2005-12-11}
```
