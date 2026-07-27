# Тестирование

## Виды тестов

| Вид | Что тестирует | Скорость | Изоляция | Доля в пирамиде |
|---|---|---|---|---|
| Unit | Одну функцию/класс | Очень быстро | Полная | Много |
| Integration | Несколько компонентов | Средне | Частичная | Меньше |
| E2E (end-to-end) | Всю систему как пользователь | Медленно | Никакой | Мало |

**Пирамида тестирования:** много unit → меньше integration → мало e2e.

---

## Unit-тестирование (pytest)

```python
import pytest
from decimal import Decimal
from app.services import calculate_discount

def test_no_discount_below_threshold():
    assert calculate_discount(Decimal("50"), items=3) == Decimal("0")

def test_discount_applied_above_threshold():
    assert calculate_discount(Decimal("200"), items=5) == Decimal("20")

# Параметризация
@pytest.mark.parametrize("amount,items,expected", [
    (Decimal("50"),  3, Decimal("0")),
    (Decimal("200"), 5, Decimal("20")),
    (Decimal("500"), 10, Decimal("75")),
])
def test_discount_parametrized(amount, items, expected):
    assert calculate_discount(amount, items) == expected

# Ожидание исключений
def test_negative_amount_raises():
    with pytest.raises(ValueError, match="amount must be positive"):
        calculate_discount(Decimal("-1"), items=1)
```

---

## Mocking и Stubbing

**Mock** — объект-заменитель, записывающий вызовы. Используется для проверки **взаимодействия** (был ли вызван метод, с какими аргументами).

**Stub** — возвращает заранее заданный ответ. Используется для **изоляции** тестируемого кода от реальных зависимостей.

```python
from unittest.mock import Mock, patch, AsyncMock, MagicMock

# Mock — проверяем что метод был вызван
def test_sends_email_on_order_creation():
    email_sender = Mock()
    service = OrderService(email_sender=email_sender)

    service.place_order(order)

    email_sender.send.assert_called_once_with(
        to="user@example.com",
        subject="Order #123 confirmed"
    )

# patch — подменяем зависимость в модуле
def test_with_patch():
    with patch('app.services.send_email') as mock_send:
        mock_send.return_value = None     # stub: всегда None
        result = place_order(order)
        assert result.status == 'confirmed'
        mock_send.assert_called_once()

# AsyncMock — для корутин
@pytest.mark.asyncio
async def test_async_service():
    mock_db = AsyncMock()
    mock_db.get_user.return_value = User(id=1, name="Alice")

    service = UserService(db=mock_db)
    user = await service.get_user(1)

    assert user.name == "Alice"
    mock_db.get_user.assert_awaited_once_with(1)
```

---

## Фикстуры (pytest fixtures)

```python
import pytest
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine

@pytest.fixture
def user():
    return User(id=1, name="Alice", email="alice@example.com")

@pytest.fixture
def order(user):
    return Order(id=100, user=user, items=[...])

# Асинхронная фикстура с реальной БД
@pytest.fixture
async def db_session():
    engine = create_async_engine("postgresql+asyncpg://...")
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    async with AsyncSession(engine) as session:
        yield session

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)

def test_order_belongs_to_user(order, user):
    assert order.user_id == user.id
```

**Scope фикстур:** `function` (по умолчанию), `class`, `module`, `session` — определяет как часто фикстура пересоздаётся.

---

## E2E тестирование

Тестирует приложение целиком — реальный HTTP, реальная БД, реальные зависимости.

```python
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_create_and_get_order():
    async with AsyncClient(app=app, base_url="http://test") as client:
        # Создать
        create_resp = await client.post("/orders", json={
            "user_id": 1,
            "items": [{"product_id": 42, "quantity": 2}]
        }, headers={"Authorization": "Bearer test-token"})
        assert create_resp.status_code == 201
        order_id = create_resp.json()["id"]

        # Получить
        get_resp = await client.get(f"/orders/{order_id}")
        assert get_resp.status_code == 200
        assert get_resp.json()["status"] == "pending"
```

---

## CI/CD

**CI (Continuous Integration)** — автоматическая сборка и тестирование при каждом пуше/PR.

**CD (Continuous Delivery)** — код всегда готов к деплою.
**CD (Continuous Deployment)** — деплой происходит автоматически при успешном CI.

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        options: --health-cmd pg_isready

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install -e ".[dev]"
      - run: ruff check .                    # линтер
      - run: mypy app/                       # типы
      - run: pytest --cov=app --cov-report=xml  # тесты + coverage
```

**Что обычно включает CI-пайплайн:**
1. Установка зависимостей
2. Линтеры (ruff, flake8)
3. Статический анализ типов (mypy)
4. Юнит и интеграционные тесты
5. Покрытие кода (coverage)
6. Сборка Docker-образа
7. Security scan (bandit, safety)
8. Деплой (только при пуше в main)
