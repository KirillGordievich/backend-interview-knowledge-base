# Архитектура — Паттерны

## IoC (Inversion of Control) и DI (Dependency Injection)

### IoC — Инверсия управления

Архитектурная концепция: управление зависимостями передаётся от кода к внешней системе. Объекты не создают зависимости сами — получают их извне.

```python
# Без IoC: класс сам создаёт зависимости — жёсткая связь
class OrderService:
    def __init__(self):
        self.db = PostgresDB()          # нельзя заменить без изменения класса
        self.mailer = SMTPMailer()

# С IoC: зависимости передаются снаружи
class OrderService:
    def __init__(self, db: Database, mailer: Mailer):
        self.db = db
        self.mailer = mailer
```

### DI — Внедрение зависимостей

Конкретный паттерн IoC: зависимости передаются объекту при создании (constructor injection), через методы или свойства.

**Преимущества:**
- Слабая связность между компонентами
- Простота тестирования — зависимость заменяется mock-объектом
- Гибкая конфигурация: разные реализации для prod/test/dev

```python
# Тест: подменяем зависимости без изменения кода
class FakeMailer(Mailer):
    def __init__(self):
        self.sent = []
    def send(self, to, subject, body):
        self.sent.append((to, subject, body))

mailer = FakeMailer()
service = OrderService(db=InMemoryDB(), mailer=mailer)
service.place_order(order)
assert len(mailer.sent) == 1
```

**DI в FastAPI через Depends:**
```python
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

async def get_db() -> AsyncSession:
    async with SessionLocal() as session:
        yield session

@app.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    return await db.get(User, user_id)
```

---

## CQRS (Command Query Responsibility Segregation)

Разделение операций чтения (Query) и записи (Command).

**CQS — базовый принцип:**
- **Query** — возвращает данные, не изменяет состояние
- **Command** — изменяет состояние, не возвращает данные

**CQRS** — расширение: модели чтения и записи полностью раздельны (разные классы, репозитории, иногда разные БД).

```python
# Write side (команда)
@dataclass
class CreateOrderCommand:
    user_id: int
    items: list[OrderItem]

class CreateOrderHandler:
    def __init__(self, repo: OrderRepo, events: EventBus):
        self.repo = repo
        self.events = events

    async def handle(self, cmd: CreateOrderCommand) -> None:
        order = Order.create(cmd.user_id, cmd.items)
        await self.repo.save(order)
        await self.events.publish(OrderCreated(order.id))

# Read side (запрос — денормализованная, оптимизированная для чтения)
class GetOrderSummaryHandler:
    async def handle(self, order_id: int) -> OrderSummaryDTO:
        return await self.read_repo.get_summary(order_id)
```

**Когда применять:**
- Разные паттерны чтения и записи (читают в 10× больше чем пишут)
- Сложная бизнес-логика записи с событиями
- Нужна разная форма данных для чтения и хранения

---

## DDD (Domain-Driven Design)

Подход к проектированию, фокусирующийся на бизнес-домене.

**Многослойная архитектура:**
```
┌─────────────────────────┐
│   Presentation Layer    │  HTTP, CLI, очереди
├─────────────────────────┤
│   Application Layer     │  Use cases, команды, оркестрация
├─────────────────────────┤
│     Domain Layer        │  Бизнес-логика, сущности, агрегаты  ← ядро
├─────────────────────────┤
│  Infrastructure Layer   │  БД, внешние API, репозитории
└─────────────────────────┘
```

Зависимости направлены **внутрь** — Domain не знает о Infrastructure.

**Строительные блоки:**

| Концепция | Описание | Равенство |
|---|---|---|
| Entity | Объект с идентичностью (Order, User) | По ID |
| Value Object | Объект без идентичности (Money, Address) | По значению |
| Aggregate | Кластер объектов с Aggregate Root — единица транзакционности | По ID root |
| Repository | Абстракция хранилища для агрегата | — |
| Domain Event | Событие произошедшее в домене (OrderPlaced) | — |

**Bounded Context** — граница применимости модели. User в Auth-сервисе и User в Profile-сервисе — разные модели.

---

## Направление зависимости vs Поток данных

Два разных понятия, которые часто путают.

**Dependency Direction (Направление зависимости)** — кто знает о ком, кто использует кого.
`A → B` = A зависит от B, A знает о B.

**Data Flow Direction (Поток данных)** — откуда куда движутся данные.
`A → B` = данные текут из A в B.

```
Пример: Controller → Service → Repository → Database

Зависимость (кто знает о ком):
  Controller знает о Service
  Service знает о Repository
  Repository знает о Database

Поток данных (request/response):
  Request:  Controller → Service → Repository → Database
  Response: Database  → Repository → Service  → Controller

Направления могут совпадать или быть противоположными!
```

В **Clean Architecture** зависимости всегда направлены **внутрь** (к Domain), а поток данных идёт в обе стороны.

---

## Unit of Work

Поведенческий паттерн: объединяет несколько операций с репозиториями в одну логическую транзакцию. Либо все изменения фиксируются, либо ни одно.

**Проблема без UoW:**
```python
# Два репозитория — два разных соединения/транзакции. Нет атомарности.
await order_repo.save(order)
await inventory_repo.reserve(items)   # если упадёт — order сохранён, а items нет
```

**UoW знает обо всех репозиториях и управляет одной транзакцией:**

```python
from abc import ABC, abstractmethod
from sqlalchemy.ext.asyncio import AsyncSession

class UnitOfWork(ABC):
    orders: OrderRepository
    inventory: InventoryRepository

    @abstractmethod
    async def __aenter__(self): ...
    @abstractmethod
    async def __aexit__(self, *args): ...
    @abstractmethod
    async def commit(self): ...
    @abstractmethod
    async def rollback(self): ...


class SQLAlchemyUnitOfWork(UnitOfWork):
    def __init__(self, session_factory):
        self._session_factory = session_factory

    async def __aenter__(self):
        self._session: AsyncSession = self._session_factory()
        self.orders    = OrderRepository(self._session)
        self.inventory = InventoryRepository(self._session)
        return self

    async def __aexit__(self, exc_type, *args):
        if exc_type:
            await self.rollback()
        await self._session.close()

    async def commit(self):
        await self._session.commit()

    async def rollback(self):
        await self._session.rollback()


# Использование в сервисе
class OrderService:
    def __init__(self, uow: UnitOfWork):
        self.uow = uow

    async def place_order(self, user_id: int, items: list):
        async with self.uow as uow:
            order = Order(user_id=user_id, items=items)
            await uow.orders.save(order)
            await uow.inventory.reserve(items)   # в одной транзакции!
            await uow.commit()                    # или rollback при исключении
```

**Связь с репозиторием:** UoW — менеджер транзакций; Repository — абстракция доступа к данным. UoW объединяет несколько репозиториев под одну транзакцию.

**Связь с DDD:** Aggregate — единица бизнес-логики, UoW — единица транзакционности. Один `commit()` = одна атомарная операция.
