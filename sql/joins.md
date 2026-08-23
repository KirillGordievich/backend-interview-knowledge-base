# SQL — JOIN и агрегация

## Виды JOIN

### INNER JOIN

Возвращает только строки с совпадающими значениями в обеих таблицах. Строки без совпадений не попадают в результат.

```sql
SELECT u.name, o.amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id;
```

### LEFT JOIN

Возвращает все строки из левой таблицы. Для строк без совпадений в правой — NULL.

```sql
SELECT u.name, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
-- пользователи без заказов тоже попадут (amount = NULL)
```

### RIGHT JOIN

Аналогично LEFT JOIN, но все строки берутся из правой таблицы.

### FULL OUTER JOIN

Возвращает все строки из обеих таблиц. Где нет совпадений — NULL.

```sql
SELECT u.name, o.amount
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;
```

### Как визуально представить разницу между JOIN-ами

```
Таблица A: [1, 2, 3]    Таблица B: [2, 3, 4]

INNER JOIN:       A ∩ B  → [2, 3]
LEFT JOIN:    A + A ∩ B  → [1, 2, 3]  (4 отсутствует)
RIGHT JOIN:   B + A ∩ B  → [2, 3, 4]  (1 отсутствует)
FULL OUTER:       A ∪ B  → [1, 2, 3, 4]
```

---

## Как работает GROUP BY

Группирует строки с одинаковыми значениями в указанных столбцах. Используется с агрегатными функциями.

```sql
SELECT department, COUNT(*) AS headcount, AVG(salary) AS avg_salary
FROM employees
GROUP BY department;
```

**Агрегатные функции:** `COUNT`, `SUM`, `AVG`, `MAX`, `MIN`

### Чем HAVING отличается от WHERE

Фильтр по результатам группировки (аналог WHERE, но для групп):

```sql
SELECT department, COUNT(*) AS headcount
FROM employees
GROUP BY department
HAVING COUNT(*) > 5;
```

**WHERE vs HAVING:**

| | WHERE | HAVING |
|---|---|---|
| Применяется | До группировки | После группировки |
| Фильтрует | Отдельные строки | Группы |
| Агрегатные функции | Нельзя | Можно |

### В каком порядке выполняется SQL-запрос

SQL выполняется не в том порядке, в котором написан. Реальный порядок важен для понимания — например, алиасы из SELECT нельзя использовать в WHERE, потому что WHERE выполняется раньше:

```sql
FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

---

## Операции над множествами

### Что такое UNION и UNION ALL

Объединяет результаты двух SELECT. Столбцы должны совпадать по количеству и совместимым типам.

```sql
-- UNION: убирает дубликаты (медленнее)
SELECT name FROM customers
UNION
SELECT name FROM suppliers;

-- UNION ALL: оставляет дубликаты (быстрее — нет сортировки для дедупликации)
SELECT name FROM customers
UNION ALL
SELECT name FROM suppliers;
```

### Что такое INTERSECT

Возвращает строки, присутствующие в **обоих** результатах.

```sql
-- Клиенты, которые также являются поставщиками
SELECT name FROM customers
INTERSECT
SELECT name FROM suppliers;
```

### Что такое EXCEPT

Возвращает строки из первого результата, которых **нет** во втором.

```sql
-- Клиенты, которые не являются поставщиками
SELECT name FROM customers
EXCEPT
SELECT name FROM suppliers;
```

### Когда использовать вместо JOIN

| Задача | Предпочтительно |
|---|---|
| Объединить одинаковые данные из разных таблиц | UNION ALL |
| Найти общие записи | INTERSECT или INNER JOIN |
| Найти записи только в одной таблице | EXCEPT или LEFT JOIN + WHERE IS NULL |
