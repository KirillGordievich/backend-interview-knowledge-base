# SQL — Представления (Views)

## Обычное представление (View)

Именованный SELECT-запрос, сохранённый в БД. При обращении к view запрос выполняется заново — данные не хранятся физически.

```sql
CREATE VIEW active_users AS
SELECT id, name, email
FROM users
WHERE status = 'active';

-- Использование как обычная таблица
SELECT * FROM active_users WHERE name LIKE 'A%';

-- Удаление
DROP VIEW active_users;
```

**Плюсы:**
- Инкапсуляция сложной логики — клиент не знает о деталях реализации
- Безопасность — можно дать доступ к view, скрыв чувствительные столбцы
- Переиспользование запросов

**Минусы:**
- Выполняется заново при каждом обращении
- Нельзя создать индекс
- Может маскировать неэффективные запросы

---

## Материализованное представление (Materialized View)

Результат SELECT физически сохраняется на диске. Читается как таблица — быстро. Нужно явно обновлять.

```sql
CREATE MATERIALIZED VIEW monthly_revenue AS
SELECT
    date_trunc('month', created_at) AS month,
    SUM(amount) AS revenue
FROM orders
WHERE status = 'paid'
GROUP BY 1;

-- Создать индекс (как на обычной таблице)
CREATE INDEX ON monthly_revenue(month);

-- Обновить данные
REFRESH MATERIALIZED VIEW monthly_revenue;

-- Обновить без блокировки чтения (данные не меняются до завершения)
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue;
```

**Плюсы:**
- Быстрое чтение — данные уже вычислены
- Можно индексировать
- Подходит для дорогих агрегаций

**Минусы:**
- Данные могут устареть
- Требует места на диске
- `REFRESH` может быть медленным на больших данных

---

## View vs Materialized View

| | View | Materialized View |
|---|---|---|
| Хранение данных | Нет (только запрос) | Да (физически) |
| Скорость чтения | Зависит от запроса | Быстро (предвычислено) |
| Актуальность данных | Всегда свежие | Устаревают до REFRESH |
| Индексы | Нельзя | Можно |
| Когда использовать | Упрощение запросов, безопасность | Дорогие агрегации, отчёты, дашборды |

---

## Обновляемые представления

Простые view (без JOIN, GROUP BY, DISTINCT, агрегатов) поддерживают INSERT/UPDATE/DELETE:

```sql
CREATE VIEW active_users AS
SELECT id, name, email FROM users WHERE status = 'active';

-- Работает, если view обновляема
UPDATE active_users SET name = 'Bob' WHERE id = 1;
```

Для сложных view — использовать `INSTEAD OF` триггеры.
