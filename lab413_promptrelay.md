# Lab 413 - PromptRelay - SQL Injection to RCE via Admin Tools

Приложение: PromptRelay (Nexora AI), шлюз для проксирования запросов к LLM-провайдерам по API-ключам, плюс отдельная админка для управления подключёнными tool-серверами (MCP, LangChain, custom).

Суть: значение из заголовка Authorization: Bearer подставляется в SQL-запрос без параметризации, классическая SQL-инъекция обходит проверку ключа и даёт полные админские права. Дальше один из админских эндпоинтов, test-connection, спроектирован так, что позволяет напрямую выполнять команды на сервере.

## Разведка

Прошёлся по функционалу: Dashboard, Playground, API Keys, Models, Usage, Docs, и отдельная вкладка Admin Tools с пометкой RESTRICTED. Боевые эндпоинты вроде /api/v1/chat/completions требуют Authorization: Bearer <api_key>.

Сначала отправил обычный запрос на /api/v1/chat/completions без ключа, чтобы посмотреть реакцию. Ожидаемо 401, invalid_api_key.

<img width="1562" height="737" alt="image" src="https://github.com/user-attachments/assets/87c5fa83-36e4-4f20-8c1b-02a5f530a8fe" />


## SQL-инъекция и обход авторизации

Вместо ключа подставил в Authorization SQLi-пейлоад:

```
Authorization: Bearer ' OR '1'='1
```

Прошло: 200 OK. В ответе, помимо симулированного ответа модели, бэкенд вернул отладочный блок relay_meta с метаданными о том, под каким ключом ушёл запрос:

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

<img width="1737" height="839" alt="image" src="https://github.com/user-attachments/assets/fa9ba5c2-14d0-466b-a568-9c35b20a8181" />


Одна инъекция в заголовке дала доступ от имени key_id 1, Platform Admin, со всеми правами: admin, admin:tools, admin:config, admin:keys. Проверка ключа собирает SQL-запрос конкатенацией строки, что-то вроде SELECT * FROM api_keys WHERE key = '<value>', и OR '1'='1 делает условие всегда истинным, подставляя первую попавшуюся запись из таблицы ключей.

Тот же ключ работает и на админских эндпоинтах. GET /api/v1/admin/tools/servers с ним отдаёт 200 OK и полный список tool-серверов: Web Search (MCP), Code Interpreter (custom), Document Retrieval (langchain), Image Generator (MCP), со всеми endpoint'ами и статусами.

<img width="1595" height="820" alt="image" src="https://github.com/user-attachments/assets/6932eec8-9dbb-4128-9562-fdeb0057bbb9" />


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

<img width="1280" height="691" alt="image" src="https://github.com/user-attachments/assets/b5354b64-121f-4dab-a466-b53acbe29bdf" />


Это прямой путь к RCE. Вместо легитимного бинаря подставляется интерпретатор и своя команда:

```json
{
  "server_type": "custom",
  "command": "/bin/sh",
  "args": ["-c", "cat /etc/passwd"]
}
```

Отправил такой запрос на POST /api/v1/admin/tools/test-connection с тем же SQLi-ключом, права admin:tools уже подтверждены предыдущими запросами. Получил выполнение произвольной команды на сервере и через тот же вектор прочитал флаг в /root/flag.txt.
