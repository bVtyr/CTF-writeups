# Lab 413 - PromptRelay - SQL Injection to RCE via Admin Tools

Приложение: PromptRelay (Nexora AI), шлюз для проксирования запросов к LLM-провайдерам по API-ключам, плюс отдельная админка для управления подключёнными tool-серверами (MCP, LangChain, custom).

Суть: значение из заголовка Authorization: Bearer подставляется в SQL-запрос без параметризации, классическая SQL-инъекция обходит проверку ключа и даёт полные админские права. Дальше один из админских эндпоинтов, test-connection, спроектирован так, что позволяет напрямую выполнять команды на сервере.

## Разведка

Прошёлся по функционалу: Dashboard, Playground, API Keys, Models, Usage, Docs, и отдельная вкладка Admin Tools с пометкой RESTRICTED. Боевые эндпоинты вроде /api/v1/chat/completions требуют Authorization: Bearer <api_key>. Проверил, что без валидного ключа ничего не работает, отправил заведомо сломанный ключ в виде SQLi-пейлоада:

```
Authorization: Bearer ' OR '1'='1
```

Получил честный 401, invalid_api_key.

![Первая попытка SQLi в заголовке, честный 401](images/lab413_01_chatcompletions_401.jpg)

То есть просто подставить кавычку в произвольный заголовок мало, проверка ключа где-то отрабатывает нормально. Нужно было найти запрос, который к инъекции чувствителен.

## SQL-инъекция и обход авторизации

Тот же пейлоад попробовал против GET /api/v1/admin/tools/servers (список tool-серверов, тоже защищён Bearer-ключом и требует админ-прав по докам). Тут сработало: 200 OK и полный список серверов, Web Search (MCP), Code Interpreter (custom), Document Retrieval (langchain), Image Generator (MCP), со всеми endpoint'ами и статусами.

![SQLi в Authorization обошла проверку ключа на admin-эндпоинте](images/lab413_02_admin_tools_servers_sqli.jpg)

Похоже, проверка ключа на части эндпоинтов собирает SQL-запрос конкатенацией строки, что-то вроде SELECT * FROM api_keys WHERE key = '<value>', и OR '1'='1 делает условие всегда истинным, подставляя первую попавшуюся запись из таблицы ключей.

Проверил тот же пейлоад на POST /api/v1/chat/completions, на этот раз прошло. В ответе, помимо симулированного ответа модели, бэкенд вернул отладочный блок relay_meta с метаданными о том, под каким ключом ушёл запрос:

```json
"relay_meta": {
  "routed_to": "openai",
  "key_owner": "Platform Admin",
  "key_id": 1,
  "permissions": "read,write,admin,admin:tools,admin:config,admin:keys",
  "latency_ms": 142,
  "gateway_version": "1.4.2"
}
```

![Та же SQLi на chat/completions, доступ от имени Platform Admin](images/lab413_03_chatcompletions_admin_sqli.jpg)

Одна инъекция в заголовке дала доступ от имени key_id 1, Platform Admin, со всеми правами: admin, admin:tools, admin:config, admin:keys.

## От админки к RCE

С правами admin:tools пошёл смотреть /docs. В левом меню нашлись разделы Admin: Tool Servers и Admin: Test Connection, которых нет в основной публичной спецификации API:

```
POST /api/v1/admin/tools/test-connection   [ADMIN:TOOLS REQUIRED]
```

Для server_type mcp эндпоинт ждёт host и port, а для server_type custom принимает произвольные command и args, которые уходят на выполнение на бэкенде для проверки доступности сервера:

```json
{
  "server_type": "custom",
  "command": "/usr/local/bin/tool-server",
  "args": ["--check", "--timeout", "5"]
}
```

![Документация test-connection, custom server_type принимает command/args](images/lab413_04_docs_test_connection.jpg)

Это прямой путь к RCE. Вместо легитимного бинаря подставляется интерпретатор и своя команда:

```json
{
  "server_type": "custom",
  "command": "/bin/sh",
  "args": ["-c", "cat /etc/passwd"]
}
```

Отправил такой запрос на POST /api/v1/admin/tools/test-connection с тем же SQLi-ключом, права admin:tools уже подтверждены предыдущими запросами. Получил выполнение произвольной команды на сервере и через тот же вектор прочитал флаг в /root/flag.txt.
