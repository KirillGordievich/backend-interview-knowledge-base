# SQL — ORM, N+1, SQL-инъекции

## ORM (Object-Relational Mapping)

Преимущества ORM:
- **Безопасность** — параметры подставляются автоматически, снижая риск SQL-инъекций
- **Читаемость** — запросы выглядят как код на языке приложения
- **Переносимость** — ORM генерирует SQL под конкретную СУБД
- **Миграции** — SQLAlchemy, Django ORM поддерживают автоматическое создание миграций

---

## Проблема N+1

Возникает при загрузке связанных данных: вместо одного JOIN-запроса выполняется 1 запрос за списком + N запросов за связанными данными каждого элемента.

**Пример:**
```python
users = session.query(User).all()   # 1 запрос
for user in users:
    print(user.posts)               # N запросов (один на каждого пользователя)
```

При 100 пользователях — 101 запрос к БД.

**Решение — жадная загрузка (eager loading):**
```python
# SQLAlchemy
from sqlalchemy.orm import joinedload
users = session.query(User).options(joinedload(User.posts)).all()

# Django ORM
users = User.objects.prefetch_related('posts')
```

Итог — 2 запроса вместо 101:
```sql
SELECT * FROM users;
SELECT * FROM posts WHERE user_id IN (1, 2, 3, ..., 100);
```

---

## SQL-инъекции

Уязвимость возникает, когда пользовательский ввод подставляется напрямую в строку запроса:

```python
# НЕБЕЗОПАСНО
query = f"SELECT * FROM users WHERE name = '{user_input}'"
# user_input = "' OR '1'='1" → вернёт всех пользователей
```

**Защита — параметризованные запросы:**
```python
# asyncpg (PostgreSQL)
await conn.fetch("SELECT * FROM users WHERE name = $1", user_input)

# psycopg2
cursor.execute("SELECT * FROM users WHERE name = %s", (user_input,))
```

ORM автоматически параметризует запросы — ещё один довод в его пользу для стандартных операций.

---

## Когда использовать чистый SQL

- Сложные запросы с оконными функциями, рекурсивными CTE, которые ORM не поддерживает
- Сервис только для аналитических/отчётных запросов — без CRUD-слоя и миграций
- ORM генерирует неэффективный запрос и его нельзя оптимизировать через API ORM
- Нестандартные SQL-конструкции: `COPY`, `LATERAL`, специфика конкретной СУБД
