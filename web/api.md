# Web — API

## REST

REST (Representational State Transfer) — архитектурный стиль для API поверх HTTP.

**Принципы:**
- **Stateless** — каждый запрос содержит всё необходимое, сервер не хранит состояние клиента
- **Uniform Interface** — единообразный интерфейс через HTTP методы + URI
- **Client-Server** — разделение ответственности
- **Cacheable** — ответы могут кэшироваться

**Структура URL:**
```
GET    /users          — список пользователей
POST   /users          — создать пользователя
GET    /users/42       — конкретный пользователь
PUT    /users/42       — заменить пользователя целиком
PATCH  /users/42       — частично обновить
DELETE /users/42       — удалить
GET    /users/42/posts — посты пользователя 42
```

---

## OpenAPI (Swagger)

Стандарт описания REST API. Позволяет генерировать документацию и клиентский код автоматически.

```yaml
openapi: 3.0.0
info:
  title: Users API
  version: 1.0.0
paths:
  /users/{id}:
    get:
      summary: Получить пользователя
      parameters:
        - name: id
          in: path
          required: true
          schema: {type: integer}
      responses:
        '200':
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          description: Не найден
components:
  schemas:
    User:
      type: object
      properties:
        id: {type: integer}
        name: {type: string}
        email: {type: string, format: email}
```

**FastAPI** генерирует OpenAPI-схему автоматически:

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI(title="My API", version="1.0.0")

class UserResponse(BaseModel):
    id: int
    name: str
    email: str

@app.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: int):
    ...

# /docs  → Swagger UI (интерактивная документация)
# /redoc → ReDoc (читабельная документация)
# /openapi.json → сырая схема
```
