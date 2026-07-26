# Архитектура — Наблюдаемость (Observability)

## Три столпа

| Столп | Что даёт | Инструменты |
|---|---|---|
| **Логи** | Что произошло (события) | ELK, Loki + Grafana, CloudWatch |
| **Метрики** | Сколько/насколько (числа во времени) | Prometheus + Grafana, Datadog |
| **Трейсинг** | Как выполнился запрос через сервисы | Jaeger, Zipkin, OpenTelemetry |

---

## Логирование

### Уровни

| Уровень | Когда |
|---|---|
| DEBUG | Детальная отладка |
| INFO | Обычные события |
| WARNING | Нестандартные ситуации |
| ERROR | Ошибки, не прерывающие работу |
| CRITICAL | Критические сбои |

### Структурированное логирование

Вместо строк — JSON. Легко парсить, фильтровать, строить метрики из логов.

```python
import structlog

log = structlog.get_logger()

# Плохо: строка → сложно парсить
logger.info(f"Order {order_id} created for user {user_id}, amount={amount}")

# Хорошо: структурированный лог
log.info("order.created", order_id=order_id, user_id=user_id, amount=amount)
# → {"event": "order.created", "order_id": 123, "user_id": 456, "amount": "99.99", "timestamp": ...}
```

### Контекст логирования

Привязать к каждому логу контекст запроса (request_id, user_id, trace_id) **автоматически**, без передачи вручную через аргументы.

```python
from contextvars import ContextVar
from uuid import uuid4
import structlog

# Переменная хранится в контексте текущей корутины
request_id_var: ContextVar[str] = ContextVar('request_id')

# Middleware: привязать request_id к каждому запросу
@app.middleware("http")
async def request_context_middleware(request: Request, call_next):
    request_id = str(uuid4())
    request_id_var.set(request_id)
    structlog.contextvars.bind_contextvars(
        request_id=request_id,
        path=request.url.path,
        method=request.method,
    )
    response = await call_next(request)
    structlog.contextvars.clear_contextvars()
    return response

# В любом месте кода — request_id попадает в лог автоматически
log.info("user.fetched", user_id=42)
# → {"event": "user.fetched", "user_id": 42, "request_id": "abc-123", "path": "/users/42"}
```

---

## Метрики

### Типы метрик (Prometheus)

| Тип | Описание | Пример |
|---|---|---|
| Counter | Только растёт | Число запросов, ошибок |
| Gauge | Текущее значение (растёт/падает) | RAM usage, активные соединения |
| Histogram | Распределение + percentiles | Latency запросов |
| Summary | Предвычисленные percentiles | Latency (клиентская сторона) |

```python
from prometheus_client import Counter, Histogram, start_http_server

REQUEST_COUNT = Counter(
    'http_requests_total', 'Total requests',
    ['method', 'endpoint', 'status_code']
)
REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds', 'Request latency',
    buckets=[0.01, 0.05, 0.1, 0.5, 1.0, 5.0]
)

@app.middleware("http")
async def metrics_middleware(request, call_next):
    with REQUEST_LATENCY.time():
        response = await call_next(request)
    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.url.path,
        status_code=response.status_code
    ).inc()
    return response
```

### Time-series метрики

Метрики, привязанные к времени. Хранятся в БД оптимизированных для временных рядов: Prometheus, InfluxDB, TimescaleDB, VictoriaMetrics.

### Алертинг

Автоматические уведомления при выходе метрик за пороги (Slack, PagerDuty, email).

```yaml
# Prometheus AlertManager
groups:
  - name: api_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status_code=~"5.."}[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Error rate > 5% for 2 minutes"

      - alert: HighLatency
        expr: histogram_quantile(0.99, http_request_duration_seconds_bucket) > 1.0
        for: 5m
        annotations:
          summary: "p99 latency > 1s"
```

---

## Distributed Tracing

Отслеживание пути запроса через несколько сервисов.

**Концепции:**
- **Trace** — полный путь запроса (дерево spans)
- **Span** — единица работы (вызов сервиса, SQL-запрос, HTTP-запрос)
- **Trace ID** — передаётся через все сервисы (в HTTP-заголовке)

```
HTTP запрос: → API Gateway → Order Service → Payment Service
                                           → Inventory Service

Trace [trace_id=abc-123]:
├── api-gateway        0ms  → 250ms
├── order-service      5ms  → 240ms
│   ├── db: SELECT     5ms  → 20ms
│   ├── payment-svc   25ms  → 180ms
│   │   └── bank-api  30ms  → 160ms
│   └── inventory-svc 30ms  → 60ms
```

**OpenTelemetry** — стандарт сбора traces/metrics/logs:

```python
from opentelemetry import trace
from opentelemetry.exporter.jaeger.thrift import JaegerExporter

tracer = trace.get_tracer(__name__)

@app.get("/orders/{order_id}")
async def get_order(order_id: int):
    with tracer.start_as_current_span("get_order") as span:
        span.set_attribute("order.id", order_id)
        order = await order_repo.get(order_id)
        span.set_attribute("order.status", order.status)
        return order
```

---

## ELK Stack

| Компонент | Роль |
|---|---|
| **Elasticsearch** | Поисковый движок — индексирует и хранит логи |
| **Logstash** | Сбор, фильтрация, трансформация данных из источников |
| **Kibana** | Веб-интерфейс для поиска и визуализации |

```
Приложения → Filebeat → Logstash → Elasticsearch → Kibana
                (или напрямую)
```

**Современная альтернатива:** Grafana + Loki (логи) + Prometheus (метрики) + Tempo (трейсинг). Проще в эксплуатации, дешевле.
