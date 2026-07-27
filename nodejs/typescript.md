# TypeScript

## Зачем TypeScript

- Статическая типизация → ошибки на этапе компиляции, а не в рантайме
- Лучшее автодополнение в IDE
- Документирует API функций и классов
- Рефакторинг без страха: компилятор найдёт все несовместимости

---

## Базовые типы

```ts
// Примитивы
let name: string = "Alice";
let age: number = 30;
let active: boolean = true;
let big: bigint = 9007199254740993n;

// Специальные
let x: undefined = undefined;
let y: null = null;
let z: any = "anything";      // отключает проверку типов
let w: unknown = fetchData();  // безопаснее any

// Массивы
let nums: number[] = [1, 2, 3];
let strs: Array<string> = ["a", "b"];

// Кортеж
let pair: [string, number] = ["Alice", 30];

// Объект
let user: { name: string; age: number } = { name: "Alice", age: 30 };
```

---

## any vs unknown

| | `any` | `unknown` |
|---|---|---|
| Присваивание | Куда угодно | Куда угодно |
| Использование | Без ограничений | Требует проверки типа |
| Безопасность | Нет | Да |

```ts
let a: any = "hello";
a.toUpperCase();  // OK (TypeScript доверяет)
a.nonExistent();  // OK (нет ошибки компиляции, возможна в рантайме)

let u: unknown = "hello";
u.toUpperCase();  // Ошибка! Тип unknown
if (typeof u === "string") {
    u.toUpperCase();  // OK — сузили тип
}
```

---

## void vs never

```ts
// void — функция ничего не возвращает (или возвращает undefined)
function log(msg: string): void {
    console.log(msg);
}

// never — функция никогда не возвращает управление
function throwError(msg: string): never {
    throw new Error(msg);
}

function infiniteLoop(): never {
    while (true) {}
}

// never используется для исчерпывающих проверок (exhaustive check)
type Shape = "circle" | "square";
function area(shape: Shape): number {
    switch (shape) {
        case "circle": return Math.PI;
        case "square": return 1;
        default:
            const _exhaustive: never = shape;  // ошибка если добавили новый тип без обработки
            throw new Error(`Unknown: ${_exhaustive}`);
    }
}
```

---

## interface vs type alias

```ts
// interface — для объектов и классов, можно расширять и мержить
interface User {
    name: string;
    age: number;
}

interface Admin extends User {
    role: string;
}

// Декларативное слияние (declaration merging) — уникально для interface
interface Window {
    myPlugin: () => void;
}

// type — для любых типов, union, intersection, mapped types
type ID = string | number;
type Point = { x: number; y: number };
type AdminUser = User & { role: string };   // intersection
```

**Когда что:**
- `interface` — когда описываешь форму объекта/класса, особенно в public API
- `type` — когда нужны union-типы, условные типы, mapped types

---

## interface vs abstract class

| | `interface` | `abstract class` |
|---|---|---|
| Компилируется в JS | Нет (стирается) | Да (класс) |
| Реализация методов | Нет | Можно (конкретные методы) |
| Несколько | `implements A, B` | `extends A` (только одно) |
| Поля | Только декларация | Может с инициализаторами |
| Конструктор | Нет | Может быть |

```ts
interface Serializable {
    serialize(): string;
}

abstract class BaseEntity {
    abstract getId(): number;  // должен реализовать наследник

    toJSON(): object {        // конкретная реализация
        return { id: this.getId() };
    }
}

class User extends BaseEntity implements Serializable {
    constructor(private id: number) { super(); }
    getId() { return this.id; }
    serialize() { return JSON.stringify(this.toJSON()); }
}
```

---

## Generics (Обобщённые типы)

```ts
// Функция
function identity<T>(value: T): T {
    return value;
}
identity<string>("hello");
identity(42);  // вывод типа: T = number

// Интерфейс
interface Repository<T> {
    findById(id: number): Promise<T>;
    save(entity: T): Promise<T>;
}

// Ограничения
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
    return obj[key];
}

// Conditional types
type NonNullable<T> = T extends null | undefined ? never : T;

// Generic класс
class Stack<T> {
    private items: T[] = [];
    push(item: T) { this.items.push(item); }
    pop(): T | undefined { return this.items.pop(); }
}
```

---

## Utility Types

```ts
interface User {
    id: number;
    name: string;
    email: string;
    age: number;
}

Partial<User>           // все поля опциональные
Required<User>          // все поля обязательные
Readonly<User>          // все поля readonly

Pick<User, "id" | "name">    // только id и name
Omit<User, "age">            // всё кроме age

Record<string, number>       // { [key: string]: number }
Record<"admin" | "user", User>  // { admin: User; user: User }

ReturnType<typeof fetchUser>    // тип возврата функции
Parameters<typeof fetchUser>    // кортеж параметров функции

NonNullable<string | null | undefined>  // string

// Пример использования
type CreateUserDTO = Omit<User, "id">;          // без id
type UpdateUserDTO = Partial<Omit<User, "id">>; // все поля опциональные, без id
```

---

## Type Assertion

```ts
// as — утверждение типа (ты говоришь компилятору "я знаю лучше")
const input = document.getElementById("name") as HTMLInputElement;
input.value;  // OK

// Двойное утверждение (опасно, используй с осторожностью)
const x = "hello" as unknown as number;

// Non-null assertion (!)
const el = document.getElementById("name")!;  // утверждаем что не null

// Предпочтительно — type guard
function isString(x: unknown): x is string {
    return typeof x === "string";
}
if (isString(x)) {
    x.toUpperCase();  // OK
}
```

---

## Enum

```ts
// Числовой enum (по умолчанию)
enum Direction {
    Up,     // 0
    Down,   // 1
    Left,   // 2
    Right   // 3
}
Direction.Up;           // 0
Direction[0];           // "Up" (reverse mapping)

// Строковый enum
enum Status {
    Active = "ACTIVE",
    Inactive = "INACTIVE",
    Pending = "PENDING"
}

// Const enum — инлайнится, нет JS объекта в рантайме
const enum Color { Red, Green, Blue }

// Альтернатива enum — union of string literals (часто предпочтительнее)
type Direction = "up" | "down" | "left" | "right";
```

---

## Модификаторы доступа

```ts
class BankAccount {
    public owner: string;         // доступен везде (по умолчанию)
    private balance: number;      // только внутри класса
    protected limit: number;      // класс + наследники
    readonly id: string;          // только чтение после инициализации

    constructor(owner: string, balance: number) {
        this.owner = owner;
        this.balance = balance;
        this.id = Math.random().toString();
    }

    // Краткая запись (shorthand)
    // constructor(public owner: string, private balance: number) {}

    getBalance() { return this.balance; }
}

class SavingsAccount extends BankAccount {
    setLimit(limit: number) {
        this.limit = limit;      // OK — protected
        // this.balance = 0;     // Ошибка — private
    }
}
```

---

## Mapped Types и Conditional Types

```ts
// Mapped type — перебираем ключи
type Optional<T> = {
    [K in keyof T]?: T[K];
};

type Nullable<T> = {
    [K in keyof T]: T[K] | null;
};

// Условные типы
type IsString<T> = T extends string ? true : false;
type IsStr1 = IsString<string>;   // true
type IsStr2 = IsString<number>;   // false

// Infer — извлечение типа
type UnpackPromise<T> = T extends Promise<infer U> ? U : T;
type Resolved = UnpackPromise<Promise<string>>;  // string

// Template literal types
type EventName = "click" | "focus" | "blur";
type Handler = `on${Capitalize<EventName>}`;  // "onClick" | "onFocus" | "onBlur"
```

---

## Декораторы (используются в NestJS, TypeORM)

```ts
// Декораторы — экспериментальная фича (нужен experimentalDecorators: true)

// Декоратор класса
function Injectable(): ClassDecorator {
    return (target) => {
        Reflect.defineMetadata("injectable", true, target);
    };
}

// Декоратор метода
function Log(): MethodDecorator {
    return (target, key, descriptor: PropertyDescriptor) => {
        const original = descriptor.value;
        descriptor.value = function(...args: any[]) {
            console.log(`Calling ${String(key)}`);
            return original.apply(this, args);
        };
    };
}

@Injectable()
class UserService {
    @Log()
    getUser(id: number) { ... }
}
```

---

## tsconfig.json — ключевые опции

```json
{
    "compilerOptions": {
        "target": "ES2022",           // в какой JS компилировать
        "module": "commonjs",          // система модулей
        "strict": true,                // включает все строгие проверки
        "noImplicitAny": true,         // запрещает неявный any
        "strictNullChecks": true,      // null/undefined — отдельные типы
        "esModuleInterop": true,       // совместимость CommonJS/ESM
        "experimentalDecorators": true, // декораторы (NestJS)
        "emitDecoratorMetadata": true,  // метаданные декораторов (reflect-metadata)
        "outDir": "./dist",
        "rootDir": "./src",
        "paths": {                     // алиасы импортов
            "@app/*": ["src/*"]
        }
    }
}
```
