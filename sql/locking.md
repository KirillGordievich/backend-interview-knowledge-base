# SQL — Блокировки и дедлоки

## Зачем нужны блокировки

Блокировки защищают данные от конкурентных изменений при параллельных транзакциях. Без блокировок возможны гонки, потерянные обновления и нарушения целостности.

---

## Уровни блокировок

### Row-level (строчный уровень)

Блокирует только конкретные строки — минимальное влияние на конкурентность. PostgreSQL использует по умолчанию.

### Table-level (уровень таблицы)

Блокирует всю таблицу. Автоматически применяется при DDL-операциях (ALTER TABLE, DROP).

```sql
-- Явная блокировка таблицы
LOCK TABLE orders IN ACCESS EXCLUSIVE MODE;
```

**Табличные блокировки PostgreSQL (от слабой к строгой):**

| Режим | Кто берёт | Конфликтует с |
|---|---|---|
| ACCESS SHARE | `SELECT` | ACCESS EXCLUSIVE |
| ROW SHARE | `SELECT FOR UPDATE/SHARE` | EXCLUSIVE, ACCESS EXCLUSIVE |
| ROW EXCLUSIVE | `INSERT`, `UPDATE`, `DELETE` | SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE |
| SHARE UPDATE EXCLUSIVE | `VACUUM`, `CREATE INDEX CONCURRENTLY` | SHARE UPDATE EXCLUSIVE и выше |
| SHARE | `CREATE INDEX` (не CONCURRENTLY) | ROW EXCLUSIVE и выше |
| SHARE ROW EXCLUSIVE | — (редко вручную) | ROW EXCLUSIVE и выше |
| EXCLUSIVE | — (редко вручную) | ROW SHARE и выше |
| ACCESS EXCLUSIVE | `ALTER TABLE`, `DROP`, `TRUNCATE`, `VACUUM FULL` | Все (полная эксклюзивная блокировка) |

Ключевое: `SELECT` берёт ACCESS SHARE → не конфликтует ни с чем, кроме ACCESS EXCLUSIVE (ALTER TABLE, DROP). Поэтому `SELECT` не блокирует `INSERT/UPDATE/DELETE` и наоборот.

### Advisory locks (рекомендательные блокировки)

Пользовательские блокировки, не привязанные к таблицам/строкам. Полезны для координации на уровне приложения.

```sql
-- Получить блокировку (блокирующий вызов)
SELECT pg_advisory_lock(12345);
-- ... критическая секция ...
SELECT pg_advisory_unlock(12345);

-- Неблокирующая попытка
SELECT pg_try_advisory_lock(12345);  -- true/false
```

Применение: запуск миграций (один процесс за раз), синглтон-задачи (cron без дублирования), распределённые блокировки.

---

## Блокировки строк (row-level)

### Режимы строчных блокировок (PostgreSQL)

| Режим | SQL | Конфликтует с |
|---|---|---|
| FOR KEY SHARE | FK-проверка | FOR UPDATE |
| FOR SHARE | `SELECT ... FOR SHARE` | FOR UPDATE, FOR NO KEY UPDATE |
| FOR NO KEY UPDATE | `UPDATE` (не меняет PK/UK) | FOR SHARE, FOR UPDATE, FOR NO KEY UPDATE |
| FOR KEY UPDATE | `UPDATE` (меняет PK/UK), `DELETE` | Все |

`FOR SHARE` — несколько транзакций могут читать, но не изменять строку.
`FOR UPDATE` — эксклюзивная блокировка, блокирует все остальные.

### SELECT FOR UPDATE

Блокирует строки для обновления — другие транзакции не смогут их изменить или заблокировать.

```sql
BEGIN;
SELECT * FROM orders WHERE id = 42 FOR UPDATE;
-- строка заблокирована до COMMIT/ROLLBACK
UPDATE orders SET status = 'processing' WHERE id = 42;
COMMIT;
```

### NOWAIT

Не ждать снятия блокировки — сразу вернуть ошибку если строка занята.

```sql
SELECT * FROM orders WHERE id = 42 FOR UPDATE NOWAIT;
-- ERROR: could not obtain lock on row in relation "orders"
```

### SKIP LOCKED

Пропустить заблокированные строки — удобно для реализации очередей задач.

```sql
-- Воркер берёт следующую незанятую задачу
SELECT * FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

---

## Дедлок (Deadlock)

Ситуация когда две (или более) транзакции взаимно ждут блокировки друг друга.

```
Транзакция A:  LOCK строка 1 → ждёт строку 2
Транзакция B:  LOCK строка 2 → ждёт строку 1
                → ни одна не может продолжить
```

PostgreSQL автоматически обнаруживает дедлоки и завершает одну из транзакций с ошибкой:
```
ERROR: deadlock detected
DETAIL: Process 123 waits for ShareLock on transaction 456
```

### Как избегать дедлоков

1. **Фиксированный порядок блокировки** — всегда блокировать ресурсы в одном и том же порядке:
   ```python
   # Плохо: A блокирует (1, 2), B блокирует (2, 1) → дедлок
   # Хорошо: оба блокируют в порядке возрастания id
   ids = sorted([id1, id2])
   SELECT * FROM accounts WHERE id = ANY($1) ORDER BY id FOR UPDATE;
   ```

2. **Короткие транзакции** — чем меньше время удержания блокировки, тем ниже вероятность дедлока

3. **NOWAIT + retry** — попытаться получить блокировку, при неудаче — повторить через время

4. **Более высокий уровень блокировки** — в редких случаях проще заблокировать таблицу целиком, чем бороться с дедлоками на строках

---

## Проблемы конкурентного доступа

| Проблема | Описание | Уровень изоляции, защищающий |
|---|---|---|
| Dirty read | Чтение незафиксированных данных другой транзакции | READ COMMITTED |
| Non-repeatable read | Повторное чтение той же строки даёт другой результат | REPEATABLE READ |
| Phantom read | Повторный запрос возвращает новые строки | SERIALIZABLE |
| Lost update | Два UPDATE перезаписывают друг друга | SELECT FOR UPDATE |

---

## Мониторинг блокировок

```sql
-- Текущие блокировки
SELECT * FROM pg_locks WHERE NOT granted;

-- Кто кого блокирует
SELECT
    blocked.pid AS blocked_pid,
    blocked_activity.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking_activity.query AS blocking_query
FROM pg_locks blocked
JOIN pg_locks blocking
    ON blocked.locktype = blocking.locktype
    AND blocked.database IS NOT DISTINCT FROM blocking.database
    AND blocked.relation IS NOT DISTINCT FROM blocking.relation
    AND blocked.pid != blocking.pid
JOIN pg_stat_activity blocked_activity ON blocked.pid = blocked_activity.pid
JOIN pg_stat_activity blocking_activity ON blocking.pid = blocking_activity.pid
WHERE NOT blocked.granted;

-- Длительные транзакции (потенциальные блокировщики)
SELECT pid, now() - xact_start AS duration, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;
```

```bash
# Принудительное завершение блокирующей транзакции
SELECT pg_cancel_backend(pid);       -- мягкое (SIGINT)
SELECT pg_terminate_backend(pid);    -- жёсткое (SIGTERM)
```

---

## Вопросы на собеседовании

**Какие уровни блокировок существуют в PostgreSQL?**
Три уровня: table-level (8 режимов от ACCESS SHARE до ACCESS EXCLUSIVE), row-level (FOR KEY SHARE, FOR SHARE, FOR NO KEY UPDATE, FOR KEY UPDATE) и advisory locks (пользовательские, не привязаны к объектам). Обычный SELECT берёт ACCESS SHARE на таблице — не конфликтует с DML, только с DDL (ALTER TABLE, DROP).

**Чем `SELECT FOR UPDATE` отличается от `SELECT FOR SHARE`?**
FOR UPDATE — эксклюзивная блокировка строки (другие транзакции не могут ни читать FOR UPDATE/SHARE, ни изменять). FOR SHARE — разделяемая (несколько транзакций могут FOR SHARE одновременно, но ни одна не может UPDATE). FOR UPDATE — для «прочитать и обновить», FOR SHARE — для «прочитать и убедиться, что не изменится».

**Зачем нужны advisory locks?**
Для координации на уровне приложения без блокировки реальных строк/таблиц. Пример: запуск миграции одним процессом (`pg_advisory_lock(migration_id)`), предотвращение дублирования cron-задач, распределённые блокировки в многосерверном окружении.

**Как найти и устранить блокировку в production?**
1. `pg_locks WHERE NOT granted` — найти заблокированные процессы. 2. Join с `pg_stat_activity` — увидеть какие запросы блокируют. 3. Решить: дождаться завершения, `pg_cancel_backend()` (мягко) или `pg_terminate_backend()` (жёстко). 4. Проанализировать причину: длинная транзакция, дедлок, неоптимальный запрос.

**Почему SELECT не блокирует UPDATE в PostgreSQL?**
Благодаря MVCC: SELECT читает snapshot (старую версию строки), UPDATE создаёт новую версию. Они работают с разными версиями строки и не конфликтуют. Табличная блокировка ACCESS SHARE (SELECT) совместима с ROW EXCLUSIVE (UPDATE).
