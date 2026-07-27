# SQL — Оконные функции

## Что такое оконные функции

Оконные функции выполняют вычисления по набору строк, связанных с текущей строкой, **не схлопывая их в одну** (в отличие от GROUP BY).

```sql
-- GROUP BY: 3 строки → 1 строка на отдел
SELECT department, AVG(salary) FROM employees GROUP BY department;

-- Оконная функция: каждая строка сохраняется + добавляется средняя по отделу
SELECT name, department, salary,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees;
```

---

## Синтаксис OVER

```sql
функция() OVER (
    [PARTITION BY ...]   -- разбить на окна (как GROUP BY, но без схлопывания)
    [ORDER BY ...]       -- порядок строк внутри окна
    [ROWS|RANGE ...]     -- рамка окна (frame)
)
```

---

## Ранжирующие функции

```sql
SELECT name, salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn,      -- уникальный номер (1,2,3,4,5)
    RANK()       OVER (ORDER BY salary DESC) AS rnk,     -- с пропусками при ничьей (1,2,2,4,5)
    DENSE_RANK() OVER (ORDER BY salary DESC) AS drnk     -- без пропусков (1,2,2,3,4)
FROM employees;
```

| salary | ROW_NUMBER | RANK | DENSE_RANK |
|--------|-----------|------|------------|
| 100    | 1         | 1    | 1          |
| 90     | 2         | 2    | 2          |
| 90     | 3         | 2    | 2          |
| 80     | 4         | 4    | 3          |

### NTILE(n)

Делит строки на n примерно равных групп:

```sql
SELECT name, salary,
    NTILE(4) OVER (ORDER BY salary DESC) AS quartile
FROM employees;
```

---

## Функции смещения

```sql
SELECT name, salary, hire_date,
    LAG(salary, 1)  OVER (ORDER BY hire_date) AS prev_salary,   -- предыдущая строка
    LEAD(salary, 1) OVER (ORDER BY hire_date) AS next_salary,    -- следующая строка
    FIRST_VALUE(salary) OVER (ORDER BY hire_date) AS first_sal,  -- первая в окне
    LAST_VALUE(salary)  OVER (
        ORDER BY hire_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_sal
FROM employees;
```

`LAG/LEAD` принимают второй параметр — смещение (по умолчанию 1), и третий — значение по умолчанию если строка не существует.

---

## Агрегатные функции как оконные

```sql
SELECT name, department, salary,
    SUM(salary)   OVER (PARTITION BY department) AS dept_total,
    AVG(salary)   OVER (PARTITION BY department) AS dept_avg,
    COUNT(*)      OVER (PARTITION BY department) AS dept_count,
    -- нарастающий итог внутри отдела по дате найма
    SUM(salary)   OVER (
        PARTITION BY department
        ORDER BY hire_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM employees;
```

---

## Рамка окна (Frame)

Определяет, какие строки включаются в вычисление относительно текущей.

```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW   -- от начала до текущей
ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING           -- 2 строки до и после
ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING   -- от текущей до конца
```

**ROWS vs RANGE:**
- `ROWS` — физические строки (по позиции)
- `RANGE` — логический диапазон (все строки с одинаковым ORDER BY значением)

По умолчанию при наличии `ORDER BY`: `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`

---

## Типичные задачи

```sql
-- Топ-1 по зарплате в каждом отделе
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
    FROM employees
) t WHERE rn = 1;

-- Процент от общей суммы
SELECT name, salary,
    ROUND(salary * 100.0 / SUM(salary) OVER (), 2) AS pct_of_total
FROM employees;

-- Разница с предыдущим значением (изменение продаж)
SELECT date, revenue,
    revenue - LAG(revenue) OVER (ORDER BY date) AS delta
FROM daily_sales;
```
