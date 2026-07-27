# RabbitMQ

## Архитектура

```
Publisher → Exchange → [Binding] → Queue → Consumer
```

**Ключевые компоненты:**

| Компонент | Роль |
|---|---|
| **Publisher** | Отправляет сообщения в Exchange |
| **Exchange** | Маршрутизирует сообщения в очереди по правилам |
| **Binding** | Связь между Exchange и Queue (с routing key) |
| **Queue** | Хранит сообщения до обработки |
| **Consumer** | Получает и обрабатывает сообщения |
| **Virtual Host** | Изоляция (как БД в PostgreSQL) |

---

## Типы Exchange

### Direct Exchange

Маршрутизация по точному совпадению routing key.

```
Publisher → Exchange(direct) → routing_key="error" → Queue "errors"
                             → routing_key="info"  → Queue "logs"
```

```python
channel.exchange_declare(exchange='logs', exchange_type='direct')
channel.queue_bind(queue='errors', exchange='logs', routing_key='error')
channel.queue_bind(queue='info',   exchange='logs', routing_key='info')

# Отправка
channel.basic_publish(exchange='logs', routing_key='error', body='DB connection failed')
```

### Fanout Exchange

Рассылает всем привязанным очередям (игнорирует routing key).

```
Publisher → Exchange(fanout) → Queue A (все получают)
                             → Queue B
                             → Queue C
```

```python
channel.exchange_declare(exchange='notifications', exchange_type='fanout')
# routing_key игнорируется
channel.basic_publish(exchange='notifications', routing_key='', body='System update')
```

### Topic Exchange

Маршрутизация по паттерну routing key. `*` — одно слово, `#` — ноль или больше.

```
routing_key: "user.created"    → привязка "user.*"   ✓, "*.created" ✓, "user.#" ✓
routing_key: "order.item.paid" → привязка "order.#"  ✓, "order.*"   ✗ (только одно слово)
```

```python
channel.exchange_declare(exchange='events', exchange_type='topic')
channel.queue_bind(queue='user_queue',  exchange='events', routing_key='user.*')
channel.queue_bind(queue='audit_queue', exchange='events', routing_key='#')  # всё
```

### Headers Exchange

Маршрутизация по заголовкам сообщения (редко используется).

---

## Default Exchange

Прямой обмен без имени. Routing key = имя очереди. Используется для простых случаев.

```python
# Отправка напрямую в очередь
channel.basic_publish(
    exchange='',           # default exchange
    routing_key='my_queue',
    body='hello'
)
```

---

## Работа с очередями

```python
import pika

connection = pika.BlockingConnection(pika.URLParameters('amqp://guest:guest@localhost/'))
channel = connection.channel()

# Объявление очереди (idempotent — создаст или проверит существующую)
channel.queue_declare(
    queue='tasks',
    durable=True,           # выживает при рестарте брокера
    arguments={
        'x-message-ttl': 3600000,      # TTL сообщений — 1 час
        'x-max-length': 10000,          # макс. сообщений
        'x-dead-letter-exchange': 'dlx' # DLQ exchange
    }
)
```

---

## Producer

```python
# Persistent сообщение — выживает при рестарте брокера
channel.basic_publish(
    exchange='',
    routing_key='tasks',
    body=json.dumps({"task_id": 123, "type": "send_email"}),
    properties=pika.BasicProperties(
        delivery_mode=pika.DeliveryMode.Persistent,  # 2
        content_type='application/json',
        message_id=str(uuid.uuid4()),   # для идемпотентности
        timestamp=int(time.time()),
        expiration='3600000'            # TTL сообщения (мс)
    )
)
```

---

## Consumer

```python
def callback(ch, method, properties, body):
    try:
        data = json.loads(body)
        process_task(data)
        ch.basic_ack(delivery_tag=method.delivery_tag)   # успешно
    except Exception as e:
        print(f"Error: {e}")
        ch.basic_nack(
            delivery_tag=method.delivery_tag,
            requeue=False   # False → в DLQ; True → вернуть в очередь
        )

# Prefetch: сколько сообщений брать за раз (1 = fair dispatch)
channel.basic_qos(prefetch_count=1)
channel.basic_consume(queue='tasks', on_message_callback=callback)
channel.start_consuming()
```

**`prefetch_count`:**
- `1` — честное распределение: Consumer берёт следующее только после ACK текущего
- `N` — буферизация N сообщений в памяти Consumer (выше throughput)

---

## Dead Letter Queue (DLQ)

```python
# Настройка DLQ
channel.exchange_declare(exchange='dlx', exchange_type='direct')
channel.queue_declare(queue='tasks.dlq', durable=True)
channel.queue_bind(queue='tasks.dlq', exchange='dlx', routing_key='tasks')

# Основная очередь — при nack/TTL → dlx
channel.queue_declare(
    queue='tasks',
    durable=True,
    arguments={
        'x-dead-letter-exchange': 'dlx',
        'x-dead-letter-routing-key': 'tasks',
        'x-message-ttl': 60000
    }
)

# Обработка DLQ (мониторинг / ручной разбор)
def dlq_callback(ch, method, properties, body):
    print(f"Dead letter: {body}, headers: {properties.headers}")
    # Логируем, алертим, пробуем повторно или отбрасываем
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(queue='tasks.dlq', on_message_callback=dlq_callback)
```

---

## Retry с экспоненциальным backoff

```python
MAX_RETRIES = 3

def callback(ch, method, properties, body):
    headers = properties.headers or {}
    retry_count = headers.get('x-retry-count', 0)

    try:
        process(json.loads(body))
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception as e:
        if retry_count < MAX_RETRIES:
            delay = (2 ** retry_count) * 1000   # 1s, 2s, 4s
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
            # Переотправить в delayed queue (или через x-delayed-message plugin)
            channel.basic_publish(
                exchange='',
                routing_key='tasks',
                body=body,
                properties=pika.BasicProperties(
                    headers={'x-retry-count': retry_count + 1},
                    expiration=str(delay)
                )
            )
        else:
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)  # → DLQ
```

---

## RPC поверх RabbitMQ

```python
# Client
result_queue = channel.queue_declare(queue='', exclusive=True)
correlation_id = str(uuid.uuid4())

channel.basic_publish(
    exchange='',
    routing_key='rpc_queue',
    body=json.dumps({"method": "add", "args": [2, 3]}),
    properties=pika.BasicProperties(
        reply_to=result_queue.method.queue,   # куда ответить
        correlation_id=correlation_id
    )
)

# Server
def on_request(ch, method, properties, body):
    result = process(json.loads(body))
    ch.basic_publish(
        exchange='',
        routing_key=properties.reply_to,
        body=json.dumps(result),
        properties=pika.BasicProperties(
            correlation_id=properties.correlation_id
        )
    )
    ch.basic_ack(delivery_tag=method.delivery_tag)
```

---

## Полезные команды

```bash
# Management UI (порт 15672)
rabbitmq-plugins enable rabbitmq_management

# CLI
rabbitmqctl list_queues name messages consumers
rabbitmqctl list_exchanges
rabbitmqctl purge_queue <queue_name>
rabbitmqctl delete_queue <queue_name>

# Просмотр сообщений без удаления
rabbitmqctl peek_messages <queue_name> 10
```

---

## Мониторинг

**Ключевые метрики:**
- `messages_ready` — сообщений ждут обработки
- `messages_unacknowledged` — взяты Consumer, но не подтверждены
- `message_stats.publish_rate` — скорость публикации
- `message_stats.deliver_rate` — скорость доставки
- `consumers` — число активных Consumer-ов

```python
# Prometheus + rabbitmq_prometheus plugin
# Метрика: rabbitmq_queue_messages{queue="tasks"}
# Алерт: queue_depth > 10000 → масштабировать Consumer
```
