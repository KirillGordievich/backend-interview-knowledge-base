# Кэширование

## Зачем кэш

Кэш — быстрое хранилище (обычно в памяти) для результатов дорогих операций.

**Когда кэшировать:**
- Данные читаются намного чаще чем записываются
- Запрос дорогой (БД, внешний API, сложные вычисления)
- Допустима некоторая устаревшесть данных (stale data)

**Виды кэша:**
- **In-process** — в памяти приложения (`lru_cache`, словарь)
- **Distributed** — Redis, Memcached (разделяется между несколькими инстансами)
- **CDN** — кэш статики у пользователя
- **HTTP cache** — `Cache-Control`, `ETag` на уровне браузера/proxy

---

## Алгоритмы вытеснения (Eviction Policies)

| Алгоритм | Что вытесняется | Когда использовать |
|---|---|---|
| **LRU** (Least Recently Used) | Дольше всего не запрашивался | Общий случай, работа с "горячими" данными |
| **LFU** (Least Frequently Used) | Наименее часто запрашивался | Долгосрочные паттерны доступа |
| **FIFO** | Самый старый (первый вошёл) | Простота, потоковые данные |
| **TTL** | Истёк срок жизни | Данные теряют актуальность со временем |
| **Random** | Случайный элемент | Простота реализации |

---

## LRU Cache

**LRU (Least Recently Used)** — при переполнении вытесняется элемент, к которому дольше всего не обращались.

**Реализация:** двусвязный список + хэш-таблица → O(1) get/put.

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache: OrderedDict[int, int] = OrderedDict()

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)   # помечаем как "недавно использованный"
        return self.cache[key]

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)  # удаляем самый "старый"

# Python: встроенный декоратор
from functools import lru_cache

@lru_cache(maxsize=128)
def get_user(user_id: int) -> dict:
    return db.query(user_id)   # будет вызван только при cache miss
```

---

## LFU Cache

**LFU (Least Frequently Used)** — при переполнении вытесняется элемент с наименьшим счётчиком обращений. При равном счётчике — наиболее старый (LRU tie-breaking).

**Реализация:** хэш-таблица ключей + хэш-таблица частот + двусвязные списки → O(1) get/put.

```python
from collections import defaultdict, OrderedDict

class LFUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.min_freq = 0
        self.key_to_val:  dict[int, int]           = {}
        self.key_to_freq: dict[int, int]           = {}
        self.freq_to_keys: dict[int, OrderedDict]  = defaultdict(OrderedDict)
        # freq_to_keys[f] — упорядоченный словарь ключей с частотой f
        # OrderedDict используется для LRU tie-breaking (FIFO внутри частоты)

    def _increment(self, key: int) -> None:
        freq = self.key_to_freq[key]
        self.key_to_freq[key] = freq + 1
        del self.freq_to_keys[freq][key]
        if not self.freq_to_keys[freq]:          # список опустел
            del self.freq_to_keys[freq]
            if self.min_freq == freq:
                self.min_freq += 1
        self.freq_to_keys[freq + 1][key] = None

    def get(self, key: int) -> int:
        if key not in self.key_to_val:
            return -1
        self._increment(key)
        return self.key_to_val[key]

    def put(self, key: int, value: int) -> None:
        if self.capacity <= 0:
            return
        if key in self.key_to_val:
            self.key_to_val[key] = value
            self._increment(key)
            return
        if len(self.key_to_val) >= self.capacity:
            # вытесняем: самый редкий + самый старый
            evict, _ = self.freq_to_keys[self.min_freq].popitem(last=False)
            del self.key_to_val[evict]
            del self.key_to_freq[evict]
        self.key_to_val[key] = value
        self.key_to_freq[key] = 1
        self.freq_to_keys[1][key] = None
        self.min_freq = 1


# Использование
cache = LFUCache(2)
cache.put(1, 10)   # freq: {1: 1}
cache.put(2, 20)   # freq: {1: 1, 2: 1}
cache.get(1)       # → 10, freq: {1: 2, 2: 1}
cache.put(3, 30)   # вытесняем 2 (min_freq=1), freq: {1: 2, 3: 1}
cache.get(2)       # → -1 (вытеснен)
```

**LRU vs LFU:**

| | LRU | LFU |
|---|---|---|
| Вытесняет | Самый давно не запрашивавшийся | Самый редко запрашивавшийся |
| Проблема | Вытесняет популярные, но временно не запрашиваемые | Новые элементы вытесняются слишком легко (freq=1) |
| Когда лучше | "Горячие" данные — недавние | Стабильные долгосрочные паттерны доступа |
| Реализация | Проще | Сложнее |

---

## Redis — распределённый кэш

```python
import redis
import json
from functools import wraps

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Базовые операции
r.set('user:1', json.dumps({"id": 1, "name": "Alice"}), ex=3600)  # TTL 1 час
user = json.loads(r.get('user:1') or 'null')

# Cache-aside паттерн
def get_user(user_id: int) -> dict:
    key = f"user:{user_id}"
    cached = r.get(key)
    if cached:
        return json.loads(cached)     # cache hit
    user = db.get(user_id)           # cache miss → идём в БД
    r.set(key, json.dumps(user), ex=300)
    return user

# Декоратор для кэширования
def cache(ttl: int = 300):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            key = f"{func.__name__}:{args}:{kwargs}"
            cached = r.get(key)
            if cached:
                return json.loads(cached)
            result = func(*args, **kwargs)
            r.set(key, json.dumps(result), ex=ttl)
            return result
        return wrapper
    return decorator

@cache(ttl=60)
def get_product(product_id: int) -> dict:
    return db.get_product(product_id)
```

---

## Паттерны кэширования

### Cache-Aside (Lazy Loading)

Приложение само управляет кэшем. Самый распространённый паттерн.

```
Read:  App → Cache miss → DB → App → Cache.set → return
       App → Cache hit  → return

Write: App → DB.update → Cache.delete (invalidate)
```

**Плюсы:** данные в кэше только те, что реально запрашивались.
**Минусы:** первый запрос всегда медленный (cache miss); риск stale data.

### Write-Through

При каждой записи в БД — одновременно обновляется кэш.

```
Write: App → DB.update + Cache.update → return
Read:  App → Cache hit (всегда актуально)
```

**Плюсы:** кэш всегда актуален.
**Минусы:** кэшируются данные, которые могут никогда не прочитаться.

### Write-Behind (Write-Back)

Приложение пишет в кэш, а в БД асинхронно через очередь.

```
Write: App → Cache.update → (async) → DB.update
```

**Плюсы:** низкая задержка записи.
**Минусы:** риск потери данных при падении кэша до синхронизации с БД.

---

## Инвалидация кэша

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

**Стратегии:**

```python
# TTL — самоинвалидация
r.set('user:1', data, ex=300)   # через 5 минут устареет само

# Event-driven — инвалидация при изменении
def update_user(user_id, data):
    db.update(user_id, data)
    r.delete(f"user:{user_id}")  # сбросить кэш

# Tag-based — группы ключей
r.sadd("tag:user:1", "user:1:profile", "user:1:orders")  # тег → список ключей
def invalidate_user(user_id):
    keys = r.smembers(f"tag:user:{user_id}")
    if keys:
        r.delete(*keys)
        r.delete(f"tag:user:{user_id}")
```

---

## Проблемы кэширования

### Cache Stampede (Dog Pile Effect)

Когда TTL истекает у популярного ключа — много запросов одновременно идут в БД.

```python
import threading

_lock = threading.Lock()

def get_with_lock(key, fetch_fn, ttl):
    cached = r.get(key)
    if cached:
        return json.loads(cached)
    with _lock:   # только один поток идёт в БД
        cached = r.get(key)   # double-check
        if cached:
            return json.loads(cached)
        result = fetch_fn()
        r.set(key, json.dumps(result), ex=ttl)
        return result
```

### Cache Penetration

Запросы за данными, которых нет ни в кэше, ни в БД (например, `user:-1`).

```python
# Кэшировать null-результат
result = db.get(key)
r.set(cache_key, json.dumps(result), ex=60)  # кэшируем даже None/{}
```

### Cache Avalanche

Много ключей истекают одновременно → spike нагрузки на БД.

```python
import random

# Добавить случайный jitter к TTL
base_ttl = 300
ttl = base_ttl + random.randint(-30, 30)
r.set(key, value, ex=ttl)
```
