# Apache Kafka

## Архитектура

```
Producers → [Topic: partitions] → Consumers (Consumer Groups)
                   ↕
              ZooKeeper / KRaft (метаданные, лидеры)
```

**Ключевые компоненты:**

| Компонент | Роль |
|---|---|
| **Broker** | Сервер Kafka, хранит партиции |
| **Topic** | Именованный лог событий |
| **Partition** | Часть топика, единица параллелизма и упорядоченности |
| **Offset** | Позиция сообщения внутри партиции (монотонно растёт) |
| **Consumer Group** | Группа Consumer-ов, делящих партиции между собой |
| **Leader / Follower** | Лидер обрабатывает записи, фолловеры реплицируют |

---

## Topic и Partitions

```
Topic "orders" (3 partitions, replication factor 2):

Partition 0: [msg0] [msg3] [msg6] ...   → Broker 1 (leader), Broker 2 (replica)
Partition 1: [msg1] [msg4] [msg7] ...   → Broker 2 (leader), Broker 3 (replica)
Partition 2: [msg2] [msg5] [msg8] ...   → Broker 3 (leader), Broker 1 (replica)
```

**Порядок гарантирован только внутри партиции.**

**Выбор партиции:**
- Если key задан → `hash(key) % num_partitions` (одинаковый key → одна партиция)
- Если key не задан → round-robin между партициями

```python
# Все события для одного пользователя → одна партиция (порядок гарантирован)
producer.send(
    topic='user-events',
    key=str(user_id).encode(),   # детерминированная маршрутизация
    value=json.dumps(event).encode()
)
```

---

## Consumer Groups

```
Topic "orders" (3 partitions):

Group "payments" (2 consumers):
    Consumer A → Partition 0, 1
    Consumer B → Partition 2

Group "analytics" (1 consumer):
    Consumer C → Partition 0, 1, 2  (получает всё)
```

**Правила:**
- Один Consumer в группе может читать несколько партиций
- Одну партицию в группе читает только один Consumer
- Если Consumer-ов больше чем партиций — лишние простаивают

---

## Offsets

Consumer сам управляет своей позицией чтения.

```
Partition 0: [0] [1] [2] [3] [4] [5] ...
                              ↑
                       committed offset = 4
                       (следующее для чтения = 4)
```

**Стратегии сброса offset (`auto.offset.reset`):**
- `earliest` — читать с начала топика
- `latest` — читать только новые сообщения (после подписки)

```python
from aiokafka import AIOKafkaConsumer

consumer = AIOKafkaConsumer(
    'orders',
    bootstrap_servers='localhost:9092',
    group_id='payments-service',
    auto_offset_reset='earliest',
    enable_auto_commit=False   # ручной commit для надёжности
)
await consumer.start()

async for msg in consumer:
    try:
        await process(msg.value)
        await consumer.commit()   # commit только после успешной обработки
    except Exception:
        # не коммитим → сообщение будет перечитано
        pass
```

---

## Producer

```python
from aiokafka import AIOKafkaProducer
import asyncio, json

async def produce():
    producer = AIOKafkaProducer(
        bootstrap_servers='localhost:9092',
        acks='all',                    # ждать подтверждения всех реплик
        enable_idempotence=True,       # exactly-once на стороне Producer
        compression_type='gzip',       # сжатие
        max_batch_size=16384,
        linger_ms=10                   # буферизация до 10ms для батчинга
    )
    await producer.start()
    try:
        # Синхронная отправка с подтверждением
        record_metadata = await producer.send_and_wait(
            topic='orders',
            key=b'user-123',
            value=json.dumps({"order_id": 456}).encode(),
            headers=[("source", b"api")]
        )
        print(f"Partition: {record_metadata.partition}, Offset: {record_metadata.offset}")
    finally:
        await producer.stop()
```

**`acks` параметр:**
- `0` — не ждать подтверждения (максимальный throughput, нет гарантий)
- `1` — лидер подтвердил запись (по умолчанию)
- `all` / `-1` — все in-sync реплики подтвердили (максимальная надёжность)

---

## Retention и хранение

Kafka хранит сообщения независимо от того, прочитал их Consumer или нет.

```bash
# Конфигурация топика
kafka-topics.sh --alter --topic orders \
  --config retention.ms=604800000 \   # 7 дней
  --config retention.bytes=1073741824 # 1 GB на партицию

# Компактирование (хранить только последнее значение для каждого key)
kafka-topics.sh --create --topic user-profiles \
  --config cleanup.policy=compact
```

**Log compaction:** полезно для event sourcing и кэша — хранится только актуальное состояние по ключу.

---

## Гарантии доставки

| Гарантия | Настройки Producer | Настройки Consumer |
|---|---|---|
| At-most-once | `acks=0` | `auto.commit=true`, commit до обработки |
| At-least-once | `acks=all` | `auto.commit=false`, commit после обработки |
| Exactly-once | `enable.idempotence=true` + транзакции | `isolation.level=read_committed` |

### Транзакции (Exactly-once)

```python
producer = AIOKafkaProducer(
    bootstrap_servers='localhost:9092',
    transactional_id='my-producer-1'   # уникальный ID для транзакций
)
await producer.start()
await producer.begin_transaction()
try:
    await producer.send('output-topic', key=b'k', value=b'v')
    await producer.commit_transaction()
except Exception:
    await producer.abort_transaction()
```

---

## Rebalancing

При добавлении/удалении Consumer или появлении новых партиций — **rebalance**: перераспределение партиций в группе.

**Во время rebalance:** обработка останавливается (stop-the-world).

**Стратегии (partition.assignment.strategy):**
- `RangeAssignor` — по диапазонам (по умолчанию)
- `RoundRobinAssignor` — равномерно
- `StickyAssignor` — минимизирует перемещения при ребалансировке
- `CooperativeStickyAssignor` — incremental rebalance (без stop-the-world)

---

## Consumer Lag

Разница между последним offset в партиции и committed offset Consumer-а.

```bash
# Проверить lag
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group payments-service

# OUTPUT:
# GROUP            TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# payments-service orders   0          1000            1050            50
# payments-service orders   1          800             800             0
```

**Мониторинг:** `kafka_consumer_group_lag` в Prometheus → алерт при lag > порога.

---

## Kafka Streams / ksqlDB

```python
# Kafka Streams (Java/Scala) — обработка прямо в топиках
# Python аналог: faust

import faust

app = faust.App('my-app', broker='kafka://localhost')

orders_topic = app.topic('orders', value_type=dict)
paid_topic = app.topic('paid-orders', value_type=dict)

@app.agent(orders_topic)
async def process_orders(orders):
    async for order in orders:
        if order['status'] == 'paid':
            await paid_topic.send(value=order)

# faust worker -A myapp worker
```

---

## Типичные паттерны

### Event Sourcing

```
User action → Kafka topic (append-only log) → State rebuilding from events
```

Kafka — идеальное хранилище событий: retention, replay, упорядоченность.

### CQRS + Kafka

```
Command → Write Service → [Kafka] → Read Service → materialized view (Redis/Postgres)
```

### Saga через Kafka (Choreography)

```
OrderService  → "order.created"  → InventoryService
InventoryService → "stock.reserved" → PaymentService
PaymentService → "payment.failed" → InventoryService (компенсация)
InventoryService → "stock.released" → OrderService (компенсация)
```

---

## RabbitMQ vs Kafka — итог

| | RabbitMQ | Kafka |
|---|---|---|
| Модель | Smart broker | Smart consumer (dumb broker) |
| Хранение | До ACK Consumer-а | По retention policy (независимо) |
| Replay | Нет | Да (с любого offset) |
| Throughput | ~50-100k msg/s | Миллионы msg/s |
| Задержка | Низкая (ms) | Чуть выше |
| Порядок | В очереди | В партиции |
| Routing | Гибкий (exchanges) | По key → partition |
| Consumer Groups | — | Да |
| Когда | Task queues, RPC, сложный routing | Event log, аналитика, микросервисы |
