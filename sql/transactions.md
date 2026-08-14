# SQL — Транзакции и ACID

## Транзакция

Транзакция — группа операций, которая выполняется как единое целое: либо все операции применяются, либо ни одна.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;   -- применить
-- ROLLBACK; — отменить всё
```

---

## ACID

ACID — набор свойств, гарантирующих надёжность транзакций.

### Atomicity — Атомарность

Транзакция выполняется полностью или не выполняется совсем. Промежуточные состояния недопустимы.

При сбое во время транзакции все изменения откатываются.

### Consistency — Согласованность

Каждая успешная транзакция переводит базу из одного корректного состояния в другое. Все ограничения целостности (CHECK, FK, UNIQUE, NOT NULL) соблюдаются.

### Isolation — Изолированность

Параллельные транзакции не влияют на результат друг друга во время выполнения.

Подробнее — см. раздел «Уровни изоляции» ниже.

### Durability — Надёжность

После получения подтверждения `COMMIT` данные сохранены навсегда — даже при отключении питания или сбое оборудования (за счёт WAL-журнала).

---

## Аномалии параллельного доступа

При параллельной работе транзакций возможны аномалии — ситуации, когда результат отличается от последовательного выполнения.

### Dirty Read (грязное чтение)

Транзакция читает данные, которые другая транзакция изменила, но ещё не зафиксировала. Если та откатится — были прочитаны данные, которых «никогда не существовало».

```
T1: UPDATE accounts SET balance = 0 WHERE id = 1;
T2: SELECT balance FROM accounts WHERE id = 1;  → 0 (незафиксировано!)
T1: ROLLBACK;
-- T2 прочитала баланс 0, хотя реально он не менялся
```

### Non-repeatable Read (неповторяемое чтение)

Повторное чтение той же строки внутри транзакции возвращает другой результат, потому что другая транзакция изменила и зафиксировала данные между чтениями.

```
T1: SELECT balance FROM accounts WHERE id = 1;  → 1000
T2: UPDATE accounts SET balance = 500 WHERE id = 1; COMMIT;
T1: SELECT balance FROM accounts WHERE id = 1;  → 500 (другое значение!)
```

### Phantom Read (фантомное чтение)

Повторный запрос с тем же условием WHERE возвращает другой набор строк, потому что другая транзакция вставила или удалила строки.

```
T1: SELECT * FROM orders WHERE status = 'new';  → 5 строк
T2: INSERT INTO orders (status) VALUES ('new'); COMMIT;
T1: SELECT * FROM orders WHERE status = 'new';  → 6 строк (фантом!)
```

### Serialization Anomaly (аномалия сериализации)

Результат параллельного выполнения транзакций не соответствует ни одному возможному порядку их последовательного выполнения.

```
-- Обе транзакции читают сумму и вставляют строку
T1: SELECT SUM(amount) FROM transactions;  → 100
T2: SELECT SUM(amount) FROM transactions;  → 100
T1: INSERT INTO transactions (amount) VALUES (100); COMMIT;
T2: INSERT INTO transactions (amount) VALUES (100); COMMIT;
-- Последовательно: T1 увидела бы 100, T2 — 200 (или наоборот)
-- Параллельно: обе увидели 100 — невозможно при последовательном выполнении
```

---

## Уровни изоляции

Уровень изоляции определяет, какие аномалии допустимы.

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- или при старте транзакции
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

### Матрица аномалий (стандарт SQL)

| Уровень | Dirty Read | Non-repeatable Read | Phantom Read | Serialization Anomaly |
|---|---|---|---|---|
| READ UNCOMMITTED | Возможно | Возможно | Возможно | Возможно |
| READ COMMITTED | Нет | Возможно | Возможно | Возможно |
| REPEATABLE READ | Нет | Нет | Возможно | Возможно |
| SERIALIZABLE | Нет | Нет | Нет | Нет |

### READ UNCOMMITTED

Самый слабый уровень. Транзакция видит незафиксированные изменения других транзакций (dirty read).

На практике почти не используется. **PostgreSQL не поддерживает** — трактует как READ COMMITTED.

### READ COMMITTED (PostgreSQL по умолчанию)

Каждый оператор видит только данные, зафиксированные на момент начала **этого оператора** (не транзакции).

```sql
-- T1
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- видит 1000
-- T2 меняет balance на 500 и делает COMMIT
SELECT balance FROM accounts WHERE id = 1;  -- видит 500 (новый snapshot)
COMMIT;
```

Защищает от dirty read, но не от non-repeatable read и phantom read.

### REPEATABLE READ

Транзакция видит snapshot данных на момент начала **транзакции**. Все SELECT внутри транзакции видят одни и те же данные.

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 1;  -- видит 1000
-- T2 меняет balance на 500 и делает COMMIT
SELECT balance FROM accounts WHERE id = 1;  -- всё ещё 1000 (snapshot)
COMMIT;
```

**PostgreSQL особенность:** REPEATABLE READ в PostgreSQL также защищает от phantom read (за счёт MVCC/SSI). Это строже, чем требует стандарт SQL.

При конфликте обновлений — ошибка:
```
ERROR: could not serialize access due to concurrent update
```
Приложение должно поймать ошибку и повторить транзакцию.

### SERIALIZABLE

Самый строгий уровень. Гарантирует, что результат параллельного выполнения эквивалентен какому-либо последовательному порядку.

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT SUM(amount) FROM transactions;
INSERT INTO transactions (amount) VALUES (100);
COMMIT;
-- при конфликте: ERROR: could not serialize access
```

PostgreSQL реализует через SSI (Serializable Snapshot Isolation) — отслеживает зависимости между транзакциями и откатывает при обнаружении цикла.

**Цена:** больше откатов (serialization failure), приложение обязано уметь retry.

---

## Реализация изоляции

### MVCC (Multi-Version Concurrency Control)

PostgreSQL (и другие БД) реализуют изоляцию через MVCC — каждая транзакция работает со своим snapshot данных:

- Каждая строка имеет `xmin` (транзакция создавшая) и `xmax` (транзакция удалившая/обновившая)
- SELECT не блокирует UPDATE, UPDATE не блокирует SELECT
- Читатели не блокируют писателей, писатели не блокируют читателей
- «Старые» версии строк очищаются VACUUM

### Блокировки vs MVCC

| Подход | Как работает | Пример |
|---|---|---|
| **Пессимистичный** (блокировки) | Блокировать данные перед изменением | `SELECT FOR UPDATE` |
| **Оптимистичный** (MVCC) | Работать с snapshot, проверять конфликты при COMMIT | REPEATABLE READ, SERIALIZABLE |

---

## SAVEPOINT

Позволяет откатить часть транзакции, не откатывая всю:

```sql
BEGIN;
INSERT INTO orders (id, total) VALUES (1, 100);

SAVEPOINT sp1;
INSERT INTO order_items (order_id, qty) VALUES (1, 5);
-- Ошибка! Откатываем до savepoint
ROLLBACK TO sp1;

-- Продолжаем транзакцию
INSERT INTO order_items (order_id, qty) VALUES (1, 3);
COMMIT;  -- order создан, item с qty=3 сохранён
```

---

## Вопросы на собеседовании

**Какой уровень изоляции по умолчанию в PostgreSQL?**
READ COMMITTED. Каждый оператор в транзакции видит данные, зафиксированные на момент начала этого оператора. Разные SELECT в одной транзакции могут видеть разные данные (non-repeatable read).

**Чем REPEATABLE READ отличается от SERIALIZABLE?**
REPEATABLE READ фиксирует snapshot на момент начала транзакции — все чтения видят одни данные. Но возможны аномалии сериализации (write skew). SERIALIZABLE гарантирует, что результат эквивалентен последовательному выполнению — при конфликте одна транзакция откатывается.

**Что такое MVCC?**
Multi-Version Concurrency Control — механизм, при котором каждая транзакция работает со своим snapshot данных. Старые версии строк хранятся рядом с новыми. Читатели не блокируют писателей, писатели не блокируют читателей. Устаревшие версии очищает VACUUM.

**Что произойдёт при конфликте на REPEATABLE READ?**
Если две транзакции на REPEATABLE READ пытаются обновить одну строку — вторая получит ошибку `could not serialize access due to concurrent update`. Приложение должно поймать ошибку и повторить транзакцию (retry loop).

**Почему не использовать SERIALIZABLE всегда?**
Больше откатов из-за serialization failure → нужен retry в приложении → больше латентности. PostgreSQL использует SSI — отслеживает зависимости, что создаёт overhead по памяти и CPU. Для большинства задач READ COMMITTED + явные блокировки (`SELECT FOR UPDATE`) достаточны.

**Что такое write skew?**
Аномалия при REPEATABLE READ: две транзакции читают одни данные, на основе прочитанного делают разные записи, и результат нарушает инвариант. Пример: два врача проверяют «есть ли хотя бы один дежурный» и оба снимают себя с дежурства. Каждый видел что дежурный есть (другой врач), но в итоге дежурных ноль. Защита: SERIALIZABLE или явные блокировки.
