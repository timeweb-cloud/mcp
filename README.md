# Timeweb Cloud MCP Server

Официальный MCP-сервер Timeweb Cloud. Сервер работает на нашей
стороне, ваш AI-клиент подключается к нему по HTTP.

| | |
|---|---|
| **Эндпоинт** | `https://api.timeweb.cloud/api/v1/mcp/search` |
| **Авторизация** | заголовок `Authorization: Bearer <токен>` |
| **Инструменты** | доступны через поиск (`search_tools` → `get_tool_definition` → `execute_tool`) |

Сервер даёт модели доступ к вашему аккаунту Timeweb Cloud: серверы, базы данных, Kubernetes,
приложения, домены, почта, S3, сети, IP, проекты, AI-агенты и базы знаний — чтение, создание и
изменение.

### Какие инструменты видит ваш клиент

В списке инструментов их три. Все остальные операции выполняются через эти три:

| Инструмент | Что делает |
|---|---|
| `search_tools` | Ищет нужную операцию по описанию задачи. Принимает ключевые слова на английском (действие + ресурс, например `create mailbox`), возвращает список подходящих `tool_id` с namespace и краткой сводкой. Шаг 1. |
| `get_tool_definition` | По `tool_id` возвращает JSON-схему аргументов операции: какие поля обязательны, какие типы, что значат. Шаг 2. |
| `execute_tool` | Выполняет операцию: принимает `tool_id` и объект `arguments` по схеме из шага 2. Единственный способ что-либо сделать — здесь же проходит подтверждение изменяющих операций. Шаг 3. |

Пример: на запрос «создай почтовый ящик» модель вызывает `search_tools("create mailbox")` → получает
`tool_id: create_mailbox` → `get_tool_definition("create_mailbox")` → узнаёт обязательные поля →
`execute_tool("create_mailbox", {...})`.

## 1. Получите токен

Панель управления → раздел **[«API и Terraform»](https://timeweb.cloud/my/api-keys)** → выпустите API-токен.

Токен действует от имени вашего аккаунта в объёме прав, выданных ключу. Если хотите ограничить, что
сможет делать модель, выпустите **отдельный ключ с урезанными правами** — сервер учитывает права ключа
и вернёт ошибку на запрещённые категории.

## 2. Подключите клиент

### В один клик

Для клиентов с поддержкой MCP-deeplink подключение укладывается в нажатие кнопки. Клиент откроется, попросит токен из шага 1 и сам сохранит конфиг.

[![Установить в VS Code](https://img.shields.io/badge/VS_Code-Установить-0098FF?style=for-the-badge&logo=visualstudiocode&logoColor=white)](https://vscode.dev/redirect?url=vscode:mcp/install?%7B%22name%22%3A%22timeweb-cloud%22%2C%22type%22%3A%22http%22%2C%22url%22%3A%22https%3A//api.timeweb.cloud/api/v1/mcp/search%22%2C%22inputs%22%3A%5B%7B%22id%22%3A%22timeweb_token%22%2C%22type%22%3A%22promptString%22%2C%22password%22%3Atrue%2C%22description%22%3A%22API-%5Cu0442%5Cu043e%5Cu043a%5Cu0435%5Cu043d%20Timeweb%20Cloud%20%28%5Cu043f%5Cu0430%5Cu043d%5Cu0435%5Cu043b%5Cu044c%20%5Cu0443%5Cu043f%5Cu0440%5Cu0430%5Cu0432%5Cu043b%5Cu0435%5Cu043d%5Cu0438%5Cu044f%20%5Cu2192%20API%20%5Cu0438%20Terraform%29%22%7D%5D%2C%22headers%22%3A%7B%22Authorization%22%3A%22Bearer%20%24%7Binput%3Atimeweb_token%7D%22%7D%7D)
[![Установить в VS Code Insiders](https://img.shields.io/badge/VS_Code_Insiders-Установить-24bfa5?style=for-the-badge&logo=visualstudiocode&logoColor=white)](https://insiders.vscode.dev/redirect?url=vscode-insiders:mcp/install?%7B%22name%22%3A%22timeweb-cloud%22%2C%22type%22%3A%22http%22%2C%22url%22%3A%22https%3A//api.timeweb.cloud/api/v1/mcp/search%22%2C%22inputs%22%3A%5B%7B%22id%22%3A%22timeweb_token%22%2C%22type%22%3A%22promptString%22%2C%22password%22%3Atrue%2C%22description%22%3A%22API-%5Cu0442%5Cu043e%5Cu043a%5Cu0435%5Cu043d%20Timeweb%20Cloud%20%28%5Cu043f%5Cu0430%5Cu043d%5Cu0435%5Cu043b%5Cu044c%20%5Cu0443%5Cu043f%5Cu0440%5Cu0430%5Cu0432%5Cu043b%5Cu0435%5Cu043d%5Cu0438%5Cu044f%20%5Cu2192%20API%20%5Cu0438%20Terraform%29%22%7D%5D%2C%22headers%22%3A%7B%22Authorization%22%3A%22Bearer%20%24%7Binput%3Atimeweb_token%7D%22%7D%7D)
[![Установить в Cursor](https://img.shields.io/badge/Cursor-Установить-000000?style=for-the-badge&logo=cursor&logoColor=white)](https://cursor.com/install-mcp?name=timeweb-cloud&config=eyJ1cmwiOiJodHRwczovL2FwaS50aW1ld2ViLmNsb3VkL2FwaS92MS9tY3Avc2VhcmNoIiwiaGVhZGVycyI6eyJBdXRob3JpemF0aW9uIjoiQmVhcmVyIEFQSV9LRVlfSEVSRSJ9fQ)

VS Code спросит токен в защищённом поле и подставит его в заголовок `Authorization`. Кнопка для Cursor подставит плейсхолдер `API_KEY_HERE` — замените его на свой токен в форме, которую откроет Cursor.

Для Claude Code, Claude Desktop, Windsurf, Zed и других клиентов автоподключения пока нет — там нужен ручной конфиг ниже.

### Claude Code

```bash
claude mcp add --transport http timeweb-cloud \
  https://api.timeweb.cloud/api/v1/mcp/search \
  --header "Authorization: Bearer ВАШ_ТОКЕН"
```

Проверить: `claude mcp list`.

### Cursor

`.cursor/mcp.json` в проекте или глобальный `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "timeweb-cloud": {
      "url": "https://api.timeweb.cloud/api/v1/mcp/search",
      "headers": {
        "Authorization": "Bearer ВАШ_ТОКЕН"
      }
    }
  }
}
```

### VS Code

`.vscode/mcp.json` — токен спрашивается при первом запуске и не попадает в репозиторий:

```json
{
  "inputs": [
    {
      "type": "promptString",
      "id": "timeweb-token",
      "description": "Timeweb Cloud API token",
      "password": true
    }
  ],
  "servers": {
    "timeweb-cloud": {
      "type": "http",
      "url": "https://api.timeweb.cloud/api/v1/mcp/search",
      "headers": {
        "Authorization": "Bearer ${input:timeweb-token}"
      }
    }
  }
}
```

## 3. Подтверждение изменяющих операций

Инструменты помечены в описании, которое возвращает `get_tool_definition`: **73** помечены
`[WRITE]` (меняют состояние), **22** — `[BILLABLE]` (приводят к списанию средств).

Любая такая операция выполняется **в два шага**. Первый вызов ничего не меняет, а возвращает токен
подтверждения:

```json
{
  "status": "confirmation_required",
  "confirm_token": "dbe74d52-a347-44ad-aaab-7402693cc24d",
  "expires_in_seconds": 300,
  "operation_summary": "[WRITE] Обновить VPC network-abda…: name → \"my-vpc\"",
  "next_step": "…вызвать тот же инструмент повторно с теми же аргументами плюс confirm_token…"
}
```

Модель показывает `operation_summary` вам и, получив согласие, повторяет вызов с тем же набором
аргументов и добавленным `confirm_token`. Требования второго шага:

- аргументы должны совпадать с первым вызовом — иначе токен не подойдёт;
- токен живёт **300 секунд**;
- повторный вызов **без** `confirm_token` не выполняет операцию, а выдаёт новый токен.

Так что случайных списаний по одной фразе модели не будет: подтверждение запрашивается всегда.

Если второй шаг не принят, вместо результата придёт один из статусов:

| `status` | Что произошло | Что делать |
|---|---|---|
| `confirmation_expired` | прошло больше 300 секунд, токен уже удалён | начать заново — вызвать инструмент без токена и подтвердить снова |
| `confirmation_args_mismatch` | аргументы отличаются от первого вызова | повторить с исходными аргументами, токен ещё действителен |
| `confirmation_tool_mismatch` | токен выдан для другого инструмента | использовать токен, полученный именно от этого инструмента |

## 4. Ограничения

- **Удаления нет.** Среди инструментов нет ни одного `delete_*`/`remove_*`: сервер не может
  удалить ресурс (убедиться можно через `search_tools`). Удаление — только через панель управления.
- **Изменяющие операции требуют двух шагов** — обойти подтверждение нельзя, см. раздел 3.
- **Права ограничены токеном.** Операции с ресурсами — серверы, базы, Kubernetes, домены, S3,
  сети, приложения, почта, реестры, диски, ключи, проекты, AI-агенты и базы знаний — сервер
  выполняет от имени вашего API-ключа. Если у ключа нет
  нужного доступа, инструмент вернёт ошибку.
