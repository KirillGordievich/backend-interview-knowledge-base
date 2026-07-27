# Web — Протоколы

## TCP/IP

TCP/IP — стек протоколов для передачи данных по сети.

**Модель TCP/IP (4 уровня):**

| Уровень | Протоколы | Назначение |
|---|---|---|
| Прикладной | HTTP, WebSocket, DNS, FTP | Взаимодействие приложений |
| Транспортный | TCP, UDP | Доставка данных между процессами |
| Сетевой | IP, ICMP | Маршрутизация пакетов между сетями |
| Канальный | Ethernet, Wi-Fi | Передача внутри одной сети |

### TCP vs UDP

| | TCP | UDP |
|---|---|---|
| Соединение | Устанавливается (handshake) | Без соединения |
| Доставка | Гарантирована | Нет гарантий |
| Порядок | Сохраняется | Нет |
| Скорость | Медленнее | Быстрее |
| Использование | HTTP, SSH, PostgreSQL | DNS, стриминг, игры, VoIP |

**TCP 3-way handshake:**
```
Client →  SYN       → Server   (хочу соединение)
Client ← SYN-ACK    ← Server   (принято, подтверждаю)
Client →  ACK       → Server   (подтверждаю)
          [данные]
```

---

## HTTP

HTTP (HyperText Transfer Protocol) — протокол передачи данных. Работает поверх TCP. Клиент делает запрос → сервер возвращает ответ.

**Версии:**

| Версия | Ключевые особенности |
|---|---|
| HTTP/1.0 | Одно соединение на запрос |
| HTTP/1.1 | Keep-Alive, pipelining |
| HTTP/2 | Мультиплексирование (несколько запросов в одном TCP), сжатие заголовков, server push |
| HTTP/3 | Поверх QUIC (UDP), быстрее при нестабильной сети |

**HTTP методы:**

| Метод | Действие | Идемпотентен? | Безопасен? |
|---|---|---|---|
| GET | Получить | Да | Да |
| POST | Создать | Нет | Нет |
| PUT | Заменить целиком | Да | Нет |
| PATCH | Частично обновить | Нет | Нет |
| DELETE | Удалить | Да | Нет |

**Коды ответа:**

| Диапазон | Смысл | Примеры |
|---|---|---|
| 2xx | Успех | 200 OK, 201 Created, 204 No Content |
| 3xx | Редирект | 301 Moved Permanently, 302 Found, 304 Not Modified |
| 4xx | Ошибка клиента | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests |
| 5xx | Ошибка сервера | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable |

---

## HTTPS и TLS

**HTTPS** = HTTP + TLS. Весь трафик шифруется. Стандарт для любого production-сервиса.

**TLS (Transport Layer Security)** — протокол шифрования поверх TCP.

### TLS Handshake (упрощённо)

1. **Client Hello** — клиент отправляет поддерживаемые алгоритмы, случайное число
2. **Server Hello** — сервер выбирает алгоритм, отправляет сертификат
3. **Проверка сертификата** — клиент проверяет по цепочке доверенных CA
4. **Обмен ключами** — через Diffie-Hellman генерируется общий сессионный ключ
5. **Finished** — обе стороны подтверждают, шифрование начато

TLS 1.3 (современный): 1 RTT вместо 2 RTT в TLS 1.2. При повторном соединении — 0-RTT.

**Сертификат:** содержит публичный ключ сервера + подпись CA (Certificate Authority). Клиент доверяет CA из системного хранилища.

---

## WebSockets

Протокол полнодуплексной (двунаправленной) связи поверх TCP. Клиент и сервер могут отправлять данные в любой момент без нового запроса.

**Установка соединения (HTTP Upgrade):**
```http
GET /ws HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==

← 101 Switching Protocols
← Upgrade: websocket
```

**Когда использовать:**
- Чат, уведомления в реальном времени
- Live котировки, биржевые данные
- Онлайн-игры
- Совместное редактирование (Google Docs)

**WebSocket vs HTTP Polling:**

| | HTTP Polling | Long Polling | WebSocket |
|---|---|---|---|
| Соединение | Новое на каждый запрос | Открытое до ответа | Постоянное |
| Задержка | Высокая | Средняя | Минимальная |
| Нагрузка | Много HTTP overhead | Меньше | Минимум |

```python
# FastAPI + WebSocket
from fastapi import WebSocket

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(ws: WebSocket, client_id: str):
    await ws.accept()
    try:
        while True:
            data = await ws.receive_text()
            await ws.send_text(f"Echo: {data}")
    except WebSocketDisconnect:
        pass
```

---

## RPC и gRPC

### RPC (Remote Procedure Call)

Вызов функции на удалённом сервере так, как если бы она была локальной.

```
Client            Network              Server
  │──── call foo(args) ────►│──── execute foo(args) ────►│
  │◄─── return result ──────│◄──── return result ─────────│
```

**RPC vs REST:**
- REST — ресурсно-ориентированный (`GET /users/1`), HTTP/1.1, JSON
- RPC — действие-ориентированный (`getUser(1)`), бинарная сериализация, эффективнее

### gRPC

gRPC — высокопроизводительный RPC-фреймворк от Google. Работает поверх **HTTP/2**, сериализация через **Protocol Buffers**.

**Преимущества:**
- **HTTP/2** — мультиплексирование, сжатие заголовков
- **Protobuf** — бинарный формат, ~3-10x компактнее JSON, быстрый парсинг
- **Строгая типизация** — контракт в `.proto` файле
- **Кодогенерация** — клиент/сервер на 10+ языках из одного файла
- **Streaming** — 4 режима взаимодействия

```protobuf
// user.proto
syntax = "proto3";

service UserService {
    rpc GetUser (GetUserRequest) returns (UserResponse);           // unary
    rpc ListUsers (ListUsersRequest) returns (stream UserResponse); // server streaming
    rpc CreateUsers (stream CreateUserRequest) returns (UserResponse); // client streaming
    rpc Chat (stream Message) returns (stream Message);            // bidirectional
}

message GetUserRequest { int32 id = 1; }
message UserResponse   { int32 id = 1; string name = 2; string email = 3; }
```

```python
# Генерация: python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. user.proto

import grpc
import user_pb2, user_pb2_grpc
from concurrent import futures

# Server
class UserServicer(user_pb2_grpc.UserServiceServicer):
    def GetUser(self, request, context):
        user = db.get(request.id)
        if not user:
            context.set_code(grpc.StatusCode.NOT_FOUND)
            return user_pb2.UserResponse()
        return user_pb2.UserResponse(id=user.id, name=user.name, email=user.email)

server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
user_pb2_grpc.add_UserServiceServicer_to_server(UserServicer(), server)
server.add_insecure_port('[::]:50051')
server.start()

# Client
with grpc.insecure_channel('localhost:50051') as channel:
    stub = user_pb2_grpc.UserServiceStub(channel)
    resp = stub.GetUser(user_pb2.GetUserRequest(id=1))
    print(resp.name)
```

**gRPC vs REST:**

| | gRPC | REST |
|---|---|---|
| Протокол | HTTP/2 | HTTP/1.1 |
| Формат | Protobuf (бинарный) | JSON (текстовый) |
| Контракт | `.proto` (строгий) | OpenAPI (опциональный) |
| Streaming | Да (4 режима) | Ограниченно (SSE) |
| Браузер | Требует proxy (grpc-web) | Нативно |
| Отладка | Сложнее | Легко (curl, Postman) |
| Когда | Микросервисы, внутренние API | Публичные API, веб |

---

## JSON-RPC

Лёгкий протокол RPC поверх JSON. Клиент вызывает методы сервера как функции — без REST-семантики (URL, HTTP-методы).

**Структура запроса:**
```json
{
    "jsonrpc": "2.0",
    "method": "getUser",
    "params": {"id": 123},
    "id": 1
}
```

**Структура ответа:**
```json
{"jsonrpc": "2.0", "result": {"id": 123, "name": "Alice"}, "id": 1}
{"jsonrpc": "2.0", "error": {"code": -32601, "message": "Method not found"}, "id": 1}
```

**Особенности:**
- **Батчинг** — массив запросов в одном HTTP-вызове
- **Уведомления** — запрос без `id`: сервер не отвечает (fire-and-forget)
- **Не привязан к транспорту** — HTTP POST, WebSockets, TCP
- **Слабая типизация** — нет схемы, ошибки в рантайме

```python
# Простой JSON-RPC сервер (FastAPI)
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class RPCRequest(BaseModel):
    jsonrpc: str
    method: str
    params: dict = {}
    id: int | str | None = None

METHODS = {
    "getUser": lambda p: {"id": p["id"], "name": "Alice"},
    "add":     lambda p: p["a"] + p["b"],
}

@app.post("/rpc")
def handle(req: RPCRequest):
    if req.method not in METHODS:
        return {"jsonrpc": "2.0", "error": {"code": -32601, "message": "Method not found"}, "id": req.id}
    result = METHODS[req.method](req.params)
    return {"jsonrpc": "2.0", "result": result, "id": req.id}
```

### JSON-RPC vs REST vs gRPC

| | REST | JSON-RPC | gRPC |
|---|---|---|---|
| Ориентация | Ресурсы (`/users/1`) | Методы (`getUser`) | Методы (строго типизированные) |
| Формат | JSON | JSON | Protobuf (бинарный) |
| Транспорт | HTTP/1.1 | HTTP, WS, TCP | HTTP/2 |
| Типизация | Нет (OpenAPI опционально) | Нет | Строгая (`.proto`) |
| Батчинг | Нет (или вручную) | Да, встроенный | Нет (стриминг вместо) |
| Кэширование | Да (GET + HTTP cache) | Нет | Нет |
| Браузер | Нативно | Нативно | Нужен grpc-web |
| Отладка | Легко (curl, Postman) | Легко (JSON) | Сложнее (бинарный) |
| Когда | Публичные API, CRUD | Блокчейны, внутренние API, батчинг | Микросервисы, high-performance |

**Где используется JSON-RPC:** Ethereum/блокчейны (eth_getBalance), Bitcoin node API, некоторые внутренние микросервисные API.

---

## Zero Copy

Оптимизация передачи файлов: данные идут из OS page cache напрямую в сетевой буфер, минуя user-space.

**Обычный путь (4 копии):**
```
Диск → kernel buffer → user buffer → kernel socket buffer → NIC
```

**Zero Copy через sendfile (2 копии, нет user-space):**
```
Диск → kernel buffer → NIC
```

Используется в nginx (`sendfile on`), Kafka, Java NIO. Значительно снижает CPU при отдаче статики.

```nginx
# nginx
sendfile on;
tcp_nopush on;   # оптимизация: отправлять полными пакетами
```
