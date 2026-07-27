# Web — Аутентификация и авторизация

## Authentication vs Authorization

| | Authentication (Аутентификация) | Authorization (Авторизация) |
|---|---|---|
| Вопрос | Кто ты? | Что тебе разрешено? |
| Проверяет | Личность | Права доступа |
| Результат | Идентифицирован как пользователь X | Действие разрешено/запрещено |
| Примеры | Логин/пароль, OAuth, биометрия | RBAC, ACL, scopes |

Аутентификация **всегда** предшествует авторизации.

---

## Cookie-based Sessions

1. Пользователь логинится → сервер создаёт сессию в хранилище (Redis, БД), генерирует `session_id`
2. Сервер возвращает `Set-Cookie: session_id=abc123; HttpOnly; Secure`
3. Браузер автоматически отправляет cookie в каждом запросе
4. Сервер по `session_id` получает данные сессии из хранилища

**Флаги cookie:**
- `HttpOnly` — недоступен из JavaScript (защита от XSS)
- `Secure` — только по HTTPS
- `SameSite=Strict/Lax` — защита от CSRF

**Плюсы:** просто инвалидировать (удалить из Redis)
**Минусы:** требует централизованного хранилища, overhead на lookup при каждом запросе

---

## JWT (JSON Web Token)

Самодостаточный токен с данными пользователя. Сервер **не хранит состояние** — проверяет подпись.

**Структура:** `header.payload.signature` (три части base64url через точку)

```
eyJhbGciOiJIUzI1NiJ9          ← header: {"alg": "HS256"}
.eyJzdWIiOiIxMjM0IiwicO...    ← payload: {"sub": "1234", "role": "admin", "exp": ...}
.SflKxwRJSMeKKF2QT4fwpM       ← HMAC-SHA256(header.payload, secret)
```

**Как работает:**
1. Логин → сервер создаёт JWT, подписывает секретом (`HS256`) или приватным ключом (`RS256`)
2. Клиент хранит токен (localStorage или httpOnly cookie)
3. Запрос: `Authorization: Bearer <token>`
4. Сервер проверяет подпись — без обращения к БД

```python
import jwt
from datetime import datetime, timedelta

SECRET = "my-secret-key"

def create_token(user_id: int) -> str:
    payload = {
        "sub": str(user_id),
        "exp": datetime.utcnow() + timedelta(minutes=15),
        "iat": datetime.utcnow(),
    }
    return jwt.encode(payload, SECRET, algorithm="HS256")

def verify_token(token: str) -> dict:
    return jwt.decode(token, SECRET, algorithms=["HS256"])
    # Бросает jwt.ExpiredSignatureError, jwt.InvalidTokenError
```

### Access + Refresh токены

- **Access token** — короткоживущий (15 мин), для API-запросов, в заголовке
- **Refresh token** — долгоживущий (7–30 дней), хранится в `httpOnly` cookie, только для получения нового access токена

**Плюсы JWT:** stateless, масштабируется горизонтально, хорошо для микросервисов
**Минусы:** нельзя инвалидировать до истечения без blacklist в Redis

### Sessions vs JWT

| | Session | JWT |
|---|---|---|
| Хранение состояния | Сервер (Redis) | Клиент (токен) |
| Инвалидация | Сразу (удалить из Redis) | Только через blacklist |
| Масштабирование | Нужен общий Redis | Stateless, просто |
| Размер передаваемых данных | Только session_id | Весь токен (~500 байт) |

---

## CORS (Cross-Origin Resource Sharing)

Браузерный механизм безопасности: запросы к **другому origin** блокируются по умолчанию.

**Origin** = протокол + домен + порт: `https://example.com:443`

**Как работает:**
1. Браузер добавляет `Origin: https://my-app.com` к запросу
2. Сервер возвращает `Access-Control-Allow-Origin: https://my-app.com`
3. Если origin не в списке — браузер **блокирует ответ** (запрос дошёл до сервера, но клиент данные не получит)

**Preflight (OPTIONS):** для «сложных» запросов (PUT/DELETE, кастомные заголовки) браузер сначала спрашивает разрешение:
```http
OPTIONS /api/orders HTTP/1.1
Origin: https://my-app.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Authorization

← 204 No Content
← Access-Control-Allow-Origin: https://my-app.com
← Access-Control-Allow-Methods: GET, POST, PUT, DELETE
```

```python
# FastAPI
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://my-frontend.com"],   # не * в production!
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
    allow_credentials=True,
)
```

**Важно:** CORS — защита браузера. `curl`, Postman, серверный код CORS не блокируют.
