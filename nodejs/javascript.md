# JavaScript

## Типы данных

**Примитивы (7):** `string`, `number`, `bigint`, `boolean`, `undefined`, `null`, `symbol`

**Ссылочные:** `object` (включая `Array`, `Function`, `Date`, `Map`, `Set`, ...)

```js
typeof "hello"     // "string"
typeof 42          // "number"
typeof true        // "boolean"
typeof undefined   // "undefined"
typeof null        // "object"  ← историческая ошибка
typeof {}          // "object"
typeof []          // "object"
typeof function(){} // "function"
typeof Symbol()    // "symbol"
typeof 42n         // "bigint"
```

**Проверка массива:** `Array.isArray([])` → `true`

---

## NaN

`NaN` (Not a Number) — результат некорректной числовой операции.

```js
NaN === NaN        // false  ← единственное значение не равное себе
isNaN("hello")     // true   (приводит к числу сначала)
Number.isNaN("hello")  // false  ← правильный способ
Number.isNaN(NaN)      // true
```

---

## == vs ===

`==` — нестрогое равенство (приводит типы), `===` — строгое (без приведения).

```js
0 == false    // true   (false → 0)
0 === false   // false
"" == false   // true
null == undefined  // true
null === undefined // false
NaN == NaN    // false
```

Всегда используй `===`.

---

## null / undefined / undeclared

| | null | undefined | undeclared |
|---|---|---|---|
| Что это | Явное "нет значения" | Переменная объявлена, не присвоена | Переменная не объявлена |
| typeof | `"object"` | `"undefined"` | `"undefined"` |

```js
let x;               // undefined
let y = null;        // null
console.log(z);      // ReferenceError (undeclared)
typeof z;            // "undefined" (не бросает ошибку!)
```

---

## Hoisting (Поднятие)

`var` и объявления функций поднимаются в начало области видимости.

```js
console.log(x);     // undefined (не ReferenceError!)
var x = 5;

foo();              // "hello" — работает!
function foo() { console.log("hello"); }

bar();              // ReferenceError
const bar = () => {};

// let/const — поднимаются, но не инициализируются (TDZ — Temporal Dead Zone)
console.log(a);     // ReferenceError
let a = 1;
```

---

## Function Declaration vs Expression

```js
// Declaration — поднимается (hoisting)
function greet(name) {
    return `Hello, ${name}`;
}

// Expression — не поднимается
const greet = function(name) {
    return `Hello, ${name}`;
};

// Arrow function — нет своего this, нет arguments
const greet = (name) => `Hello, ${name}`;
```

**Arrow function отличия:**
- Нет своего `this` — берёт из лексического окружения
- Нет `arguments` объекта
- Нельзя использовать как конструктор (`new`)
- Нет `prototype`

---

## Замыкания (Closures)

Функция, которая помнит переменные из своей области видимости даже после завершения внешней функции.

```js
function makeCounter() {
    let count = 0;                // "закрытая" переменная
    return {
        increment() { count++; },
        get() { return count; }
    };
}

const counter = makeCounter();
counter.increment();
counter.increment();
counter.get();  // 2

// Типичный вопрос на собеседовании:
for (var i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 0);
}
// Выведет: 3 3 3 (var — одна переменная на все итерации)

for (let i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 0);
}
// Выведет: 0 1 2 (let — отдельная переменная для каждой итерации)
```

---

## this

Значение `this` зависит от **контекста вызова**, а не от места объявления.

```js
const obj = {
    name: "Alice",
    greet() {
        console.log(this.name);   // "Alice" — this = obj
    },
    greetArrow: () => {
        console.log(this.name);   // undefined — this из лексики (module/глобальный)
    }
};

// call/apply/bind — явная установка this
function greet(greeting) {
    return `${greeting}, ${this.name}`;
}
greet.call({ name: "Alice" }, "Hello");     // "Hello, Alice"
greet.apply({ name: "Alice" }, ["Hello"]);  // "Hello, Alice"
const bound = greet.bind({ name: "Alice" });
bound("Hi");                                // "Hi, Alice"
```

**Правила this (по приоритету):**
1. `new` — новый объект
2. `bind/call/apply` — явный контекст
3. Метод объекта — сам объект
4. Обычный вызов — `undefined` (strict) или `window` (non-strict)
5. Arrow — лексический `this`

---

## Копирование объектов

```js
const obj = { a: 1, b: { c: 2 } };

// Поверхностная (shallow) копия
const copy1 = { ...obj };
const copy2 = Object.assign({}, obj);

copy1.b.c = 99;      // obj.b.c тоже изменится!

// Глубокая (deep) копия
const deep = structuredClone(obj);   // современный способ
const deep2 = JSON.parse(JSON.stringify(obj));  // не работает с функциями, Date, undefined
```

---

## Array методы

```js
const nums = [1, 2, 3, 4, 5];

// Преобразование (возвращает новый массив)
nums.map(x => x * 2)          // [2, 4, 6, 8, 10]
nums.filter(x => x % 2 === 0) // [2, 4]
nums.reduce((acc, x) => acc + x, 0)  // 15
nums.flatMap(x => [x, x * 2]) // [1, 2, 2, 4, 3, 6, ...]

// Поиск
nums.find(x => x > 3)         // 4
nums.findIndex(x => x > 3)    // 3
nums.some(x => x > 4)         // true
nums.every(x => x > 0)        // true
nums.includes(3)               // true

// forEach vs map:
// forEach — только для побочных эффектов, возвращает undefined
// map — возвращает новый массив

// Изменяют оригинал:
nums.push(6)      // добавить в конец
nums.pop()        // удалить с конца
nums.shift()      // удалить с начала
nums.unshift(0)   // добавить в начало
nums.splice(1, 2) // удалить 2 элемента с индекса 1
nums.sort((a, b) => a - b)  // сортировка
nums.reverse()    // разворот
```

---

## Деструктуризация

```js
// Массивы
const [a, b, ...rest] = [1, 2, 3, 4, 5];
// a=1, b=2, rest=[3,4,5]

// Объекты
const { name, age = 0, address: { city } } = user;

// В параметрах функции
function greet({ name, age = 18 }) {
    return `${name}, ${age}`;
}

// Swap
let x = 1, y = 2;
[x, y] = [y, x];
```

---

## Event Loop

JavaScript — однопоточный. Асинхронность через Event Loop.

**Компоненты:**

```
Call Stack          — текущий выполняемый код (LIFO)
Web APIs / Node APIs — setTimeout, fetch, fs, etc.
Macrotask Queue     — setTimeout, setInterval, I/O, setImmediate (Node)
Microtask Queue     — Promise.then, queueMicrotask, MutationObserver
```

**Порядок выполнения:**
1. Выполнить весь синхронный код (Call Stack)
2. Выполнить все микрозадачи (Microtask Queue) — до полного опустошения
3. Взять одну макрозадачу из Macrotask Queue
4. Снова выполнить все микрозадачи
5. Повторить

```js
console.log("1");

setTimeout(() => console.log("2"), 0);   // macrotask

Promise.resolve().then(() => console.log("3"));  // microtask

console.log("4");

// Порядок: 1, 4, 3, 2
```

```js
// Сложный пример
console.log("start");

setTimeout(() => {
    console.log("timeout");
    Promise.resolve().then(() => console.log("promise inside timeout"));
}, 0);

Promise.resolve()
    .then(() => console.log("promise 1"))
    .then(() => console.log("promise 2"));

console.log("end");

// start → end → promise 1 → promise 2 → timeout → promise inside timeout
```

---

## Promises

```js
// Создание
const p = new Promise((resolve, reject) => {
    // async operation
    setTimeout(() => resolve("result"), 1000);
});

// Цепочка
p.then(result => result.toUpperCase())
 .then(upper => console.log(upper))
 .catch(err => console.error(err))
 .finally(() => console.log("done"));

// Параллельно
Promise.all([p1, p2, p3])          // ждёт всех, падает если хоть один упал
Promise.allSettled([p1, p2, p3])   // ждёт всех, возвращает статус каждого
Promise.race([p1, p2, p3])         // возвращает первый завершившийся
Promise.any([p1, p2, p3])          // возвращает первый успешный
```

---

## async/await

Синтаксический сахар над Promises.

```js
async function fetchUser(id) {
    try {
        const response = await fetch(`/api/users/${id}`);
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        return await response.json();
    } catch (err) {
        console.error("Failed:", err);
        throw err;
    }
}

// Параллельные запросы
async function fetchAll() {
    const [user, posts] = await Promise.all([
        fetchUser(1),
        fetchPosts(1)
    ]);
    return { user, posts };
}

// await в цикле — последовательно (часто не то что нужно)
for (const id of ids) {
    await fetchUser(id);   // каждый ждёт предыдущего!
}

// Параллельно через map
const users = await Promise.all(ids.map(id => fetchUser(id)));
```

---

## setTimeout / setInterval

```js
// Один раз через N мс
const timerId = setTimeout(() => console.log("hello"), 1000);
clearTimeout(timerId);  // отменить

// Каждые N мс
const intervalId = setInterval(() => console.log("tick"), 500);
clearInterval(intervalId);  // остановить

// setTimeout с 0 — отложить на следующую итерацию event loop (после микрозадач!)
setTimeout(() => console.log("after sync"), 0);
```

---

## Prototype и прототипное наследование

```js
function Animal(name) {
    this.name = name;
}
Animal.prototype.speak = function() {
    return `${this.name} makes a sound`;
};

class Dog extends Animal {
    speak() {
        return `${this.name} barks`;
    }
}

const dog = new Dog("Rex");
dog.speak();            // "Rex barks"
dog instanceof Dog;     // true
dog instanceof Animal;  // true

// Цепочка прототипов:
// dog → Dog.prototype → Animal.prototype → Object.prototype → null
Object.getPrototypeOf(dog) === Dog.prototype;  // true
```

---

## var / let / const — области видимости

| | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function | Block | Block |
| Hoisting | Да (→ `undefined`) | Да (TDZ) | Да (TDZ) |
| Повторное объявление | Да | Нет | Нет |
| Переприсваивание | Да | Да | Нет |

```js
function example() {
    var x = 1;
    if (true) {
        var x = 2;   // та же переменная!
        let y = 3;   // только в блоке if
        const z = 4; // только в блоке if
    }
    console.log(x); // 2 (var — function scope)
    // console.log(y); // ReferenceError
}
```

---

## Map / Set / WeakMap / WeakRef

### Map — словарь с любыми ключами

```js
const map = new Map();
map.set("key", "value");
map.set({id: 1}, "obj-value");  // объект как ключ — в отличие от обычного объекта
map.get("key");        // "value"
map.has("key");        // true
map.size;              // 2
map.delete("key");

// Итерация
for (const [key, value] of map) { ... }
map.forEach((value, key) => { ... });

// Object vs Map:
// Map — любые ключи, сохраняет порядок вставки, быстрее при частых add/delete
// Object — только string/symbol ключи, прототип загрязняет ключи
```

### Set — коллекция уникальных значений

```js
const set = new Set([1, 2, 2, 3, 3]);   // {1, 2, 3}
set.add(4);
set.has(2);    // true
set.delete(2);
set.size;      // 3

// Удаление дубликатов из массива
const unique = [...new Set([1, 2, 2, 3])];   // [1, 2, 3]

// Пересечение/объединение множеств
const a = new Set([1, 2, 3]);
const b = new Set([2, 3, 4]);
const union        = new Set([...a, ...b]);           // {1,2,3,4}
const intersection = new Set([...a].filter(x => b.has(x))); // {2,3}
```

### WeakMap / WeakSet — слабые ссылки

```js
// WeakMap: ключи — только объекты, не препятствует GC
const cache = new WeakMap();

function process(obj) {
    if (cache.has(obj)) return cache.get(obj);
    const result = expensiveComputation(obj);
    cache.set(obj, result);   // если obj удалится → запись тоже удалится
    return result;
}

// Применение: хранение метаданных без утечки памяти
// Нельзя итерировать — нет .size, нет forEach
```

---

## Spread / Rest

```js
// Spread — разворачивает итерируемое
const arr = [1, 2, 3];
const arr2 = [...arr, 4, 5];         // [1, 2, 3, 4, 5]
const obj2 = { ...obj, extra: true }; // клонировать + добавить

Math.max(...arr);  // 3

// Rest — собирает оставшиеся аргументы
function sum(...nums) {
    return nums.reduce((a, b) => a + b, 0);
}
sum(1, 2, 3, 4);   // 10

function first(a, b, ...rest) {
    console.log(a, b, rest);   // 1, 2, [3, 4, 5]
}
first(1, 2, 3, 4, 5);

// Rest в деструктуризации
const [head, ...tail] = [1, 2, 3, 4];   // head=1, tail=[2,3,4]
const { a, ...others } = { a: 1, b: 2, c: 3 };  // a=1, others={b:2,c:3}
```

---

## Optional Chaining `?.` и Nullish Coalescing `??`

```js
const user = { profile: { address: { city: "Moscow" } } };

// Без optional chaining — пробрасывать каждый уровень
const city = user && user.profile && user.profile.address && user.profile.address.city;

// С optional chaining — короче и безопаснее
const city = user?.profile?.address?.city;   // "Moscow" или undefined
const name = user?.getName?.();              // вызов метода, если существует
const first = arr?.[0];                      // индекс массива

// ?? — nullish coalescing: возвращает правый операнд только при null/undefined
const port = process.env.PORT ?? 3000;       // 0 или "" НЕ заменяются!

// vs || — заменяет при любом falsy (0, "", false тоже)
const port2 = process.env.PORT || 3000;      // "" → 3000 (нежелательно для порта)

// ??= — присвоить если null/undefined
let config = null;
config ??= { debug: false };   // { debug: false }

// ?. + ?? вместе
const timeout = config?.timeout ?? 5000;
```

---

## Symbol

Уникальный примитив. Два символа всегда разные, даже с одинаковым описанием.

```js
const id = Symbol("id");
const id2 = Symbol("id");
id === id2;   // false

// Используется как уникальный ключ объекта (не видно в for...in и Object.keys)
const user = {
    [Symbol("id")]: 123,
    name: "Alice"
};
Object.keys(user);   // ["name"]   — Symbol не включается

// Symbol.iterator — делает объект итерируемым
class Range {
    constructor(start, end) {
        this.start = start;
        this.end = end;
    }

    [Symbol.iterator]() {
        let current = this.start;
        const end = this.end;
        return {
            next() {
                if (current <= end) {
                    return { value: current++, done: false };
                }
                return { value: undefined, done: true };
            }
        };
    }
}

for (const n of new Range(1, 5)) {
    console.log(n);   // 1 2 3 4 5
}
[...new Range(1, 3)];  // [1, 2, 3]

// Глобальные символы
const s1 = Symbol.for("shared");
const s2 = Symbol.for("shared");
s1 === s2;   // true  (из глобального реестра)
```

---

## Generators

Функция, которая может **приостанавливать** выполнение и **возобновлять** его.

```js
function* counter(start = 0) {
    while (true) {
        const reset = yield start;  // yield возвращает значение, ждёт next()
        if (reset) {
            start = 0;
        } else {
            start++;
        }
    }
}

const gen = counter(5);
gen.next();        // { value: 5, done: false }
gen.next();        // { value: 6, done: false }
gen.next(true);    // { value: 0, done: false }  ← передали reset=true

// Конечный генератор
function* fibonacci() {
    let [a, b] = [0, 1];
    while (true) {
        yield a;
        [a, b] = [b, a + b];
    }
}

const fib = fibonacci();
Array.from({ length: 8 }, () => fib.next().value);  // [0,1,1,2,3,5,8,13]

// yield* — делегирование другому генератору
function* concat(...iterables) {
    for (const iterable of iterables) {
        yield* iterable;
    }
}
[...concat([1,2], [3,4], [5])];  // [1, 2, 3, 4, 5]

// Генераторы vs async/await:
// async/await — синтаксический сахар над генераторами + промисами
// Генераторы — ленивые вычисления, бесконечные последовательности
```
