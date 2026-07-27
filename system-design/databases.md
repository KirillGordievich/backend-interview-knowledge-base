# Масштабирование баз данных

## SQL vs NoSQL — когда что

| Критерий | SQL (PostgreSQL, MySQL) | NoSQL (MongoDB, DynamoDB, Redis) |
|---|---|---|
| Данные | Структурированные, связи между сущностями | Гибкие, денормализованные, key-value |
| Схема | Строгая (schema-on-write) | Гибкая (schema-on-read) |
| Транзакции | ACID, multi-table | Ограниченные (обычно per-document) |
| Масштабирование | Вертикальное + read replicas | Горизонтальное (sharding из коробки) |
| Запросы | Сложные JOIN, агрегации | По ключу/индексу, без JOIN |
| Consistency | Strong по умолчанию | Часто eventual |
| Когда | Финансы, e-commerce, сложные связи | Логи, кэш, каталоги, real-time, high write |

**Не бинарный выбор** — в реальных системах часто polyglot persistence:
```
PostgreSQL — основные бизнес-данные (заказы, пользователи)
Redis — кэш, сессии, rate limiting
Elasticsearch — полнотекстовый поиск
ClickHouse/TimescaleDB — аналитика, метрики
MongoDB — гибкие документы (каталоги, конфиги)
```

---

## Replication (репликация)

**Цели:** отказоустойчивость, read scaling, географическое распределение.

### Master-Slave (Primary-Replica)

```
         ┌─── Replica 1 (read)
Write → Master ─┤
         └─── Replica 2 (read)
```

```sql
-- PostgreSQL: streaming replication
-- На master:
ALTER SYSTEM SET wal_level = 'replica';
ALTER SYSTEM SET max_wal_senders = 5;

-- На replica:
-- primary_conninfo = 'host=master port=5432 user=replicator'
```

**Проблемы:**
- **Replication lag** — реплика может быть на секунды позади
- **Read-after-write consistency** — записал → сразу читаю с реплики → не видно
- **Failover** — при падении master нужно промотить реплику

**Решение read-after-write:**
```python
# Читать с master если данные "свежие"
def get_user(user_id: int, just_updated: bool = False) -> User:
    if just_updated:
        return primary_db.query(User).get(user_id)   # read from master
    return replica_db.query(User).get(user_id)       # read from replica
```

### Master-Master (Multi-Primary)

```
Master 1 ←──→ Master 2
(write)         (write)
```

**Проблемы:** конфликты записи (одна запись изменена одновременно на двух мастерах). Нужна стратегия разрешения: last-write-wins, application-level merge, CRDTs.

**Когда:** geo-distributed (мастер в каждом регионе), high write availability.

---

## Sharding (горизонтальное партиционирование)

**Sharding** — разделение данных по нескольким серверам. Каждый шард содержит подмножество данных.

### Стратегии шардирования

**1. Range-based (по диапазону):**
```
Shard 1: user_id 1–1M
Shard 2: user_id 1M–2M
Shard 3: user_id 2M–3M
```
- Плюс: range-запросы эффективны
- Минус: hot spots (популярные диапазоны перегружены)

**2. Hash-based:**
```python
shard_id = hash(user_id) % num_shards
```
- Плюс: равномерное распределение
- Минус: resharding при добавлении шардов (перекладка данных)

**3. Consistent Hashing (кольцевой хеш):**
```
             Node A
            /      \
     Node D    hash ring    Node B
            \      /
             Node C

key → hash → ближайший узел по часовой стрелке
```

```python
import hashlib
from bisect import bisect_right

class ConsistentHash:
    def __init__(self, nodes: list[str], vnodes: int = 150):
        self.ring: list[tuple[int, str]] = []
        for node in nodes:
            for i in range(vnodes):  # виртуальные ноды для равномерности
                key = f"{node}:{i}"
                h = int(hashlib.md5(key.encode()).hexdigest(), 16)
                self.ring.append((h, node))
        self.ring.sort()
        self.keys = [h for h, _ in self.ring]

    def get_node(self, key: str) -> str:
        h = int(hashlib.md5(key.encode()).hexdigest(), 16)
        idx = bisect_right(self.keys, h) % len(self.ring)
        return self.ring[idx][1]

# При добавлении ноды перемещается только ~1/N данных (а не всё)
ring = ConsistentHash(["db1", "db2", "db3"])
print(ring.get_node("user:12345"))  # → "db2"
```

**4. Directory-based:**
```
Lookup Service (metadata)
    user_id=123 → shard_3
    user_id=456 → shard_1
```
- Плюс: полная гибкость
- Минус: single point of failure (lookup), extra hop

### Проблемы шардирования

| Проблема | Описание | Решение |
|---|---|---|
| **Cross-shard queries** | JOIN между шардами невозможен | Денормализация, application-level join |
| **Cross-shard transactions** | Нет единой транзакции | 2PC, Saga pattern |
| **Resharding** | Добавление шардов требует миграции | Consistent hashing, virtual shards |
| **Hot spots** | Один шард перегружен | Rebalancing, split hot shard |
| **Global sequences** | AUTO_INCREMENT не работает глобально | UUID, Snowflake ID, central sequence |

### Snowflake ID

```
| 1 bit (unused) | 41 bits (timestamp ms) | 10 bits (machine id) | 12 bits (sequence) |
```

```python
import time

class SnowflakeGenerator:
    EPOCH = 1609459200000  # 2021-01-01 custom epoch

    def __init__(self, machine_id: int):
        self.machine_id = machine_id & 0x3FF  # 10 bits
        self.sequence = 0
        self.last_ts = 0

    def next_id(self) -> int:
        ts = int(time.time() * 1000) - self.EPOCH
        if ts == self.last_ts:
            self.sequence = (self.sequence + 1) & 0xFFF  # 12 bits
            if self.sequence == 0:
                ts = self._wait_next_ms(ts)
        else:
            self.sequence = 0
        self.last_ts = ts
        return (ts << 22) | (self.machine_id << 12) | self.sequence

    def _wait_next_ms(self, ts):
        while int(time.time() * 1000) - self.EPOCH <= ts:
            pass
        return int(time.time() * 1000) - self.EPOCH
```

---

## Partitioning vs Sharding

| | Partitioning | Sharding |
|---|---|---|
| Расположение | Один сервер, разные файлы/tablespaces | Разные серверы |
| Цель | Ускорение запросов (partition pruning) | Масштабирование нагрузки |
| Пример | PostgreSQL table partitioning | Vitess, Citus, MongoDB sharding |
| Управление | БД управляет сама | Application / middleware |

```sql
-- PostgreSQL: native partitioning
CREATE TABLE orders (
    id BIGINT,
    user_id INT,
    created_at TIMESTAMP,
    amount DECIMAL
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2024_q1 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
CREATE TABLE orders_2024_q2 PARTITION OF orders
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

-- Запрос автоматически обращается только к нужной партиции:
SELECT * FROM orders WHERE created_at = '2024-02-15';
-- → сканирует только orders_2024_q1
```

---

## Connection Pooling

**Проблема:** каждое подключение к PostgreSQL — отдельный процесс (~10MB RAM). 1000 микросервисов × 10 connections = 10000 процессов → OOM.

```
App instances ──→ PgBouncer (pool) ──→ PostgreSQL
(1000 conns)      (50 active conns)    (50 backends)
```

**PgBouncer modes:**

| Mode | Описание | Ограничения |
|---|---|---|
| **session** | Один backend на всю сессию клиента | Безопасно, но неэффективно |
| **transaction** | Backend освобождается после каждой транзакции | Нельзя SET, PREPARE между tx |
| **statement** | Backend освобождается после каждого запроса | Нельзя multi-statement tx |

```ini
# pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
reserve_pool_size = 5
```

**Application-level pooling (SQLAlchemy):**
```python
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql://user:pass@localhost/db",
    pool_size=20,           # постоянных соединений
    max_overflow=10,        # дополнительных при пиковой нагрузке
    pool_timeout=30,        # сколько ждать свободного соединения
    pool_recycle=3600,      # пересоздавать через 1 час (от stale connections)
    pool_pre_ping=True,     # проверять соединение перед использованием
)
```

---

## Read Replicas — паттерны

### CQRS на уровне БД

```python
# Write → primary, Read → replica
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

write_engine = create_engine("postgresql://primary:5432/db")
read_engine = create_engine("postgresql://replica:5432/db")

class UnitOfWork:
    def __init__(self):
        self.write_session = Session(write_engine)
        self.read_session = Session(read_engine)

    def query(self, *args, **kwargs):
        return self.read_session.query(*args, **kwargs)

    def add(self, obj):
        self.write_session.add(obj)

    def commit(self):
        self.write_session.commit()
```

### Automatic routing (Django)

```python
# settings.py
DATABASES = {
    'default': {'HOST': 'primary', ...},
    'replica': {'HOST': 'replica1', ...},
}

# Router
class PrimaryReplicaRouter:
    def db_for_read(self, model, **hints):
        return 'replica'

    def db_for_write(self, model, **hints):
        return 'default'

DATABASE_ROUTERS = ['myapp.routers.PrimaryReplicaRouter']
```

---

## NoSQL паттерны масштабирования

### MongoDB — Auto-Sharding

```javascript
// Включить sharding
sh.enableSharding("mydb")
sh.shardCollection("mydb.orders", { user_id: "hashed" })

// MongoDB автоматически:
// 1. Разбивает данные на chunks (~64MB)
// 2. Балансирует chunks между шардами
// 3. Роутит запросы через mongos
```

### DynamoDB — Partition Keys

```python
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Orders')

# Partition key = равномерно распределять!
# Плохо: partition_key = "2024-01-15" (все запросы одного дня → hot partition)
# Хорошо: partition_key = user_id (распределено)

table.put_item(Item={
    'user_id': '12345',        # partition key
    'order_id': 'ord_abc',     # sort key
    'amount': 99.99,
})

# Single-table design: разные сущности в одной таблице
# PK=USER#123, SK=PROFILE       → профиль
# PK=USER#123, SK=ORDER#001     → заказ
# PK=USER#123, SK=ORDER#002     → заказ
```

---

## Индексирование для масштабирования

**Типы индексов и когда их использовать:**

| Тип | Структура | Когда |
|---|---|---|
| **B-Tree** | Сбалансированное дерево | По умолчанию. =, <, >, BETWEEN, ORDER BY |
| **Hash** | Хеш-таблица | Только = (точное совпадение) |
| **GiST** | Обобщённое дерево | Геоданные, полнотекст, ranges |
| **GIN** | Инвертированный индекс | JSONB, массивы, full-text search |
| **BRIN** | Block Range | Очень большие таблицы с коррелированными данными (timestamp) |

```sql
-- Covering index (index-only scan)
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at)
    INCLUDE (amount, status);
-- Вся нужная информация в индексе → heap не читается

-- Partial index (только для подмножества)
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;
-- Меньше индекс → быстрее → меньше write amplification

-- BRIN для time-series (миллионы строк)
CREATE INDEX idx_logs_time ON logs USING BRIN(created_at);
-- 100x меньше обычного B-tree, работает для коррелированных данных
```

---

## Типичные вопросы на собесе

**Q: Как определить, нужен ли sharding?**
- Single node не справляется (>1TB данных, >50k QPS для write)
- Vertical scaling достиг предела
- Задержки растут из-за объёма данных
- Сначала: read replicas, caching, query optimization, partitioning

**Q: Как мигрировать на sharded архитектуру без downtime?**
```
1. Dual-write: пишем и в старую, и в новую (sharded) БД
2. Backfill: мигрируем исторические данные
3. Shadow reads: читаем из обеих, сравниваем
4. Switch reads: переключаем чтение на sharded
5. Stop dual-write: убираем запись в старую
```

**Q: UUID vs auto-increment в распределённой системе?**
- UUID: не нужен координатор, но плохая locality в B-tree (random), 128 bit
- UUIDv7: timestamp-prefix → хорошая locality, сортируемый, 128 bit
- Snowflake: 64 bit, сортируемый, нужен machine_id
- Auto-increment: нужен single source (sequence table/service)

**Q: Как обеспечить consistency при read replicas?**
- Monotonic reads: привязать пользователя к одной реплике (sticky session)
- Read-your-writes: после write читать с master (по времени/версии)
- Causal consistency: передавать logical timestamp между запросами
