# SQL — CTE и подзапросы

## Подзапросы (Subqueries)

Запрос внутри другого запроса. Может быть в SELECT, FROM, WHERE, HAVING.

```sql
-- В WHERE
SELECT name FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);

-- В FROM (derived table)
SELECT dept, avg_sal
FROM (
    SELECT department AS dept, AVG(salary) AS avg_sal
    FROM employees
    GROUP BY department
) AS dept_stats
WHERE avg_sal > 60000;

-- В SELECT (скалярный подзапрос — возвращает одно значение)
SELECT name, salary,
    (SELECT AVG(salary) FROM employees) AS company_avg
FROM employees;
```

### Коррелированный подзапрос

Ссылается на столбец из внешнего запроса — выполняется для каждой строки внешнего запроса.

```sql
-- Сотрудники с зарплатой выше средней по своему отделу
SELECT name, department, salary
FROM employees e
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
    WHERE department = e.department  -- ссылка на внешний запрос
);
```

**Недостаток:** выполняется N раз (по одному для каждой строки). Часто можно заменить JOIN или оконной функцией.

---

## CTE (Common Table Expressions)

Именованный временный результат, определяемый через `WITH`. Читается сверху вниз, улучшает читаемость.

```sql
WITH dept_avg AS (
    SELECT department, AVG(salary) AS avg_sal
    FROM employees
    GROUP BY department
)
SELECT e.name, e.salary, d.avg_sal
FROM employees e
JOIN dept_avg d ON e.department = d.department
WHERE e.salary > d.avg_sal;
```

### Несколько CTE

```sql
WITH
high_earners AS (
    SELECT * FROM employees WHERE salary > 80000
),
senior_employees AS (
    SELECT * FROM employees WHERE tenure_years > 5
)
SELECT h.name
FROM high_earners h
JOIN senior_employees s ON h.id = s.id;
```

---

## Рекурсивные CTE

Позволяют обходить иерархические структуры (деревья, графы). Состоят из двух частей, соединённых `UNION ALL`:
- **Базовый случай** — начальные строки
- **Рекурсивный шаг** — ссылка на сам CTE

```sql
-- Иерархия сотрудников (менеджер → подчинённые)
WITH RECURSIVE org_chart AS (
    -- базовый случай: CEO (без менеджера)
    SELECT id, name, manager_id, 1 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- рекурсивный шаг: подчинённые каждого найденного сотрудника
    SELECT e.id, e.name, e.manager_id, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT * FROM org_chart ORDER BY level, name;
```

```sql
-- Числовая последовательность 1..10
WITH RECURSIVE nums AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM nums WHERE n < 10
)
SELECT * FROM nums;
```

---

## CTE vs Подзапрос vs Временная таблица

| | CTE | Подзапрос | Временная таблица |
|---|---|---|---|
| Читаемость | Высокая | Низкая (вложенность) | Средняя |
| Переиспользование | Только в одном запросе | Нет | Да |
| Рекурсия | Да | Нет | Нет |
| Материализация | Зависит от СУБД | Нет | Да (физически) |
| Индексы | Нет | Нет | Можно создать |
| Когда использовать | Читаемость, рекурсия | Простые однократные фильтры | Сложная логика, переиспользование |

---

## EXISTS vs IN vs JOIN

```sql
-- EXISTS: останавливается на первом совпадении — эффективен для больших таблиц
SELECT name FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);

-- IN: вычисляет весь подзапрос заранее
SELECT name FROM customers
WHERE id IN (SELECT customer_id FROM orders);

-- JOIN: гибче, позволяет получать данные из обеих таблиц
SELECT DISTINCT c.name
FROM customers c
JOIN orders o ON c.id = o.customer_id;
```

**Рекомендация:** `EXISTS` предпочтительнее `IN` с подзапросом при большом наборе данных. С конкретными значениями `IN (1, 2, 3)` — нет разницы.
