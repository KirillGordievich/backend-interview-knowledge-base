# Распределённые системы

## CAP теорема

В распределённой системе можно гарантировать **только 2 из 3** свойств:

| Свойство | Описание |
|---|---|
| **C** — Consistency (согласованность) | Каждое чтение возвращает самую актуальную запись |
| **A** — Availability (доступность) | Каждый узел всегда успешно отвечает на запросы |
| **P** — Partition tolerance (устойчивость к разделению) | Система продолжает работать при потере связи между узлами |

**P обязателен** в любой реальной сети — разрывы неизбежны. Поэтому выбор стоит между **CP** и **AP**.

```
CP — согласованность + устойчивость к разделению:
  При разрыве: узлы отвечают ошибкой (недоступны), но данные согласованы.
  Примеры: HBase, Zookeeper, etcd, MongoDB (при w=majority)

AP — доступность + устойчивость к разделению:
  При разрыве: узлы отвечают (возможно устаревшими данными).
  Примеры: Cassandra, DynamoDB, CouchDB, DNS
```

**CA** — без устойчивости к разделению: возможно только в одноузловых системах (PostgreSQL, MySQL).

### PACELC — расширение CAP

CAP описывает поведение при разрыве. PACELC добавляет случай нормальной работы:

```
If Partition → (A или C)
Else (нормально) → (Latency или Consistency)
```

При нормальной работе тоже приходится выбирать: отвечать быстро (low latency) или дождаться синхронизации реплик (strong consistency).

---

## BASE vs ACID

| | ACID (OLTP БД) | BASE (распределённые системы) |
|---|---|---|
| **Расшифровка** | Atomicity, Consistency, Isolation, Durability | Basically Available, Soft state, Eventually consistent |
| **Консистентность** | Строгая, сразу | Eventual (в конечном итоге) |
| **Доступность** | Может блокировать | Всегда доступна |
| **Когда** | Финансы, транзакции | Лента соцсети, счётчики лайков |

---

## Распределённые транзакции

Проблема: ACID работает в рамках одной БД. В микросервисах у каждого сервиса — своя БД.

### Two-Phase Commit (2PC)

Протокол атомарной фиксации через координатора.

```
Фаза 1 — Prepare:
  Coordinator → PREPARE → Service A (locks resources)
  Coordinator → PREPARE → Service B (locks resources)
  Coordinator ← YES ← Service A
  Coordinator ← YES ← Service B

Фаза 2 — Commit (если все YES):
  Coordinator → COMMIT → Service A
  Coordinator → COMMIT → Service B

Фаза 2 — Rollback (если хоть один NO):
  Coordinator → ROLLBACK → Service A
  Coordinator → ROLLBACK → Service B
```

**Проблемы 2PC:**
- **Blocking protocol** — если координатор упал после PREPARE, участники заблокированы навсегда
- **SPOF** — координатор — единая точка отказа
- Низкая производительность — 2 round-trip + блокировки ресурсов
- Плохо масштабируется при большом числе участников

**Когда применять:** небольшое число участников, критически важная согласованность (банковский перевод между двумя БД в рамках одного монолита).

---

### Saga Pattern

Транзакция разбивается на цепочку **локальных транзакций**. При сбое — запускаются **компенсирующие транзакции** (rollback бизнес-логикой, а не СУБД).

```
Успешный сценарий:
  1. OrderService:     CreateOrder     → OK  → publishes OrderCreated
  2. InventoryService: ReserveStock    → OK  → publishes StockReserved
  3. PaymentService:   ChargePayment   → OK  → publishes PaymentCharged
  4. OrderService:     ConfirmOrder    → OK

Сценарий с ошибкой на шаге 3:
  1. CreateOrder     → OK
  2. ReserveStock    → OK
  3. ChargePayment   → FAIL
  ← компенсация:
  2. ReleaseStock    (компенсирует ReserveStock)
  1. CancelOrder     (компенсирует CreateOrder)
```

#### Choreography (Хореография)

Без оркестратора. Каждый сервис слушает события и реагирует.

```
OrderService → [OrderCreated] → InventoryService
InventoryService → [StockReserved] → PaymentService
PaymentService → [PaymentFailed] → InventoryService → [StockReleased] → OrderService
```

**Плюсы:** нет SPOF, слабая связность.
**Минусы:** сложно отследить общий поток, бизнес-логика размазана, трудно дебажить.

#### Orchestration (Оркестрация)

Центральный оркестратор явно управляет шагами и компенсациями.

```python
class OrderSaga:
    async def execute(self, order_data: dict):
        order_id = await self.order_service.create_order(order_data)
        try:
            await self.inventory_service.reserve_stock(order_id)
            try:
                await self.payment_service.charge(order_id)
                await self.order_service.confirm(order_id)
            except PaymentError:
                await self.inventory_service.release_stock(order_id)   # компенсация
                await self.order_service.cancel(order_id)              # компенсация
                raise
        except InventoryError:
            await self.order_service.cancel(order_id)                  # компенсация
            raise
```

**Плюсы:** весь поток виден в одном месте, легко дебажить.
**Минусы:** оркестратор — SPOF, риск превращения в "умный монолит".

#### 2PC vs Saga

| | 2PC | Saga |
|---|---|---|
| Консистентность | Строгая (ACID) | Eventual consistency |
| Блокировки | Да (на всё время) | Нет |
| Производительность | Низкая | Высокая |
| Сложность | Протокол | Компенсирующие транзакции |
| SPOF | Координатор | Оркестратор (в orchestration) |
| Когда | Монолит, несколько БД | Микросервисы |

---

## Outbox Pattern

Решает проблему: как атомарно сохранить данные в БД **и** опубликовать событие в MQ.

**Проблема:**
```python
# Не атомарно! Если упадём между двумя операциями — рассинхрон
await db.save(order)
await kafka.publish(OrderCreated(order.id))   # может не выполниться
```

**Решение — Transactional Outbox:**
```python
async with db.transaction():
    await db.save(order)
    await db.save(OutboxEvent(
        type="OrderCreated",
        payload=order.to_dict(),
        status="pending"
    ))
# Outbox Processor (отдельный воркер) читает pending события → публикует → помечает sent
```

```sql
CREATE TABLE outbox_events (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type  VARCHAR(100) NOT NULL,
    payload     JSONB NOT NULL,
    status      VARCHAR(20) DEFAULT 'pending',
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    sent_at     TIMESTAMPTZ
);
```

**Гарантия:** или оба (запись + событие в outbox) попадают в транзакцию, или ни один.

---

## Eventual Consistency

Система гарантирует, что **в конечном счёте** все реплики придут к одному состоянию — без гарантии когда именно.

**Примеры:**
- DNS propagation (обновление записи видно через минуты)
- Лента соцсети (разные пользователи видят разное на протяжении секунд)
- Корзина в Amazon (счётчик товаров может временно расходиться)

**Техники работы с eventual consistency:**
- Идемпотентные операции — повторная обработка не меняет результат
- Version vectors / CRDTs — автоматическое разрешение конфликтов
- Read-your-writes — пользователь видит свои изменения сразу (sticky sessions, read own writes)

---

## Идемпотентность в распределённых системах

Операция идемпотентна, если повторное выполнение даёт тот же результат.

```python
# НЕ идемпотентно — каждый раз создаёт новую запись
async def create_order(data):
    return await db.insert("orders", data)

# Идемпотентно — по idempotency_key
async def create_order(data, idempotency_key: str):
    existing = await db.get("orders", {"idempotency_key": idempotency_key})
    if existing:
        return existing   # вернуть уже созданный
    return await db.insert("orders", {**data, "idempotency_key": idempotency_key})
```

**Idempotency key:** клиент генерирует UUID, передаёт в заголовке (`Idempotency-Key: uuid`). Сервер кэширует ответ по ключу на время TTL.
