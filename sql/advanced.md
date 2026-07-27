# SQL — OLAP, OLTP и распределённые транзакции

## OLTP vs OLAP

| | OLTP (Online Transaction Processing) | OLAP (Online Analytical Processing) |
|---|---|---|
| Назначение | Операционные транзакции | Аналитика и отчёты |
| Операции | INSERT, UPDATE, DELETE, SELECT (по ключу) | SELECT с большими агрегациями |
| Объём данных в запросе | Мало строк | Сотни миллионов строк |
| Нормализация | Высокая (3NF) | Низкая (Star/Snowflake schema) |
| Типичные СУБД | PostgreSQL, MySQL | ClickHouse, BigQuery, Redshift, DuckDB |
| Индексы | B-tree, по первичным ключам | Колоночное хранение, bitmap |
| Конкурентность | Много параллельных транзакций | Мало долгих аналитических запросов |

**Star Schema** (для OLAP):
```
              dim_time
                 │
dim_product ─ fact_sales ─ dim_customer
                 │
            dim_location
```

Центральная таблица фактов (fact) + окружающие таблицы измерений (dim). Денормализована — быстрые JOIN-ы, медленные обновления.

---

## Гонки (Race Conditions) и как их избегать

Возникают когда несколько транзакций читают и изменяют одни данные параллельно.

**Пример — Lost Update:**
```sql
-- Транзакция A и B одновременно:
SELECT balance FROM accounts WHERE id = 1;  -- оба читают 100
-- A: UPDATE ... SET balance = 100 - 30     → 70
-- B: UPDATE ... SET balance = 100 - 50     → 50
-- Результат: 50 вместо 20 (потеряно списание A)
```

**Решение:**
```sql
-- Пессимистичная блокировка
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- блокируем строку
UPDATE accounts SET balance = balance - 30 WHERE id = 1;
COMMIT;

-- Оптимистичная блокировка (version/timestamp)
UPDATE accounts
SET balance = balance - 30, version = version + 1
WHERE id = 1 AND version = 5;  -- если version изменилась — 0 строк → повторить
```

---

## Распределённые транзакции

Транзакция, охватывающая несколько баз данных или сервисов.

**Проблема:** ACID работает в рамках одной БД. Как гарантировать атомарность через несколько систем?

### Two-Phase Commit (2PC, Двухфазная фиксация)

Протокол координации распределённой транзакции.

**Участники:** Coordinator (координатор) + Participants (участники)

**Фаза 1 — Prepare:**
```
Coordinator → PREPARE → Participant A
Coordinator → PREPARE → Participant B
Coordinator ← YES     ← Participant A  (заблокировал ресурсы, готов)
Coordinator ← YES     ← Participant B
```

**Фаза 2 — Commit/Rollback:**
```
Если все YES:
Coordinator → COMMIT → Participant A
Coordinator → COMMIT → Participant B

Если хоть один NO:
Coordinator → ROLLBACK → Participant A
Coordinator → ROLLBACK → Participant B
```

**Проблемы 2PC:**
- **Blocking protocol** — если координатор упал после PREPARE, участники заблокированы навсегда
- **Single point of failure** — координатор узкое место
- Низкая производительность — две сетевых round-trip + ожидание

### Saga Pattern (альтернатива 2PC)

Транзакция разбивается на цепочку локальных транзакций с **компенсирующими** действиями при откате.

```
Orchestrator-сага:

  1. CreateOrder         → OK    (orders DB)
  2. ReserveInventory    → OK    (inventory DB)
  3. ChargePayment       → FAIL  (payment service)

  Компенсация (в обратном порядке):
  2. ReleaseInventory    → компенсирует ReserveInventory
  1. CancelOrder         → компенсирует CreateOrder
```

**Choreography** (без оркестратора): каждый сервис слушает события и реагирует.

| | 2PC | Saga |
|---|---|---|
| Консистентность | Строгая (ACID) | Eventual consistency |
| Блокировки | Да | Нет |
| Сложность | Протокол | Компенсирующие транзакции |
| Производительность | Низкая | Высокая |
| Когда | Монолит, несколько БД | Микросервисы |
