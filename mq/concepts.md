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

---

## Celery

### Что такое Celery и как она устроена

Celery — это не брокер. Это **Python-библиотека**, которая реализует task queue поверх существующего брокера (RabbitMQ, Redis, и др.).

```
[Django/FastAPI] → celery.delay(task) → [Broker: RabbitMQ/Redis] → [Celery Worker] → [Result Backend: Redis/DB]
```

Что Celery добавляет поверх голого брокера:
- **Retry** с backoff — `autoretry_for`, `max_retries`, `retry_backoff`
- **Scheduling** (celery beat) — аналог cron, но управляемый из кода
- **Rate limiting** — не более N задач в секунду на воркер
- **Canvas** — цепочки (`chain`), параллельные группы (`group`), аккордеоны (`chord`)
- **Result backend** — хранение статуса и результата задачи
- **Routing** — разные очереди для разных типов задач

---

### Какие проблемы есть у Celery

**1. Дублирование задач (at-least-once).**
По умолчанию воркер подтверждает (ACK) сообщение *до* выполнения задачи. Если воркер упал в середине — задача не выполнена, но ACK уже отправлен → потеря. Если включить `acks_late=True` — ACK после выполнения, но тогда при падении задача запустится повторно → дубликат. Идемпотентность задач обязательна.

**2. Prefetch — воркер хватает больше задач, чем может обработать.**
По умолчанию `worker_prefetch_multiplier=4`: воркер резервирует 4 задачи из очереди. Если задачи тяжёлые и воркер упал — эти задачи зависли. Решение: `worker_prefetch_multiplier=1`.

**3. Утечки памяти в долгоживущих воркерах.**
Задачи накапливают состояние, Python не всегда освобождает память. Стандартное решение — `worker_max_tasks_per_child=N`: воркер перезапускается после N задач.

**4. Celery Beat — единая точка отказа.**
Планировщик запускается как один процесс. Если упал — все cron-задачи не выполняются. Решение: `django-celery-beat` с хранением расписания в БД + внешний keep-alive.

**5. Сложность отладки.**
Исключения в воркере не видны в основном приложении. Нужен отдельный мониторинг (Flower, Sentry для Celery).

**6. Нет replay.**
Задачи не хранятся в брокере после выполнения — нельзя перезапустить историю. В отличие от Kafka.

---

### Почему нельзя просто отправить задачу напрямую в Kafka или RabbitMQ

Технически — **можно**, и это иногда правильное решение. Но нужно понимать, что теряешь и что приобретаешь.

**Что нужно реализовать самостоятельно без Celery:**

| Фича | В Celery | В голом Kafka/RabbitMQ |
|---|---|---|
| Retry с backoff | `autoretry_for`, встроено | сам пиши логику |
| Статус задачи | Result backend | сам храни в БД/Redis |
| Cron / scheduling | Celery Beat | отдельный сервис |
| Rate limiting | `rate_limit` на задачу | сам реализуй |
| Отмена задачи | `task.revoke()` | нет стандартного способа |
| Мониторинг | Flower | сам или внешнее |

**Kafka плохо подходит для классической task queue:**
- Kafka удаляет сообщения по offset'у, а не по факту обработки. Если задача 5 упала — нельзя перезапустить только её, не затронув 6, 7, 8.
- Kafka оптимизирован под высокий throughput потока событий, не под единичные задачи с retry/state.
- Партиции в Kafka — для параллелизма по ключу, не для пула воркеров. Добавить воркеров сложнее, чем в Celery.

**RabbitMQ — лучший выбор для task queue**, и Celery как раз работает поверх него. Напрямую через RabbitMQ имеет смысл, если:
- Нужны не-Python консьюмеры (Go, Java)
- Нет нужды в retry/scheduling/result storage
- Хочешь минимальный overhead

**Когда использовать Celery:**
- Python-приложение, нужны фоновые задачи с retry и статусом
- Периодические задачи (замена cron)
- Обработка очереди задач с rate limiting

**Когда использовать Kafka напрямую:**
- Нужен audit log / replay событий
- Высокий throughput (миллионы событий в день)
- Несколько разных сервисов подписываются на один поток событий
- Нужна история — кто и когда что сделал

**Когда использовать RabbitMQ напрямую (без Celery):**
- Разные языки на стороне консьюмера
- Сложный routing по exchange/binding
- Нет нужды в Celery-специфичных фичах
