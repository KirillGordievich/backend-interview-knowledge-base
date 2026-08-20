# JavaScript

## Какие типы данных есть в JavaScript

В JavaScript **7 примитивных** типов: `string`, `number`, `bigint`, `boolean`, `undefined`, `null`, `symbol`. Всё остальное — **объекты** (`object`), включая массивы, функции, даты, Map, Set и т.д.

Проверить тип значения можно оператором `typeof`:

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

**Как проверить что объект является массивом:** Встроенный метод `Array.isArray()`. Возвращает `true` для массивов и `false` для остальных данных. Оператор `typeof` не подходит — для массивов он возвращает `"object"`.

**Какие подвохи есть у `typeof`:**

`typeof` не всегда возвращает то, что ожидаешь. Самые известные случаи:

```js
typeof null          // "object" — баг языка с 1995 года, не исправлен
typeof NaN           // "number" — NaN формально число
typeof []            // "object" — массив это объект
typeof function(){}  // "function" — единственный объект с отдельным typeof
```

Если `typeof` недостаточно, для точной проверки можно использовать `Array.isArray()`, `instanceof` или `Object.prototype.toString.call()`.

---

## Разница между изменяемыми и неизменяемыми значениями

Все примитивы (`string`, `number`, `boolean` и т.д.) — **неизменяемые** (immutable). Когда ты "меняешь" строку, на самом деле создаётся новая строка, а старая остаётся в памяти до сборки мусора. Методы строк всегда возвращают новое значение, не модифицируя исходное.

Объекты и массивы — **изменяемые** (mutable). Их содержимое можно менять на месте, и все ссылки на этот объект увидят изменения.

```js
// Примитивы — неизменяемые
let str = "hello";
str.toUpperCase();    // возвращает "HELLO", но str всё ещё "hello"
str[0] = "H";        // молча проигнорируется (в strict mode — TypeError)
console.log(str);     // "hello"

// Объекты — изменяемые
const user = { name: "Alice" };
user.name = "Bob";    // изменили оригинальный объект
console.log(user);    // { name: "Bob" }

// Это важно при передаче в функции
function addAge(obj) {
    obj.age = 25;     // изменяет оригинал, а не копию!
}
addAge(user);
console.log(user);    // { name: "Bob", age: 25 }
```

Именно поэтому при работе с объектами важно понимать разницу между копированием по ссылке и по значению — об этом подробнее в секции про shallow и deep copy.

---

## Как передаются аргументы в функции — по значению или по ссылке

В JavaScript все аргументы передаются **по значению**. Но для объектов "значение" — это ссылка на объект в памяти. Поэтому поведение разное для примитивов и объектов:

- **Примитивы** — передаётся копия значения. Изменения внутри функции не влияют на оригинал.
- **Объекты** — передаётся копия ссылки. Функция может изменить содержимое объекта через эту ссылку, но не может заменить сам объект для вызывающего кода.

```js
// Примитив — копия значения
function double(x) {
    x = x * 2;   // меняем локальную копию
}
let num = 5;
double(num);
console.log(num);   // 5 — не изменился

// Объект — копия ссылки (можно менять содержимое)
function rename(user) {
    user.name = "Bob";   // меняем оригинальный объект через ссылку
}
const alice = { name: "Alice" };
rename(alice);
console.log(alice);   // { name: "Bob" } — изменился!

// Переприсвоение параметра — меняем только локальную переменную
function replace(user) {
    user = { name: "Charlie" };   // user теперь указывает на новый объект, но alice — на старый
}
replace(alice);
console.log(alice);   // { name: "Bob" } — не изменился
```

Технически это называют **"pass by sharing"** — передаётся копия ссылки, а не сама ссылка. Именно поэтому переприсвоение параметра внутри функции не меняет оригинальную переменную.

---

## Что такое NaN и как его проверить

`NaN` (Not a Number) — специальное значение, которое появляется при некорректных числовых операциях (например, `0/0` или `parseInt("abc")`). Главная особенность — `NaN` не равен сам себе:

```js
NaN === NaN        // false  ← единственное значение не равное себе
isNaN("hello")     // true   (приводит к числу сначала — ненадёжно)
Number.isNaN("hello")  // false  ← правильный способ, без приведения типов
Number.isNaN(NaN)      // true
```

Для проверки всегда используй `Number.isNaN()`, а не глобальный `isNaN()` — последний сначала приводит аргумент к числу, что даёт ложные срабатывания.

---

## В чём разница между `==` и `===`

`==` (нестрогое равенство) перед сравнением приводит типы к общему, что приводит к неочевидным результатам. `===` (строгое равенство) сравнивает без приведения типов — и значение, и тип должны совпадать.

```js
0 == false    // true   (false приводится к 0)
0 === false   // false  (разные типы)
"" == false   // true
null == undefined  // true  (специальное правило)
null === undefined // false
NaN == NaN    // false  (NaN не равен ничему)
```

На практике всегда используй `===`, чтобы избежать неожиданного приведения типов.

---

## Разница между null, undefined и undeclared

Три разных понятия, которые часто путают:

- **`null`** — явное "нет значения". Разработчик специально присвоил `null`, чтобы обозначить пустоту.
- **`undefined`** — переменная объявлена, но значение не присвоено. Также возвращается при обращении к несуществующему свойству объекта.
- **undeclared** — переменная вообще не объявлена. Обращение к ней вызовет `ReferenceError`.

| | null | undefined | undeclared |
|---|---|---|---|
| Что это | Явное "нет значения" | Объявлена, не присвоена | Не объявлена |
| typeof | `"object"` | `"undefined"` | `"undefined"` |

```js
let x;               // undefined
let y = null;        // null
console.log(z);      // ReferenceError (undeclared)
typeof z;            // "undefined" (не бросает ошибку — подвох!)
```

---

## Что такое hoisting (поднятие)

Hoisting — это механизм JavaScript, при котором объявления переменных и функций "поднимаются" в начало своей области видимости ещё до выполнения кода. Но поднимается только **объявление**, а не присваивание.

`var` поднимается и инициализируется как `undefined`. Объявления функций (`function declaration`) поднимаются целиком — их можно вызвать до объявления. А `let`/`const` поднимаются, но остаются в "мёртвой зоне" (TDZ) до строки объявления — обращение к ним раньше вызовет ошибку.

```js
console.log(x);     // undefined (не ReferenceError — var поднялся)
var x = 5;

foo();              // "hello" — работает, function declaration поднимается целиком
function foo() { console.log("hello"); }

bar();              // ReferenceError — const не доступен до объявления
const bar = () => {};

console.log(a);     // ReferenceError — let в TDZ
let a = 1;
```

---

## Чем отличаются Function Declaration, Expression и Arrow Function

В JavaScript есть три способа создать функцию, и у каждого свои особенности:

**Function Declaration** — классическое объявление. Поднимается (hoisting), поэтому можно вызвать до объявления в коде.

**Function Expression** — функция присваивается переменной. Не поднимается — вызов до объявления вызовет ошибку.

**Arrow Function** — краткий синтаксис, но с важными ограничениями: не имеет своего `this` (берёт из окружения, где создана), нет объекта `arguments`, нельзя использовать как конструктор (`new`), нет `prototype`.

```js
// Declaration — поднимается
function greet(name) {
    return `Hello, ${name}`;
}

// Expression — не поднимается
const greet = function(name) {
    return `Hello, ${name}`;
};

// Arrow function — краткий синтаксис, но без своего this
const greet = (name) => `Hello, ${name}`;
```

**Где используются стрелочные функции:** чаще всего как колбэки — в методах массивов (`map`, `filter`, `reduce`), в промисах (`.then`), в таймерах (`setTimeout`). Благодаря тому, что стрелочная функция не имеет своего `this`, она удобна внутри методов класса и объекта — не теряет контекст.

---

## В каких случаях используются анонимные функции

Анонимная функция — это функция без имени. Чаще всего они используются как **колбэки** (функции обратного вызова), которые передаются в другие функции и не нуждаются в имени:

```js
// Колбэки в методах массивов
users.filter(u => u.active);
orders.map(o => o.total);

// Колбэки в промисах и таймерах
fetch(url).then(res => res.json());
setTimeout(() => console.log("done"), 1000);

// IIFE — немедленно вызываемая функция
(function() {
    // изолированный scope
})();
```

Каждая стрелочная функция по своей природе анонимная — у неё нет имени. Но анонимной может быть и обычная функция (`function() {}`). Разница между ними не в анонимности, а в поведении:

- Стрелочная функция не имеет своего `this` — берёт его из окружающего контекста
- Стрелочную функцию нельзя вызвать через `new` (она не может быть конструктором)
- Обычная анонимная функция (`function() {}`) имеет свой `this` и может быть конструктором

---

## Что такое замыкание (closure)

Замыкание — это функция, которая запоминает переменные из своей области видимости даже после того, как внешняя функция завершилась. Это позволяет создавать приватные переменные и фабрики функций.

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

## Что такое каррирование (currying)

Каррирование — это трансформация функции с несколькими аргументами в цепочку функций, каждая из которых принимает один аргумент: `f(a, b, c)` превращается в `f(a)(b)(c)`. Это позволяет создавать специализированные функции на базе общей.

```js
const multiply = (a) => (b) => a * b;
const double = multiply(2);
double(5); // 10

// Практическое применение — создание специализированных функций
const log = (level) => (message) => console.log(`[${level}] ${message}`);
const error = log("ERROR");
error("disk full"); // [ERROR] disk full
```

**Partial application** — похожий приём, но вместо одного аргумента за раз фиксируется сразу часть аргументов. Например, через `bind`: `const addTax = add.bind(null, 0.2)`.

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

// Частые грабли — потеря this при передаче метода как callback:
const fn = obj.greet;
fn();                          // undefined — this потерян
setTimeout(obj.greet, 100);    // undefined — this потерян

// Решения:
setTimeout(() => obj.greet(), 100);           // обёртка в arrow
setTimeout(obj.greet.bind(obj), 100);         // bind

// call/apply/bind — явная установка this
function greet(greeting) {
    return `${greeting}, ${this.name}`;
}
greet.call({ name: "Alice" }, "Hello");     // "Hello, Alice"
greet.apply({ name: "Alice" }, ["Hello"]);  // "Hello, Alice"
const bound = greet.bind({ name: "Alice" });
bound("Hi");                                // "Hi, Alice"
```

**Arrow function и `this`:**

Arrow function **не имеет своего `this`** — берёт из окружения, где была создана.

```js
const obj = {
    name: "Alice",
    // Обычная — this = obj при вызове obj.greet()
    greet() { setTimeout(function() { console.log(this.name); }, 100); },
    // Arrow — this = obj, потому что arrow создана внутри greetFixed(), где this = obj
    greetFixed() { setTimeout(() => { console.log(this.name); }, 100); },
    // Arrow как метод — this = внешний scope (module/global), НЕ obj
    broken: () => { console.log(this.name); }
};
obj.greet();      // undefined — обычная функция в setTimeout теряет this
obj.greetFixed(); // "Alice" — arrow захватила this из greetFixed()
obj.broken();     // undefined — arrow взяла this из внешнего scope
```

Вывод: arrow function **хороша для callback'ов** (сохраняет `this`), но **не подходит как метод объекта**.

---

## Как копировать объекты: shallow vs deep copy

В JavaScript объекты передаются по ссылке, поэтому простое присваивание `const copy = obj` не создаёт копию — обе переменные указывают на один и тот же объект.

**Поверхностная (shallow) копия** — копирует только первый уровень свойств. Вложенные объекты по-прежнему ссылаются на оригинал. Создаётся через spread (`{...obj}`) или `Object.assign()`.

**Глубокая (deep) копия** — рекурсивно копирует все уровни вложенности. Современный способ — `structuredClone()`. Старый хак через `JSON.parse(JSON.stringify())` не работает с функциями, `Date`, `undefined` и циклическими ссылками.

```js
const obj = { a: 1, b: { c: 2 } };

// Shallow copy
const copy1 = { ...obj };
const copy2 = Object.assign({}, obj);
copy1.b.c = 99;      // obj.b.c тоже изменится — вложенный объект общий!

// Deep copy
const deep = structuredClone(obj);   // современный способ, копирует всё
const deep2 = JSON.parse(JSON.stringify(obj));  // хак, не работает с функциями и Date
```

---

## Object.freeze, seal и preventExtensions — как ограничить изменение объекта

JavaScript предоставляет три уровня "заморозки" объекта, каждый строже предыдущего:

- **`preventExtensions`** — запрещает добавлять новые свойства, но существующие можно менять и удалять.
- **`seal`** — запрещает добавление и удаление свойств, но значения существующих можно менять.
- **`freeze`** — полная заморозка: нельзя ни добавить, ни удалить, ни изменить свойства.

| | Добавить свойства | Удалить свойства | Изменить значения |
|---|---|---|---|
| `preventExtensions` | Нет | Да | Да |
| `seal` | Нет | Нет | Да |
| `freeze` | Нет | Нет | Нет |

Важный нюанс: все три метода работают **только на первом уровне** (shallow). Вложенные объекты не замораживаются:

```js
const obj = Object.freeze({ nested: { x: 1 } });
obj.nested.x = 2; // работает! freeze не затрагивает вложенные объекты
```

---

## Основные методы массивов

Методы массивов делятся на две группы: те, что **возвращают новый массив** (не мутируют оригинал), и те, что **изменяют оригинал**.

**Методы преобразования** (возвращают новый массив, оригинал не меняется):

```js
const nums = [1, 2, 3, 4, 5];

nums.map(x => x * 2)          // [2, 4, 6, 8, 10] — трансформация каждого элемента
nums.filter(x => x % 2 === 0) // [2, 4] — отбор по условию
nums.reduce((acc, x) => acc + x, 0)  // 15 — свёртка в одно значение
nums.flatMap(x => [x, x * 2]) // [1, 2, 2, 4, 3, 6, ...] — map + flatten
```

**Методы поиска:**

```js
nums.find(x => x > 3)         // 4 — первый подходящий элемент (или undefined)
nums.findIndex(x => x > 3)    // 3 — индекс первого подходящего
nums.some(x => x > 4)         // true — хотя бы один удовлетворяет
nums.every(x => x > 0)        // true — все удовлетворяют
nums.includes(3)               // true — есть ли элемент в массиве
```

**`forEach` vs `map`:** `forEach` не возвращает результат — используется только для побочных эффектов (логирование, запись в БД). `map` возвращает новый массив с результатами трансформации.

**Мутирующие методы** (изменяют оригинальный массив):

```js
nums.push(6)      // добавить в конец
nums.pop()        // удалить с конца
nums.shift()      // удалить с начала
nums.unshift(0)   // добавить в начало
nums.splice(1, 2) // удалить 2 элемента с индекса 1
nums.sort((a, b) => a - b)  // сортировка (без компаратора сортирует как строки!)
nums.reverse()    // разворот
```

---

## Деструктуризация

Деструктуризация позволяет извлекать значения из массивов и объектов в отдельные переменные. Это делает код короче и читаемее, особенно при работе с API-ответами и параметрами функций.

```js
// Из массива — по позиции, с rest-оператором для "остатка"
const [a, b, ...rest] = [1, 2, 3, 4, 5];
// a=1, b=2, rest=[3,4,5]

// Из объекта — по имени свойства, с дефолтным значением и вложенностью
const { name, age = 0, address: { city } } = user;

// В параметрах функции — удобно для конфигурационных объектов
function greet({ name, age = 18 }) {
    return `${name}, ${age}`;
}

// Swap двух переменных без временной
let x = 1, y = 2;
[x, y] = [y, x];
```

---

## Как работает Event Loop

JavaScript — однопоточный язык, и весь код выполняется в одном потоке. Асинхронность реализуется через Event Loop — механизм, который координирует выполнение синхронного кода, колбэков и очередей задач.

Event Loop работает с несколькими компонентами:

- **Call Stack** — стек текущего выполняемого кода (LIFO). Когда функция вызвана — она в стеке, завершилась — удалена.
- **Web APIs / Node APIs** — среда выполнения обрабатывает асинхронные операции (таймеры, HTTP, файлы) и по завершении кладёт колбэк в очередь.
- **Microtask Queue** — очередь для `Promise.then`, `queueMicrotask`. Имеет **приоритет** — опустошается полностью перед каждой макрозадачей.
- **Macrotask Queue** — очередь для `setTimeout`, `setInterval`, I/O, `setImmediate` (Node).

**Порядок выполнения:**
1. Выполнить весь синхронный код (Call Stack)
2. Выполнить **все** микрозадачи (Microtask Queue) — до полного опустошения
3. Взять **одну** макрозадачу из Macrotask Queue
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

## Promises — что это и как работают

Promise — это объект, представляющий результат асинхронной операции. У него три состояния:
- **`pending`** — операция ещё выполняется
- **`fulfilled`** — операция завершилась успешно (вызван `resolve`)
- **`rejected`** — операция завершилась с ошибкой (вызван `reject`)

После перехода из `pending` в `fulfilled` или `rejected` — состояние **неизменяемо** (settled). Повторный вызов `resolve`/`reject` просто игнорируется.

Напрямую узнать состояние промиса **нельзя** — нет свойства `.state` или `.status`. Можно только подписаться на результат через `.then`/`.catch`.

```js
// Создание промиса
const p = new Promise((resolve, reject) => {
    setTimeout(() => resolve("result"), 1000);
});

// Цепочка — каждый .then возвращает новый промис
p.then(result => result.toUpperCase())
 .then(upper => console.log(upper))
 .catch(err => console.error(err))
 .finally(() => console.log("done"));
```

**Статические методы для работы с несколькими промисами:**

```js
Promise.all([p1, p2, p3])          // ждёт всех, падает если хоть один rejected
Promise.allSettled([p1, p2, p3])   // ждёт всех, возвращает статус каждого (никогда не rejected)
Promise.race([p1, p2, p3])         // возвращает первый завершившийся (fulfilled или rejected)
Promise.any([p1, p2, p3])          // возвращает первый успешный (игнорирует rejected)
```

---

## async/await

`async`/`await` — синтаксический сахар над промисами, который позволяет писать асинхронный код как синхронный. Функция с `async` всегда возвращает промис, а `await` приостанавливает выполнение до завершения промиса.

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

## Обработка ошибок

Ошибки в синхронном коде обрабатываются через `try/catch/finally`. Блок `finally` выполняется всегда — даже если в `try` или `catch` есть `return`.

```js
try {
    const data = JSON.parse(input);
} catch (err) {
    if (err instanceof SyntaxError) {
        console.error("Invalid JSON:", err.message);
    } else {
        throw err;  // перебросить неизвестную ошибку
    }
} finally {
    cleanup();  // выполнится в любом случае
}
```

**Ошибки в асинхронном коде** обрабатываются иначе — `try/catch` не поймает ошибку внутри колбэка:
- В цепочке промисов — `.catch()` в конце: `fetch(url).then(handle).catch(err => ...)`
- С `async/await` — оборачиваем в `try/catch`, это предпочтительный способ
- `Promise.allSettled()` — не прерывается при ошибках, возвращает результат каждого промиса

Если промис rejected и нигде не обработан — в Node.js это генерирует событие `unhandledRejection` и завершает процесс.

---

## Модули: ESM vs CommonJS

В JavaScript две системы модулей:

**ESM (ES Modules)** — стандарт языка. Импорты статические — анализируются до выполнения кода, что позволяет бандлерам делать tree-shaking (удаление неиспользуемого кода). Поддерживает top-level `await`.

**CommonJS** — исторический формат Node.js. Импорты динамические — `require()` выполняется в runtime и может быть условным. Синхронный.

```js
// ESM
import { foo } from "./module.js";
export const bar = 42;

// CommonJS
const foo = require("./module");
module.exports = { bar: 42 };
```

**Что такое tree-shaking:** при сборке бандлер (Webpack, Rollup) удаляет код, который нигде не импортируется. Это работает только с ESM, потому что `import`/`export` статические и бандлер точно знает, что используется. С `require()` это невозможно — вызов динамический и бандлер не может определить, какие экспорты нужны.

---

## setTimeout и setInterval

`setTimeout` выполняет функцию **один раз** через указанное количество миллисекунд. `setInterval` выполняет функцию **повторно** с заданным интервалом. Оба возвращают ID таймера для отмены.

```js
const timerId = setTimeout(() => console.log("hello"), 1000);
clearTimeout(timerId);  // отменить до выполнения

const intervalId = setInterval(() => console.log("tick"), 500);
clearInterval(intervalId);  // остановить повторение
```

Важный нюанс: `setTimeout(fn, 0)` не выполняет функцию немедленно — он откладывает её на следующую итерацию event loop, после всех микрозадач (промисов, `nextTick`).

---

## Прототипное наследование

В JavaScript наследование реализовано через цепочку прототипов. Каждый объект имеет скрытую ссылку `[[Prototype]]` на другой объект (его прототип). Когда обращаемся к свойству, которого нет у объекта — JavaScript ищет его вверх по цепочке прототипов.

Классы (`class`) — это синтаксический сахар над прототипами. Под капотом всё работает через `prototype`.

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

## Разница между var, let и const

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

## Map, Set, WeakMap, WeakSet

### Map — словарь с любыми типами ключей

`Map` — коллекция пар ключ-значение, похожая на обычный объект, но с важными отличиями: ключом может быть **любое значение** (объект, функция, число), сохраняется порядок вставки, и есть свойство `.size`.

Когда использовать `Map` вместо объекта: когда ключи не строки, когда важен порядок, или когда часто добавляются и удаляются записи.

```js
const map = new Map();
map.set("key", "value");
map.set({id: 1}, "obj-value");  // объект как ключ — в обычном объекте так нельзя
map.get("key");        // "value"
map.has("key");        // true
map.size;              // 2
map.delete("key");

for (const [key, value] of map) { ... }
```

### Set — коллекция уникальных значений

`Set` хранит только уникальные значения — дубликаты автоматически отбрасываются. Часто используется для удаления дубликатов из массива и для быстрой проверки наличия элемента (O(1) вместо O(n) у массива).

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

### WeakMap и WeakSet — коллекции со слабыми ссылками

`WeakMap` и `WeakSet` похожи на `Map` и `Set`, но хранят **слабые ссылки** на ключи (объекты). Если на объект-ключ больше нигде нет ссылок — сборщик мусора его удалит, а запись из WeakMap/WeakSet исчезнет автоматически. Это предотвращает утечки памяти.

Ограничения: нельзя итерировать (нет `forEach`, `for...of`), нет `.size` — потому что содержимое может измениться в любой момент из-за GC.

```js
// WeakMap: ключи — только объекты, не препятствует сборке мусора
const cache = new WeakMap();

function process(obj) {
    if (cache.has(obj)) return cache.get(obj);
    const result = expensiveComputation(obj);
    cache.set(obj, result);   // если obj удалится → запись тоже удалится
    return result;
}

// Применение: хранение метаданных без утечки памяти
// Нельзя итерировать — нет .size, нет forEach

// WeakSet — аналогично, но множество объектов
const visited = new WeakSet();
visited.add(node);       // пометить как посещённый
visited.has(node);       // проверить
// Применение: трекинг обработанных DOM-нод, защита от циклов
```

**Map vs WeakMap — ключевые отличия:**

| | Map | WeakMap |
|---|---|---|
| Ключи | любые значения | только объекты / символы |
| GC | ключи удерживаются (strong ref) | ключи **не удерживаются** (weak ref) → GC собирает |
| Итерация | `for..of`, `.keys()`, `.entries()` | **нельзя** итерировать |
| `.size` | есть | нет |
| Утечки памяти | возможны, если забыть `.delete()` | невозможны — GC сам чистит |

```js
// Проблема Map — утечка памяти:
const map = new Map();
(function() {
    const obj = { heavy: new Array(1_000_000) };
    map.set(obj, "data");
    // obj выходит из scope, но Map держит strong ref → НЕ собирается GC
})();
// map.size === 1, объект жив, память занята

// Решение WeakMap — автоочистка:
const weak = new WeakMap();
(function() {
    const obj = { heavy: new Array(1_000_000) };
    weak.set(obj, "data");
    // obj выходит из scope → WeakMap НЕ держит → GC собирает
})();
// объект удалён, память свободна
```

---

## Spread и Rest операторы

Одинаковый синтаксис `...`, но противоположные по смыслу:

**Spread** (`...`) — разворачивает массив или объект на отдельные элементы. Используется для копирования, объединения и передачи аргументов.

**Rest** (`...`) — собирает оставшиеся элементы в массив или объект. Используется в параметрах функций и деструктуризации.

```js
// Spread — разворачивает
const arr = [1, 2, 3];
const arr2 = [...arr, 4, 5];         // [1, 2, 3, 4, 5]
const obj2 = { ...obj, extra: true }; // shallow-копия + новое свойство
Math.max(...arr);  // 3

// Rest — собирает оставшиеся аргументы в массив
function sum(...nums) {
    return nums.reduce((a, b) => a + b, 0);
}
sum(1, 2, 3, 4);   // 10

// Rest в деструктуризации — забирает "остаток"
const [head, ...tail] = [1, 2, 3, 4];   // head=1, tail=[2,3,4]
const { a, ...others } = { a: 1, b: 2, c: 3 };  // a=1, others={b:2,c:3}
```

---

## Optional Chaining `?.` и Nullish Coalescing `??`

**Optional chaining** (`?.`) позволяет безопасно обращаться к вложенным свойствам без проверки каждого уровня на `null`/`undefined`. Если на каком-то уровне значение отсутствует, вернётся `undefined` вместо ошибки.

**Nullish coalescing** (`??`) возвращает правый операнд только когда левый — `null` или `undefined`. В отличие от `||`, он **не заменяет** `0`, `""` и `false` — это важно для значений, где ноль или пустая строка допустимы.

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

## Что такое Symbol и зачем он нужен

`Symbol` — примитивный тип данных, каждое значение которого **гарантированно уникально**. Даже два символа с одинаковым описанием не равны друг другу. Основное применение — создание уникальных ключей для свойств объекта, которые не конфликтуют с другими ключами и не видны в обычной итерации (`for...in`, `Object.keys`).

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

## Генераторы (Generators)

Генератор — это особая функция, которая может **приостанавливать** своё выполнение на `yield` и **возобновлять** его при вызове `.next()`. В отличие от обычной функции, генератор не выполняется целиком за один раз — он выдаёт значения по одному. Это полезно для ленивых вычислений и бесконечных последовательностей.

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

---

## Задачи "Что выведет?"

Классические задачи с собеседований на понимание Event Loop, замыканий и приведения типов. В каждой нужно определить порядок или результат вывода.

```js
console.log(1);
setTimeout(() => console.log(2), 0);
Promise.resolve().then(() => console.log(3));
console.log(4);
// 1, 4, 3, 2 — синхронный код → микрозадача Promise → макрозадача setTimeout
```

```js
const a = {};
const b = { key: "b" };
const c = { key: "c" };
a[b] = 123;
a[c] = 456;
console.log(a[b]);
// 456 — объекты как ключи приводятся к "[object Object]", второй перезаписал первый
```

```js
let x = 1;
const fn = () => console.log(x);
{
    let x = 2;
    fn();
}
// 1 — arrow function замкнулась на x из scope где создана, блочный let x = 2 — другая переменная
```

```js
console.log(typeof typeof 1);
// "string" — typeof 1 → "number" (строка), typeof "number" → "string"
```

```js
const arr = [1, 2, 3];
arr[10] = 11;
console.log(arr.length);           // 11
console.log(arr.filter(Boolean).length);  // 4
// arr[10] расширяет массив до длины 11, индексы 3–9 — пустые слоты. filter пропускает их.
```
