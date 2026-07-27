# MQ — Концепции и основы

## Зачем нужны очереди сообщений

**Без очереди:** Producer напрямую вызывает Consumer → жёсткая связность, синхронность, нет буфера.

**С очередью:**
```
Producer → [Queue/Topic] → Consumer(s)
```

| Проблема | Решение через MQ |
|---|---|
| Жёсткая связность | Producer не знает о Consumer, они независимы |
| Пиковые нагрузки | Очередь буферизует; Consumer обрабатывает по возможности |
| Надёжность | Сообщение не потеряется, если Consumer временно упал |
| Масштабируемость | Добавить Consumer-воркеров, не меняя Producer |
| Асинхронность | Producer не ждёт обработки |

---

## Базовые паттерны

### Point-to-Point (Queue)

Одно сообщение обрабатывается **одним** Consumer. Несколько воркеров конкурируют за сообщения.

```
Producer → [Queue] → Consumer A
                  → Consumer B  (один из них получит сообщение)
```

Использование: распределение задач (task queue), обработка заказов.

### Publish/Subscribe (Topic)

Одно сообщение получают **все** подписчики независимо.

```
Producer → [Topic] → Consumer A (получит)
                   → Consumer B (получит)
                   → Consumer C (получит)
```

Использование: уведомления, инвалидация кэша, event streaming.

### Consumer Groups (Kafka-стиль)

Комбинация: внутри группы — point-to-point; разные группы — pub/sub.

```
Topic → Group "analytics":  [Consumer A | Consumer B]  (делят партиции)
      → Group "reporting":  [Consumer C]                (получает всё)
```

---

## Гарантии доставки

| Гарантия | Описание | Дубликаты | Потери |
|---|---|---|---|
| **At-most-once** | Сообщение доставляется 0 или 1 раз | Нет | Возможны |
| **At-least-once** | Сообщение доставляется 1+ раз | Возможны | Нет |
| **Exactly-once** | Ровно один раз | Нет | Нет |

**На практике:** at-least-once + идемпотентный Consumer = exactly-once семантика.

---

## Acknowledgment (ACK)

Consumer подтверждает обработку сообщения. Без ACK брокер считает сообщение необработанным и переотправит.

```
Consumer получает сообщение
    → обрабатывает
    → отправляет ACK
    → брокер удаляет сообщение из очереди

Если Consumer упал до ACK:
    → брокер переотправит другому Consumer
```

**Виды ACK в RabbitMQ:**
- `basic_ack` — успешно обработано
- `basic_nack` — ошибка, переотправить (`requeue=True`) или отбросить
- `basic_reject` — отбросить одно сообщение

---

## Dead Letter Queue (DLQ)

Специальная очередь для сообщений, которые не удалось обработать.

**Сообщение попадает в DLQ когда:**
- Consumer отклонил (`nack/reject`) с `requeue=False`
- Истёк TTL сообщения
- Очередь переполнена (max-length)
- Превышено число попыток переотправки

```
Main Queue → [processing fails] → Dead Letter Queue → alert / manual review
```

Зачем: не потерять сообщения при ошибках, иметь возможность разобраться и переотправить.

---

## Идемпотентность Consumer

Consumer должен корректно обрабатывать **дубликаты** (при at-least-once).

**Способы:**
```python
# 1. Уникальный message_id в БД
def process(message):
    if db.exists("processed_messages", message.id):
        return   # уже обработано
    db.insert("processed_messages", message.id)
    do_work(message)

# 2. Upsert вместо Insert
db.execute("""
    INSERT INTO orders (id, status) VALUES (:id, :status)
    ON CONFLICT (id) DO NOTHING
""", {"id": order_id, "status": "created"})

# 3. Версионирование (optimistic locking)
db.execute("""
    UPDATE accounts SET balance = :new_balance, version = :new_version
    WHERE id = :id AND version = :expected_version
""", ...)
```

---

## Backpressure

Ситуация когда Consumer не успевает обрабатывать входящий поток.

**Симптомы:** растущая очередь, увеличение lag (отставания).

**Решения:**
- Масштабировать Consumer-ов горизонтально
- Ограничить prefetch (сколько сообщений Consumer берёт за раз)
- Rate limiting на Producer
- Оповещения по метрике queue depth

---

## Ordering (Порядок сообщений)

**Гарантии порядка:**
- **RabbitMQ:** FIFO в рамках одной очереди при одном Consumer
- **Kafka:** FIFO в рамках одной партиции

**Проблема при масштабировании:** несколько Consumer-ов = нет гарантии порядка.

**Решение:** все сообщения для одного entity (user_id, order_id) → одна партиция/очередь.

```python
# Kafka: routing key = user_id → одна партиция для одного пользователя
producer.send(
    topic="user-events",
    key=str(user_id).encode(),   # одинаковый key → одна партиция
    value=event_json.encode()
)
```

---

## TTL (Time-To-Live)

Время жизни сообщения. По истечении — удаляется или переходит в DLQ.

```python
# RabbitMQ — TTL на сообщение
channel.basic_publish(
    exchange='',
    routing_key='queue',
    body='message',
    properties=pika.BasicProperties(
        expiration='60000'  # 60 секунд в миллисекундах
    )
)

# RabbitMQ — TTL на очередь (все сообщения)
channel.queue_declare(
    queue='short-lived',
    arguments={'x-message-ttl': 60000}
)
```

---

## Популярные реализации

| Система | Модель | Когда |
|---|---|---|
| **RabbitMQ** | Smart broker, dumb consumer | Task queues, сложный routing, RPC |
| **Kafka** | Dumb broker, smart consumer | Event streaming, высокий throughput, replay |
| **Redis Streams** | Лёгковесный log | Уже используешь Redis, простые случаи |
| **Celery** | Task queue поверх RabbitMQ/Redis | Python: cron, retry, rate limiting |
| **AWS SQS** | Managed queue | Облако, serverless |
| **AWS SNS** | Managed pub/sub | Фанаут в облаке |
