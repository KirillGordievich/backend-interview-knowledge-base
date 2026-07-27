# Rate Limiting и Throttling

## Зачем rate limiting

- **Защита от DDoS** — ограничение количества запросов от одного клиента
- **Справедливое распределение** — один клиент не забирает все ресурсы
- **Защита downstream-сервисов** — от перегрузки БД, внешних API
- **Cost control** — ограничение затрат на платные API
- **Compliance** — SLA гарантии для разных тарифов

---

## Алгоритмы

### 1. Token Bucket

Самый популярный. Токены добавляются с фиксированной скоростью, каждый запрос забирает токен. Позволяет burst.

```python
import time
from threading import Lock

class TokenBucket:
    def __init__(self, rate: float, capacity: int):
        self.rate = rate            # токенов/сек
        self.capacity = capacity    # максимум накопленных
        self.tokens = capacity
        self.last_refill = time.monotonic()
        self.lock = Lock()

    def allow(self, tokens: int = 1) -> bool:
        with self.lock:
            now = time.monotonic()
            elapsed = now - self.last_refill
            self.tokens = min(self.capacity, self.tokens + elapsed * self.rate)
            self.last_refill = now

            if self.tokens >= tokens:
                self.tokens -= tokens
                return True
            return False

# 10 запросов/сек, burst до 20
bucket = TokenBucket(rate=10, capacity=20)

if bucket.allow():
    process_request()
else:
    return Response(status=429, body="Too Many Requests")
```

**Свойства:**
- Burst: до `capacity` запросов мгновенно
- Steady-state: `rate` запросов/сек
- Простота реализации

---

### 2. Leaky Bucket

Запросы попадают в "ведро" (очередь). Обработка с фиксированной скоростью. Сглаживает трафик — без burst.

```python
from collections import deque
import time
from threading import Lock

class LeakyBucket:
    def __init__(self, rate: float, capacity: int):
        self.rate = rate            # запросов/сек (скорость вытекания)
        self.capacity = capacity    # размер очереди
        self.queue: deque = deque()
        self.last_leak = time.monotonic()
        self.lock = Lock()

    def allow(self) -> bool:
        with self.lock:
            now = time.monotonic()
            # "Вытекание" — удалить обработанные
            leaked = int((now - self.last_leak) * self.rate)
            for _ in range(min(leaked, len(self.queue))):
                self.queue.popleft()
            if leaked > 0:
                self.last_leak = now

            if len(self.queue) < self.capacity:
                self.queue.append(now)
                return True
            return False
```

**Token Bucket vs Leaky Bucket:**

| | Token Bucket | Leaky Bucket |
|---|---|---|
| Burst | Да (до capacity) | Нет (строго равномерно) |
| Поведение | Пропускает burst, потом ограничивает | Сглаживает трафик |
| Аналогия | Копилка с монетками | Дырявое ведро |
| Где | API rate limiting, AWS, Nginx | Traffic shaping, сети |

---

### 3. Fixed Window Counter

Считаем количество запросов в фиксированном окне (например, 1 минута). Просто, но уязвимо к boundary burst.

```python
import time

class FixedWindow:
    def __init__(self, limit: int, window_sec: int):
        self.limit = limit
        self.window_sec = window_sec
        self.counters: dict[str, tuple[int, int]] = {}  # key → (count, window_start)

    def allow(self, key: str) -> bool:
        now = int(time.time())
        window_start = now - (now % self.window_sec)

        count, start = self.counters.get(key, (0, window_start))
        if start != window_start:
            count, start = 0, window_start

        if count < self.limit:
            self.counters[key] = (count + 1, start)
            return True
        return False
```

**Проблема boundary burst:**
```
Window 1: [............50 req]  ← последние 50 за секунду
Window 2: [50 req............]  ← первые 50 за секунду
→ 100 req за 2 секунды при лимите 50/окно
```

---

### 4. Sliding Window Log

Храним timestamp каждого запроса. Точнее, но дороже по памяти.

```python
from collections import deque
import time

class SlidingWindowLog:
    def __init__(self, limit: int, window_sec: int):
        self.limit = limit
        self.window_sec = window_sec
        self.logs: dict[str, deque] = {}

    def allow(self, key: str) -> bool:
        now = time.time()
        if key not in self.logs:
            self.logs[key] = deque()

        window = self.logs[key]
        # Удалить старые записи
        while window and window[0] <= now - self.window_sec:
            window.popleft()

        if len(window) < self.limit:
            window.append(now)
            return True
        return False
```

---

### 5. Sliding Window Counter (гибрид)

Комбинирует Fixed Window + интерполяцию. Хороший баланс точности и расхода памяти.

```python
import time

class SlidingWindowCounter:
    def __init__(self, limit: int, window_sec: int):
        self.limit = limit
        self.window_sec = window_sec
        self.prev_count: dict[str, int] = {}
        self.curr_count: dict[str, int] = {}
        self.prev_window: dict[str, int] = {}

    def allow(self, key: str) -> bool:
        now = int(time.time())
        curr_window = now - (now % self.window_sec)
        prev_window = curr_window - self.window_sec

        # Сбросить, если новое окно
        if self.prev_window.get(key) != prev_window:
            self.prev_count[key] = self.curr_count.get(key, 0)
            self.curr_count[key] = 0
            self.prev_window[key] = prev_window

        # Взвешенная сумма: часть предыдущего окна + текущее
        elapsed_in_window = (now % self.window_sec) / self.window_sec
        weight = 1 - elapsed_in_window
        estimated = self.prev_count.get(key, 0) * weight + self.curr_count.get(key, 0)

        if estimated < self.limit:
            self.curr_count[key] = self.curr_count.get(key, 0) + 1
            return True
        return False
```

---

## Сравнение алгоритмов

| Алгоритм | Память | Точность | Burst | Сложность |
|---|---|---|---|---|
| Token Bucket | O(1) на ключ | Хорошая | Да | Простая |
| Leaky Bucket | O(n) очередь | Высокая | Нет | Средняя |
| Fixed Window | O(1) на ключ | Низкая (boundary) | Да (2x) | Очень простая |
| Sliding Window Log | O(n) записей | Идеальная | Нет | Средняя |
| Sliding Window Counter | O(1) на ключ | Хорошая (~99.7%) | Минимальный | Простая |

---

## Distributed Rate Limiting (Redis)

В распределённой системе (несколько инстансов) нужен общий счётчик.

### Token Bucket через Redis + Lua

```lua
-- rate_limit.lua (атомарная операция)
local key = KEYS[1]
local rate = tonumber(ARGV[1])       -- tokens/sec
local capacity = tonumber(ARGV[2])    -- max tokens
local now = tonumber(ARGV[3])         -- current timestamp
local requested = tonumber(ARGV[4])   -- tokens to consume

local data = redis.call("HMGET", key, "tokens", "last_refill")
local tokens = tonumber(data[1]) or capacity
local last_refill = tonumber(data[2]) or now

-- Refill
local elapsed = math.max(0, now - last_refill)
tokens = math.min(capacity, tokens + elapsed * rate)

-- Check
local allowed = 0
if tokens >= requested then
    tokens = tokens - requested
    allowed = 1
end

redis.call("HMSET", key, "tokens", tokens, "last_refill", now)
redis.call("EXPIRE", key, math.ceil(capacity / rate) + 1)

return allowed
```

```python
import redis
import time

r = redis.Redis()

# Загрузить Lua-скрипт
SCRIPT = open("rate_limit.lua").read()
rate_limit_sha = r.script_load(SCRIPT)

def is_allowed(user_id: str, rate: float = 10, capacity: int = 20) -> bool:
    return bool(r.evalsha(
        rate_limit_sha,
        1,                  # numkeys
        f"rl:{user_id}",   # KEYS[1]
        rate,              # ARGV[1]
        capacity,          # ARGV[2]
        time.time(),       # ARGV[3]
        1,                 # ARGV[4] — 1 token
    ))
```

### Sliding Window через Redis ZSET

```python
import redis
import time

r = redis.Redis()

def sliding_window_rate_limit(key: str, limit: int, window_sec: int) -> bool:
    now = time.time()
    window_start = now - window_sec
    pipe = r.pipeline()

    # Удалить старые записи
    pipe.zremrangebyscore(key, 0, window_start)
    # Посчитать текущие
    pipe.zcard(key)
    # Добавить текущий запрос
    pipe.zadd(key, {f"{now}": now})
    # TTL для автоочистки
    pipe.expire(key, window_sec)

    results = pipe.execute()
    current_count = results[1]

    if current_count >= limit:
        # Откатить добавление (необязательно — и так будет очищен)
        r.zrem(key, f"{now}")
        return False
    return True
```

---

## Уровни применения

```
Client → CDN/WAF → API Gateway → Application → Database
         ↑           ↑              ↑            ↑
         DDoS       Per-user       Per-endpoint  Connection pool
         protection  rate limit    throttling    limits
```

| Уровень | Инструмент | Что ограничивает |
|---|---|---|
| **CDN / WAF** | Cloudflare, AWS Shield | DDoS, IP-based |
| **API Gateway** | Kong, AWS API GW, Nginx | Per-user, per-API key |
| **Application** | Middleware, decorator | Per-endpoint, per-action |
| **Database** | Connection pool (PgBouncer) | Max connections |

---

## Rate Limiting в API Gateway (Nginx)

```nginx
# Определить зону лимитирования
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
# $binary_remote_addr — по IP (16 байт на запись)
# zone=api:10m — 10MB shared memory (~160000 IP)
# rate=10r/s — 10 запросов/сек

server {
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        # burst=20 — допускает burst до 20
        # nodelay — не задерживает burst-запросы (отвечает сразу или 429)

        limit_req_status 429;
        limit_req_log_level warn;

        proxy_pass http://backend;
    }
}
```

---

## Rate Limiting в FastAPI

```python
from fastapi import FastAPI, Request, HTTPException
from functools import wraps
import time
import redis

app = FastAPI()
r = redis.Redis()

def rate_limit(limit: int = 60, window: int = 60):
    """Decorator: limit requests per user per window"""
    def decorator(func):
        @wraps(func)
        async def wrapper(request: Request, *args, **kwargs):
            # Идентификация клиента
            client_id = request.headers.get("X-API-Key") or request.client.host
            key = f"rl:{func.__name__}:{client_id}"

            # Sliding window counter
            now = time.time()
            pipe = r.pipeline()
            pipe.zremrangebyscore(key, 0, now - window)
            pipe.zcard(key)
            pipe.zadd(key, {str(now): now})
            pipe.expire(key, window)
            results = pipe.execute()

            count = results[1]
            if count >= limit:
                retry_after = window - int(now % window)
                raise HTTPException(
                    status_code=429,
                    detail="Too Many Requests",
                    headers={
                        "Retry-After": str(retry_after),
                        "X-RateLimit-Limit": str(limit),
                        "X-RateLimit-Remaining": "0",
                        "X-RateLimit-Reset": str(int(now) + retry_after),
                    }
                )

            return await func(request, *args, **kwargs)
        return wrapper
    return decorator

@app.get("/api/data")
@rate_limit(limit=100, window=60)   # 100 req/min
async def get_data(request: Request):
    return {"data": "..."}
```

---

## HTTP-заголовки Rate Limiting (RFC 6585 / draft-ietf-httpapi-ratelimit-headers)

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1625097600
```

| Header | Описание |
|---|---|
| `Retry-After` | Секунд до снятия ограничения |
| `X-RateLimit-Limit` | Максимум запросов в окне |
| `X-RateLimit-Remaining` | Осталось запросов |
| `X-RateLimit-Reset` | Unix timestamp сброса окна |

---

## Стратегии при превышении лимита

| Стратегия | Описание | Когда |
|---|---|---|
| **Reject (429)** | Сразу отклонить | API, жёсткие лимиты |
| **Queue** | Поставить в очередь, обработать позже | Background jobs |
| **Throttle** | Замедлить (добавить delay) | Потоковая обработка |
| **Degrade** | Вернуть упрощённый ответ (кэш) | Graceful degradation |
| **Backoff signal** | Ответить с Retry-After | Клиент сам ретраит |

---

## Типичные вопросы

**Q: Как rate-limit-ить в serverless (Lambda)?**
Через внешнее хранилище (Redis/DynamoDB), т.к. нет shared state между invocations.

**Q: Token Bucket vs Sliding Window — что выбрать?**
- Token Bucket: нужен burst (API для клиентов)
- Sliding Window: нужна строгая равномерность (защита БД)

**Q: Как rate-limit по нескольким измерениям?**
```python
# Лимит по IP + по user + по endpoint одновременно
checks = [
    is_allowed(f"ip:{ip}", rate=100, window=60),
    is_allowed(f"user:{user_id}", rate=1000, window=3600),
    is_allowed(f"endpoint:{user_id}:{path}", rate=10, window=60),
]
if not all(checks):
    raise HTTPException(429)
```
