# SQL — Ключи, ограничения и отношения

## Какие ограничения (constraints) бывают в SQL

Ограничения — это правила целостности данных, которые задаются на уровне схемы. Они гарантируют, что данные в таблице всегда соответствуют определённым условиям.

### Что такое PRIMARY KEY

Уникально идентифицирует каждую строку. Не может содержать NULL. Один на таблицу. Автоматически создаёт уникальный индекс.

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email TEXT NOT NULL
);

-- Составной первичный ключ
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    PRIMARY KEY (order_id, product_id)
);
```

### Что такое FOREIGN KEY

Ссылочная целостность — значение должно существовать в родительской таблице.

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id) ON DELETE CASCADE
);
```

**Действия при удалении/обновлении родителя:**

| Действие | Поведение |
|---|---|
| `CASCADE` | Удалить/обновить дочерние строки автоматически |
| `SET NULL` | Поставить NULL в дочерней строке |
| `SET DEFAULT` | Поставить значение по умолчанию |
| `RESTRICT` | Запретить удаление, если есть дочерние строки |
| `NO ACTION` | Аналог RESTRICT, но проверяется в конце транзакции |

### Что такое UNIQUE

Запрещает дублирование значений. В отличие от PRIMARY KEY — может быть несколько на таблицу и допускает NULL (обычно один NULL на столбец).

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email TEXT UNIQUE,
    phone TEXT UNIQUE
);
```

### Что такое CHECK

Произвольное условие на значение.

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    price NUMERIC CHECK (price > 0),
    discount NUMERIC CHECK (discount BETWEEN 0 AND 100)
);
```

### Что делают NOT NULL и DEFAULT

`NOT NULL` запрещает вставку NULL в столбец. `DEFAULT` задаёт значение, которое подставляется автоматически, если при INSERT столбец не указан:

```sql
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Какие виды отношений между таблицами бывают

### Один-к-одному (1:1)

Каждая запись в таблице A связана максимум с одной записью в таблице B.

```sql
CREATE TABLE users (id SERIAL PRIMARY KEY, name TEXT);
CREATE TABLE user_profiles (
    user_id INT PRIMARY KEY REFERENCES users(id),  -- PK = FK → гарантирует 1:1
    bio TEXT
);
```

### Один-ко-многим (1:N)

Один пользователь — много заказов. Самое распространённое отношение.

```sql
CREATE TABLE users  (id SERIAL PRIMARY KEY, name TEXT);
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),  -- FK на стороне "многих"
    amount NUMERIC
);
```

### Многие-ко-многим (M:N)

Реализуется через промежуточную (junction) таблицу.

```sql
CREATE TABLE students (id SERIAL PRIMARY KEY, name TEXT);
CREATE TABLE courses  (id SERIAL PRIMARY KEY, title TEXT);

CREATE TABLE enrollments (  -- junction table
    student_id INT REFERENCES students(id) ON DELETE CASCADE,
    course_id  INT REFERENCES courses(id)  ON DELETE CASCADE,
    enrolled_at DATE,
    PRIMARY KEY (student_id, course_id)
);
```

---

## В чём разница между суррогатным и натуральным ключом

| | Суррогатный | Натуральный |
|---|---|---|
| Пример | `id SERIAL` | `email`, `passport_no` |
| Стабильность | Никогда не меняется | Может измениться |
| Читаемость | Нет смысла | Понятен из данных |
| Размер | Маленький (int) | Может быть большим |
| **Рекомендация** | Предпочтителен | Только если 100% стабилен и уникален |
