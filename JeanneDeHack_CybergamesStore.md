# Cybergames Store — CTF Writeup 

## TL;DR
Цепочка: **LDAP Injection → JWT → подмена `alg`  → доступ в XML Admin Panel → флаг**.

---

## 1) Стартовая точка
Доступна форма логина. Предположение: авторизация использует LDAP, значит возможна инъекция в LDAP-фильтр.

**Цель:** войти под `admin` и получить JWT.

---

## 2) Уязвимость: LDAP Injection в форме логина

### Что делаем
- `username`: `admin*`  
- `password`: `*`

username=admin*&password=*&login_type=ldap

### Результат
Успешный вход и выдача **JWT** (сессионный токен).

---

## 3)Видим то что мы имеем аккаунт Админа и в своем профиле находим линк admin/adafg541/21232f297a57a5a743894a0e4a801fc3panel.php 
Используя стандартные креды admin:admin заходим в панель и видим что нам нужна роль sqladmin/xmladmin/ldapadmin
---

## 4) Эксплуатация: манипуляции с `alg` и `kid`
Из файла /composer.lock мы находим версию php-jwt 5.1.1

### Идея атаки
Используются две проблемы:
1. **Смена алгоритма**: `RS256 → HS256` (key confusion).


### Что меняем в токене
- В **header**:
  - `alg`: `RS256` → `HS256`

- В **payload**:
  - роль поднимается до `xmladmin`

### Подпись
Токен пересобирается под `HS256`,  используя публичную известную CVE под библиотеку php-jwt

---

## 5) Доступ в XML Admin Panel

С поддельным токеном открывается:

```text
http://cybergames_store.web03.jeanne-hack-ctf.org/admin/adafg541/xml/admin_xml_panel.php
```

**Результат:** панель **XML Admin Panel** становится доступна.

---

## 6) Флаг
```text
JDHACK{J0urn3y_Thr0ugh_JWT_Darkn3ss}
```

---

