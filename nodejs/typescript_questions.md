# TypeScript: Вопросы

> Теория: [typescript.md](typescript.md)

---

## Базовые

**Q: Зачем нужен TypeScript?**

Статическая типизация поверх JavaScript. Ошибки находятся на этапе компиляции, а не в рантайме. Лучшее автодополнение в IDE, документация API в коде, безопасный рефакторинг.

---

**Q: `any` vs `unknown` — в чём разница?**

- `any` — отключает проверку типов, можно делать что угодно (опасно)
- `unknown` — безопасная альтернатива: принимает любое значение, но использовать можно только после проверки типа (type guard/narrowing)

```ts
let a: any = "hello";
a.foo();  // OK для компилятора (возможная ошибка в рантайме)

let u: unknown = "hello";
u.foo();  // Ошибка! Нужно сначала проверить тип
```

---

**Q: `void` vs `never` — в чём разница?**

- `void` — функция ничего не возвращает (или возвращает `undefined`)
- `never` — функция никогда не возвращает управление (throw, бесконечный цикл) или тип, который не может существовать

`never` используется для exhaustive check — компилятор проверяет что все случаи обработаны.

---

**Q: `interface` vs `type` — когда что?**

- `interface` — для объектов/классов, поддерживает declaration merging и `extends`
- `type` — для union-типов, intersection, mapped types, conditional types

Правило: `interface` для public API и объектов, `type` для всего остального.

---

**Q: Что такое Union и Intersection типы?**

- **Union** (`A | B`) — значение может быть типа A **или** B. Доступны только общие поля/методы.
- **Intersection** (`A & B`) — значение должно удовлетворять **оба** типа. Объединяет все поля.

```ts
// Union — "одно из"
type Result = Success | Error;
function handle(r: Result) {
    if ("data" in r) { /* Success */ }
}

// Intersection — "всё вместе"
type Admin = User & { role: "admin"; permissions: string[] };
// Admin имеет все поля User + role + permissions
```

Частая ошибка: union на объектах — не "объединение полей", а "один из объектов".

---

**Q: Что такое declaration merging?**

Уникальная возможность `interface` — несколько объявлений с одним именем сливаются в одно. Полезно для расширения типов библиотек:

```ts
interface Window {
    myPlugin: () => void;  // дополняет существующий интерфейс Window
}
```

---

**Q: `interface` vs `abstract class` — когда что?**

- `interface` — стирается при компиляции, нет runtime кода, можно `implements` несколько
- `abstract class` — компилируется в JS, может содержать реализацию методов, только одно наследование

Используй `interface` когда нужна только форма. `abstract class` — когда нужна общая реализация.

---

**Q: `implements` vs `extends` — в чём разница?**

- `extends` — наследование: класс получает реализацию (методы, свойства) от родителя. Только один `extends`.
- `implements` — контракт: класс обязуется реализовать интерфейс. Можно `implements` несколько. Не наследует реализацию.

```ts
interface Serializable {
    serialize(): string;
}

interface Loggable {
    log(): void;
}

class Base {
    id = Math.random();
}

// extends — получает id от Base
// implements — обязан реализовать serialize() и log()
class User extends Base implements Serializable, Loggable {
    serialize() { return JSON.stringify(this); }
    log() { console.log(this.id); }
}
```

---

## Generics

**Q: Что такое Generics и зачем нужны?**

Параметрический полиморфизм — функция/класс работает с разными типами, сохраняя информацию о типе. Позволяет писать переиспользуемый типобезопасный код.

```ts
function identity<T>(value: T): T { return value; }
identity("hello")  // T = string
identity(42)       // T = number
```

---

**Q: Что такое `keyof`?**

Оператор, возвращающий union всех ключей типа:

```ts
interface User { id: number; name: string; email: string }
type UserKeys = keyof User;  // "id" | "name" | "email"
```

С индексным доступом `T[K]` можно получить тип значения по ключу:

```ts
type NameType = User["name"];         // string
type IdOrName = User["id" | "name"];  // number | string
```

---

**Q: Что означает `K extends keyof T`?**

Ограничивает K только существующими ключами типа T. Компилятор не даст передать произвольную строку — только реальные поля объекта:

```ts
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
    return obj[key];
}

const user: User = { id: 1, name: "Alice", email: "a@b.com" };
getProperty(user, "name");  // OK → string
getProperty(user, "foo");   // Ошибка! — "foo" не ключ User
```

Этот паттерн — основа для `Pick`, `Omit`, `Record`, типизации событий и builder-паттернов.

---

**Q: Что такое `extends` в Generics?**

Ограничение (constraint) — T должен быть совместим с указанным типом:

```ts
// T должен иметь поле length
function logLength<T extends { length: number }>(x: T): void {
    console.log(x.length);
}
logLength("hello");  // OK — у string есть length
logLength([1, 2]);   // OK — у массива есть length
logLength(42);       // Ошибка! — у number нет length
```

---

## Utility Types

**Q: Назовите основные Utility Types.**

- `Partial<T>` — все поля опциональные
- `Required<T>` — все обязательные
- `Readonly<T>` — все readonly
- `Pick<T, K>` — только указанные поля
- `Omit<T, K>` — всё кроме указанных
- `Record<K, V>` — словарь с ключами K и значениями V
- `ReturnType<F>` — тип возвращаемого значения функции
- `Parameters<F>` — кортеж типов параметров
- `NonNullable<T>` — убирает null | undefined

---

**Q: Как типизировать DTO для создания и обновления?**

```ts
type CreateUserDTO = Omit<User, "id">;           // без id
type UpdateUserDTO = Partial<Omit<User, "id">>;  // без id, все поля опциональные
```

---

## Type Guards и Narrowing

**Q: Что такое Type Assertion (`as`) и почему это опасно?**

Ты говоришь компилятору "я знаю тип лучше". Не проверяется в рантайме — если ошибся, получишь runtime ошибку. Предпочитай type guards:

```ts
// Опасно
const el = document.getElementById("name") as HTMLInputElement;

// Безопаснее — type guard
function isInput(el: Element): el is HTMLInputElement {
    return el.tagName === "INPUT";
}
```

---

**Q: Что такое type guard (`is`)?**

Функция, которая уточняет тип для компилятора:

```ts
function isString(x: unknown): x is string {
    return typeof x === "string";
}
if (isString(value)) {
    value.toUpperCase();  // TS знает: string
}
```

---

## Enum

**Q: `enum` vs union of string literals — что лучше?**

Union предпочтительнее в большинстве случаев:
- Не генерирует JS код (стирается при компиляции)
- Проще и легковеснее
- Tree-shaking работает

```ts
// Предпочтительно
type Direction = "up" | "down" | "left" | "right";

// enum — когда нужен reverse mapping или runtime объект
enum Status { Active = "ACTIVE", Inactive = "INACTIVE" }
```

---

## Overloads

**Q: Что такое перегрузка функций (function overloads)?**

Несколько сигнатур для одной функции — разные типы входа дают разные типы выхода:

```ts
// Сигнатуры перегрузки (что видит вызывающий код)
function parse(input: string): object;
function parse(input: Buffer): object;
function parse(input: string, asArray: true): object[];

// Реализация (одна, обрабатывает все случаи)
function parse(input: string | Buffer, asArray?: boolean): object | object[] {
    const data = typeof input === "string" ? JSON.parse(input) : JSON.parse(input.toString());
    return asArray ? [data] : data;
}

parse("{}");           // → object
parse(buf);            // → object
parse("{}", true);     // → object[]
```

**Когда нужно:** когда тип возвращаемого значения зависит от типа/количества аргументов. Если можно обойтись union или generics — предпочитай их (проще).

---

## Продвинутое

**Q: Что такое Mapped Types?**

Типы, которые перебирают ключи другого типа и создают новый:

```ts
type Optional<T> = { [K in keyof T]?: T[K] };
type Nullable<T> = { [K in keyof T]: T[K] | null };
```

---

**Q: Что такое Conditional Types?**

Типы с условием `extends ? :`:

```ts
type IsString<T> = T extends string ? true : false;
type UnpackPromise<T> = T extends Promise<infer U> ? U : T;
```

`infer` — извлекает тип из структуры (аналог переменной в паттерне).

---

**Q: Что такое Template Literal Types?**

Строковые типы-шаблоны:

```ts
type EventName = "click" | "focus" | "blur";
type Handler = `on${Capitalize<EventName>}`;  // "onClick" | "onFocus" | "onBlur"
```

---

**Q: Что такое декораторы в TypeScript?**

Функции, которые модифицируют классы, методы, свойства на этапе определения. Экспериментальная фича (`experimentalDecorators: true`). Активно используются в NestJS, TypeORM. Работают через `Reflect.metadata`.

---

**Q: Какие ключевые опции `tsconfig.json`?**

- `strict: true` — включает все строгие проверки
- `strictNullChecks` — null/undefined — отдельные типы
- `noImplicitAny` — запрещает неявный any
- `target` — в какой JS компилировать (ES2022, ESNext)
- `module` — система модулей (commonjs, esnext)
- `esModuleInterop` — совместимость CJS/ESM импортов

---

**Q: Модификаторы доступа: `public` vs `private` vs `protected` vs `readonly`?**

- `public` — доступен везде (по умолчанию)
- `private` — только внутри класса
- `protected` — класс + наследники
- `readonly` — только чтение после инициализации

В отличие от Python — `private` реально ограничивает доступ на уровне компилятора (но не в рантайме).

---

## Практические задачи

**Q: Типизировать API-ответ с пагинацией:**

```ts
interface PaginatedResponse<T> {
    data: T[];
    meta: {
        total: number;
        page: number;
        perPage: number;
        lastPage: number;
    };
}

async function fetchUsers(): Promise<PaginatedResponse<User>> { ... }

const res = await fetchUsers();
res.data[0].name;  // TS знает: User
res.meta.total;    // number
```

---

**Q: Типизировать event map (строгие события):**

```ts
interface EventMap {
    "user:created": { id: number; email: string };
    "user:deleted": { id: number };
    "order:paid":   { orderId: string; amount: number };
}

function on<K extends keyof EventMap>(event: K, handler: (data: EventMap[K]) => void): void { ... }

on("user:created", (data) => {
    data.email;   // OK — TS знает тип payload
});
on("user:created", (data) => {
    data.orderId; // Ошибка! — нет такого поля у user:created
});
```

---

**Q: Типизировать builder-паттерн с цепочкой вызовов:**

```ts
class QueryBuilder<T> {
    private filters: Partial<T> = {};
    private sortField?: keyof T;

    where<K extends keyof T>(field: K, value: T[K]): this {
        this.filters[field] = value;
        return this;
    }

    orderBy(field: keyof T): this {
        this.sortField = field;
        return this;
    }

    build(): { filters: Partial<T>; sort?: keyof T } {
        return { filters: this.filters, sort: this.sortField };
    }
}

interface User { id: number; name: string; role: string }

new QueryBuilder<User>()
    .where("role", "admin")   // OK — "role" это keyof User, значение string
    .where("role", 42)        // Ошибка! — role ожидает string, не number
    .where("foo", "bar")      // Ошибка! — "foo" не keyof User
    .orderBy("name")
    .build();
```

---

**Q: Типизировать глубокий Partial (DeepPartial):**

```ts
type DeepPartial<T> = {
    [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

interface Config {
    db: { host: string; port: number; pool: { min: number; max: number } };
    redis: { url: string };
}

// Можно передать частичную конфигурацию любой глубины
function mergeConfig(override: DeepPartial<Config>): Config { ... }

mergeConfig({ db: { pool: { max: 20 } } });  // OK
```

---

**Q: Типизировать функцию, которая делает все поля required кроме указанных:**

```ts
type RequiredExcept<T, K extends keyof T> = Required<Omit<T, K>> & Partial<Pick<T, K>>;

interface Form {
    name?: string;
    email?: string;
    bio?: string;
    avatar?: string;
}

// name и email обязательны, остальные — опциональны
type SubmitForm = RequiredExcept<Form, "bio" | "avatar">;
// { name: string; email: string; bio?: string; avatar?: string }
```

---

**Q: Типизировать маппинг DTO → Entity (Pick + трансформация):**

```ts
// Entity из БД
interface UserEntity {
    id: number;
    email: string;
    passwordHash: string;
    createdAt: Date;
    updatedAt: Date;
}

// Только безопасные поля для API
type UserResponse = Pick<UserEntity, "id" | "email" | "createdAt">;

// Создание — без автогенерируемых полей
type CreateUser = Omit<UserEntity, "id" | "createdAt" | "updatedAt">;

// Обновление — частичное, без id
type UpdateUser = Partial<Omit<UserEntity, "id" | "createdAt" | "updatedAt">>;

function toResponse(entity: UserEntity): UserResponse {
    const { id, email, createdAt } = entity;
    return { id, email, createdAt };  // passwordHash не утечёт
}
```

---

**Q: Discriminated union для обработки разных типов сообщений:**

```ts
interface TextMessage {
    type: "text";
    content: string;
}

interface ImageMessage {
    type: "image";
    url: string;
    width: number;
    height: number;
}

interface FileMessage {
    type: "file";
    filename: string;
    size: number;
}

type Message = TextMessage | ImageMessage | FileMessage;

function render(msg: Message): string {
    switch (msg.type) {
        case "text":
            return msg.content;         // TS знает: TextMessage
        case "image":
            return `<img src="${msg.url}" width="${msg.width}" />`;  // ImageMessage
        case "file":
            return `📎 ${msg.filename} (${msg.size}b)`;  // FileMessage
        default:
            const _: never = msg;       // exhaustive check
            throw new Error(`Unknown message type`);
    }
}
```
