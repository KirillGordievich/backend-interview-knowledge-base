# Паттерны коммуникации между сервисами

## Sync vs Async

| | Синхронная | Асинхронная |
|---|---|---|
| Протокол | REST, gRPC | Message Queue, Events |
| Связность | Высокая (temporal coupling) | Низкая |
| Latency | Немедленный ответ | Eventual |
| Отказоустойчивость | Если downstream упал — caller тоже | Очередь буферизирует |
| Когда | Нужен ответ прямо сейчас | Fire-and-forget, длинные процессы |

```
Sync:  OrderService ──HTTP──► PaymentService ──HTTP──► NotificationService
       (ждёт ответа)          (ждёт ответа)

Async: OrderService ──event──► Queue ──► PaymentService
                                    ──► NotificationService
       (не ждёт, продолжает работу)
```

**Правило:** если можешь — делай async. Sync только когда ответ нужен немедленно для продолжения работы.

---

## Event-Driven Architecture (EDA)

**Идея:** сервисы общаются через события. Продюсер не знает, кто получит событие. Получатели подписываются самостоятельно.

```
OrderService                  EventBus (Kafka/RabbitMQ)
    │                              │
    ├── emit("order.created") ──►  ├──► PaymentService
    │                              ├──► InventoryService
    │                              ├──► NotificationService
    │                              └──► AnalyticsService
```

**Преимущества:**
- Слабая связность (loose coupling)
- Легко добавлять новые потребителей без изменения продюсера
- Resilience: если consumer упал — событие в очереди, обработается позже

**Проблемы:**
- Сложнее отлаживать (distributed tracing обязателен)
- Eventual consistency
- Event ordering
- Дублирование сообщений → нужна идемпотентность

---

## Event Sourcing

**Вместо хранения текущего состояния — хранить последовательность событий.** Состояние = replay всех событий.

```python
# Вместо: UPDATE accounts SET balance = 150 WHERE id = 1
# Храним:
events = [
    {"type": "account_created", "data": {"id": 1, "balance": 0}},
    {"type": "deposited",       "data": {"id": 1, "amount": 200}},
    {"type": "withdrawn",       "data": {"id": 1, "amount": 50}},
]
# current_balance = 0 + 200 - 50 = 150

class Account:
    def __init__(self):
        self.balance = 0
        self.events: list = []

    def apply(self, event: dict):
        match event["type"]:
            case "deposited":
                self.balance += event["data"]["amount"]
            case "withdrawn":
                self.balance -= event["data"]["amount"]
        self.events.append(event)

    def replay(self, events: list[dict]):
        for event in events:
            self.apply(event)
```

**Когда нужен Event Sourcing:**
- Аудит (полная история изменений): финансы, медицина
- Replay/восстановление: отладка, миграции
- Temporal queries: "какое было состояние на дату X?"
- Разные проекции данных для разных потребителей

**Когда НЕ нужен:**
- CRUD без аудита
- Простые приложения
- Когда не нужна история изменений

---

## CQRS + Event Sourcing

**CQRS (Command Query Responsibility Segregation):** разделение модели записи и модели чтения.

```
Command (Write)                     Query (Read)
    │                                   │
    ▼                                   ▼
Command Handler                   Read Model (проекция)
    │                                   ▲
    ▼                                   │
Event Store ──── events ────► Projector ─┘
(append-only)                (обновляет read model)
```

```python
# Event Store (write side)
class EventStore:
    def __init__(self):
        self.events: list[dict] = []

    def append(self, stream_id: str, event: dict):
        self.events.append({
            "stream_id": stream_id,
            "sequence": len(self.events),
            "timestamp": time.time(),
            **event
        })

    def get_events(self, stream_id: str) -> list[dict]:
        return [e for e in self.events if e["stream_id"] == stream_id]


# Read Model (query side) — оптимизирована для чтения
class OrderReadModel:
    """Денормализованная проекция для быстрых запросов"""
    def __init__(self):
        self.orders: dict[str, dict] = {}  # id → flat dict

    def handle_event(self, event: dict):
        match event["type"]:
            case "order_created":
                self.orders[event["order_id"]] = {
                    "id": event["order_id"],
                    "status": "created",
                    "total": event["total"],
                    "items_count": len(event["items"]),
                }
            case "order_paid":
                self.orders[event["order_id"]]["status"] = "paid"
            case "order_shipped":
                self.orders[event["order_id"]]["status"] = "shipped"
                self.orders[event["order_id"]]["tracking"] = event["tracking_number"]

    def get_order(self, order_id: str) -> dict:
        return self.orders.get(order_id)

    def get_orders_by_status(self, status: str) -> list[dict]:
        return [o for o in self.orders.values() if o["status"] == status]
```

**Преимущества CQRS:**
- Read model оптимизирована для запросов (денормализована, предвычислена)
- Write model — для бизнес-логики (нормализована, валидации)
- Можно масштабировать read и write независимо
- Несколько read-проекций для разных потребителей

---

## Choreography vs Orchestration

### Choreography (хореография)

Каждый сервис реагирует на события и публикует свои. Нет центрального координатора.

```
OrderService ── "order.created" ──►
    PaymentService ── "payment.completed" ──►
        InventoryService ── "items.reserved" ──►
            ShippingService ── "shipment.created" ──►
                NotificationService → email
```

**Плюсы:** простота, нет single point of failure, слабая связность.
**Минусы:** сложно понять полный flow, трудно менять порядок, нет retry на уровне процесса.

### Orchestration (оркестрация)

Центральный сервис (orchestrator/saga) управляет последовательностью шагов.

```python
class OrderOrchestrator:
    async def process_order(self, order_id: str):
        try:
            # Последовательность шагов, управляемая из одного места
            payment = await self.payment_service.charge(order_id)
            inventory = await self.inventory_service.reserve(order_id)
            shipment = await self.shipping_service.create(order_id)
            await self.notification_service.send_confirmation(order_id)
        except PaymentFailed:
            await self.compensate(order_id, steps_completed=0)
        except InventoryFailed:
            await self.payment_service.refund(order_id)
            await self.compensate(order_id, steps_completed=1)
```

**Плюсы:** весь flow виден в одном месте, легко менять, retry, компенсации.
**Минусы:** single point of failure (orchestrator), тесная связность с ним.

| | Choreography | Orchestration |
|---|---|---|
| Связность | Минимальная | Средняя (через orchestrator) |
| Видимость flow | Распределена (сложнее) | Централизована (проще) |
| Масштабирование | Лучше | Orchestrator = bottleneck |
| Error handling | Каждый сам | Orchestrator управляет |
| Когда | Простые flows, <5 шагов | Сложные flows, компенсации |

---

## Backpressure

**Backpressure** — механизм, при котором медленный потребитель сигнализирует продюсеру "притормози".

**Зачем:** без backpressure быстрый продюсер переполняет буферы → OOM, потеря данных.

**Стратегии:**

| Стратегия | Описание | Пример |
|---|---|---|
| **Drop** | Отбросить лишнее | UDP, метрики |
| **Buffer** | Буфер ограниченного размера | Kafka (retention), Redis Streams |
| **Block** | Заблокировать продюсера | TCP flow control, bounded queue |
| **Sample** | Пропустить часть | Logging с sampling |
| **Throttle** | Замедлить продюсера | Rate limiting на ingress |

```python
import asyncio

class BackpressuredQueue:
    def __init__(self, maxsize: int = 1000):
        self.queue = asyncio.Queue(maxsize=maxsize)

    async def produce(self, item):
        # Заблокирует продюсера если очередь полная
        await self.queue.put(item)

    async def consume(self):
        return await self.queue.get()

# В Kafka:
# - max.block.ms — сколько ждать перед ошибкой при полном буфере
# - buffer.memory — размер буфера продюсера
# - Consumer lag — если lag растёт, consumer не справляется
```

---

## Transactional Outbox Pattern

Гарантирует атомарность записи в БД + публикации события (без 2PC).

```
┌────────────────────────────────────┐
│ BEGIN TRANSACTION                  │
│   INSERT INTO orders (...)         │
│   INSERT INTO outbox_events (      │  ← одна транзакция
│       type, payload, published=F)  │
│ COMMIT                             │
└────────────────────────────────────┘
            │
            ▼
   Outbox Poller / CDC
            │
            ▼
   Message Broker (Kafka/RabbitMQ)
```

```python
# Poller-based approach
async def outbox_publisher():
    while True:
        events = await db.fetch("""
            SELECT * FROM outbox_events
            WHERE published = FALSE
            ORDER BY created_at
            LIMIT 100
            FOR UPDATE SKIP LOCKED
        """)
        for event in events:
            await broker.publish(event["type"], event["payload"])
            await db.execute(
                "UPDATE outbox_events SET published = TRUE WHERE id = $1",
                event["id"]
            )
        await asyncio.sleep(1)  # poll interval
```

**CDC (Change Data Capture):** вместо polling'а — Debezium читает WAL PostgreSQL и публикует изменения в Kafka. Более эффективно, near-realtime.

---

## Idempotency (идемпотентность)

**Идемпотентная операция** — при повторном выполнении даёт тот же результат.

```python
# Проблема: сообщение обработано дважды → деньги списаны дважды
# Решение: idempotency key

async def process_payment(event: dict):
    idempotency_key = event["event_id"]  # уникальный ID события

    # Проверить: уже обрабатывали?
    existing = await db.fetchrow(
        "SELECT * FROM processed_events WHERE key = $1", idempotency_key
    )
    if existing:
        return existing["result"]  # вернуть сохранённый результат

    # Обработать
    result = await charge_card(event["amount"], event["card_id"])

    # Записать факт обработки
    await db.execute(
        "INSERT INTO processed_events (key, result) VALUES ($1, $2)",
        idempotency_key, result
    )
    return result
```

**Stripe-подход** (клиент передаёт idempotency key):
```http
POST /v1/charges
Idempotency-Key: req_abc123
Content-Type: application/json

{"amount": 1000, "currency": "usd", ...}
```

---

## Dead Letter Queue (DLQ)

Если сообщение не может быть обработано после N попыток → отправить в DLQ для ручного разбора.

```
Main Queue → Consumer → [ошибка] → retry (3x) → DLQ
                                                   │
                                           Мониторинг / alert
                                           Ручной разбор
                                           Replay в main queue
```

---

## Service Mesh и Sidecar

**Service Mesh** — инфраструктурный слой для коммуникации между сервисами (Istio, Linkerd).

```
┌─────────────────────┐   ┌─────────────────────┐
│ Service A           │   │ Service B           │
│  ┌───────────────┐  │   │  ┌───────────────┐  │
│  │  Application  │  │   │  │  Application  │  │
│  └───────┬───────┘  │   │  └───────▲───────┘  │
│          │          │   │          │          │
│  ┌───────▼───────┐  │   │  ┌───────┴───────┐  │
│  │ Sidecar Proxy │──────────│ Sidecar Proxy │  │
│  │ (Envoy)       │  │   │  │ (Envoy)       │  │
│  └───────────────┘  │   │  └───────────────┘  │
└─────────────────────┘   └─────────────────────┘
```

**Что делает sidecar proxy:**
- mTLS (шифрование между сервисами)
- Load balancing
- Circuit breaker, retry, timeout
- Observability (метрики, трейсинг)
- Rate limiting
- Canary deployments, traffic splitting

---

## Circuit Breaker

Предотвращает каскадный отказ при недоступности downstream-сервиса.

```
States:  CLOSED ──(failures > threshold)──► OPEN ──(timeout)──► HALF-OPEN
           ▲                                                        │
           └──────────(success)────────────────────────────────────┘
           └──────────(failure)────► OPEN
```

```python
import time
from enum import Enum

class State(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, recovery_timeout: int = 30):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.state = State.CLOSED
        self.failure_count = 0
        self.last_failure_time = 0

    async def call(self, func, *args, **kwargs):
        if self.state == State.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = State.HALF_OPEN
            else:
                raise CircuitOpenError("Service unavailable")

        try:
            result = await func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        self.failure_count = 0
        self.state = State.CLOSED

    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = State.OPEN


# Использование
payment_breaker = CircuitBreaker(failure_threshold=3, recovery_timeout=60)

async def charge_user(user_id: str, amount: float):
    try:
        return await payment_breaker.call(payment_service.charge, user_id, amount)
    except CircuitOpenError:
        # Fallback: поставить в очередь, уведомить, вернуть pending
        await queue.put({"user_id": user_id, "amount": amount})
        return {"status": "pending", "message": "Payment will be processed later"}
```

---

## Типичные вопросы

**Q: Sync vs Async — как выбрать?**
- Sync: UI ждёт ответ, валидация, авторизация, read queries
- Async: уведомления, аналитика, длинные обработки, inter-service events

**Q: Event Sourcing vs обычная CRUD-БД?**
- ES: нужен аудит, undo, replay, temporal queries, CQRS
- CRUD: простой домен, не нужна история, latency чтения критична

**Q: Как гарантировать ordering в event-driven?**
- Один partition/queue на entity (Kafka: partition key = entity_id)
- Sequence numbers на событиях + consumer валидирует порядок
- Single-writer pattern (один продюсер на entity)
