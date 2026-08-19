# NestJS: Вопросы

> Теория: [nestjs.md](nestjs.md)

---

## Архитектура

**Q: Что такое NestJS и чем отличается от Express?**

Фреймворк на TypeScript с модульной архитектурой, DI из коробки, декораторами. Express — минималистичный, без структуры. NestJS даёт: организацию кода (модули, контроллеры, сервисы), встроенную валидацию, Guards/Interceptors/Pipes, поддержку микросервисов.

---

**Q: Опиши типичную структуру NestJS приложения.**

Слоистая архитектура — каждый слой отвечает за своё:

```
Controller       — принимает HTTP-запрос, извлекает параметры, возвращает ответ
    ↓
Service          — бизнес-логика, оркестрация
    ↓
Repository/ORM   — работа с БД (TypeORM, Prisma, MikroORM)
    ↓
Database         — PostgreSQL, MongoDB, ...
```

Controller не знает про БД. Service не знает про HTTP. Repository не знает про бизнес-правила. Каждый слой зависит только от слоя ниже — легко тестировать и заменять.

---

**Q: Из чего состоит модуль в NestJS?**

```
@Module({
    imports: [],      // другие модули, чьи providers нужны
    controllers: [],  // HTTP-обработчики
    providers: [],    // сервисы, guards, interceptors, etc.
    exports: [],      // что доступно другим модулям
})
```

Модуль инкапсулирует функциональность. Зависимости между модулями — через `imports/exports`.

---

**Q: Controller vs Service — какие обязанности?**

- **Controller** — принимает HTTP-запросы, извлекает параметры, вызывает сервис, возвращает ответ. Не содержит бизнес-логику.
- **Service** — бизнес-логика, работа с БД, валидация. Переиспользуется между контроллерами.

---

## Dependency Injection

**Q: Зачем нужен Dependency Injection? Почему не создавать зависимости напрямую?**

Без DI — жёсткая связность: `new UsersService(new Repository(new Database()))` в каждом месте. Проблемы:
- **Тестирование** — нельзя подменить зависимость на мок
- **Связность** — изменил конструктор Repository → правь везде где создаёшь
- **Переиспользование** — нельзя легко заменить реализацию (Redis cache → Memory cache)
- **Lifecycle** — кто управляет созданием и уничтожением?

DI решает всё это: фреймворк создаёт зависимости, управляет их жизненным циклом, позволяет подменять реализации через провайдеры.

---

**Q: Как работает DI в NestJS?**

IoC-контейнер: ты объявляешь зависимости через конструктор с `@Injectable()`, фреймворк автоматически создаёт и внедряет экземпляры. По умолчанию — Singleton (один экземпляр на всё приложение).

```ts
@Injectable()
class AuthService {
    constructor(
        private usersService: UsersService,  // NestJS создаст и инъектирует
        private jwtService: JwtService,
    ) {}
}
```

---

**Q: Какие виды кастомных провайдеров есть?**

- `useValue` — конкретное значение (конфиг, константа)
- `useFactory` — создаёт провайдер через фабрику (с зависимостями)
- `useClass` — подменяет реализацию (полезно для тестирования)

```ts
{ provide: 'CONFIG', useValue: { dbUrl: '...' } }
{ provide: 'CACHE', useFactory: (config) => new Redis(config.get('REDIS_URL')), inject: [ConfigService] }
```

---

**Q: Какие Scopes провайдеров существуют?**

- `DEFAULT` (Singleton) — один на всё приложение
- `REQUEST` — новый экземпляр на каждый HTTP-запрос
- `TRANSIENT` — новый экземпляр при каждой инъекции

**Важно:** REQUEST scope поднимается по дереву зависимостей — все зависящие провайдеры тоже станут REQUEST scope.

---

## Request Pipeline

**Q: В каком порядке выполняются компоненты при обработке запроса?**

```
Request → Middleware → Guards → Interceptors (before) → Pipes → Handler → Interceptors (after) → Exception Filters → Response
```

---

**Q: Что такое Pipe и для чего?**

Pipe обрабатывает входные данные перед хендлером: валидация и трансформация.

- `ValidationPipe` — валидирует DTO через class-validator
- `ParseIntPipe` — конвертирует string → number
- Кастомный Pipe — любая логика

```ts
app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
```

---

**Q: Что такое Middleware в NestJS и когда использовать?**

Функция, которая выполняется **до** всего остального (Guards, Interceptors, Pipes). Имеет доступ к `req`, `res`, `next()` — как в Express. Не знает какой handler будет вызван.

Когда использовать: логирование запросов, CORS, парсинг cookies, rate limiting — всё что не зависит от конкретного роута.

```ts
@Injectable()
export class LoggerMiddleware implements NestMiddleware {
    use(req: Request, res: Response, next: NextFunction) {
        console.log(`${req.method} ${req.url}`);
        next();
    }
}
```

---

**Q: Чем Guard отличается от Middleware?**

| | Middleware | Guard |
|---|---|---|
| Когда | Самый первый в pipeline | После Middleware |
| Контекст | Только `req`/`res` | `ExecutionContext` — знает какой handler вызовется |
| Назначение | Общая обработка (логи, CORS) | Авторизация, проверка прав |
| Доступ к метаданным | Нет | Да (`Reflector` → `@Roles()`, `@Public()`) |

Guard знает о NestJS-контексте (контроллер, метод, декораторы). Middleware — нет.

---

**Q: Что такое Interceptor и для чего?**

Перехватывает запрос и/или ответ. Использования:
- Логирование (замер времени)
- Трансформация ответа (обернуть в `{ data: ... }`)
- Кэширование
- Обработка ошибок

Работает через RxJS Observable — можно использовать `map`, `tap`, `catchError`.

---

**Q: Чем Pipe отличается от Interceptor?**

| | Pipe | Interceptor |
|---|---|---|
| Когда | Перед handler'ом | До и после handler'а |
| Что делает | Валидация и трансформация **входных данных** | Трансформация запроса/ответа, side-effects |
| Доступ к ответу | Нет | Да (через Observable) |
| Примеры | `ValidationPipe`, `ParseIntPipe` | Логирование, кэширование, оборачивание ответа |

Pipe — про **данные на входе**. Interceptor — про **поведение вокруг** handler'а.

---

**Q: Что такое Exception Filter?**

Перехватывает исключения и формирует HTTP-ответ. NestJS имеет встроенные: `NotFoundException` (404), `BadRequestException` (400), `UnauthorizedException` (401) и т.д. Кастомный фильтр через `@Catch()` + `ExceptionFilter`.

---

## DTO и валидация

**Q: Что такое DTO и зачем нужен?**

Data Transfer Object — класс, описывающий форму входных данных. Используется с `class-validator` для автоматической валидации. Не бизнес-объект, а контракт API.

```ts
class CreateUserDto {
    @IsString() name: string;
    @IsEmail() email: string;
    @IsInt() @Min(0) age: number;
}
```

---

**Q: Зачем DTO, если TypeScript уже проверяет типы?**

TypeScript проверяет типы **только на этапе компиляции** — в рантайме его нет. Когда приходит HTTP-запрос, `body` — это обычный JSON-объект, TypeScript ничего не проверяет.

DTO + `class-validator` + `ValidationPipe` — это **runtime-валидация**: проверяет что пришедшие данные реально соответствуют ожидаемой форме. Без этого можно отправить `{ age: "not a number" }` и TypeScript не спасёт.

---

**Q: Что делает `whitelist: true` в ValidationPipe?**

Автоматически убирает все поля, которые не определены в DTO. С `forbidNonWhitelisted: true` — выбросит ошибку при лишних полях. Защита от mass assignment.

---

## Аутентификация

**Q: Как реализовать JWT-аутентификацию в NestJS?**

1. `@nestjs/passport` + `passport-jwt` — стратегия извлечения и валидации токена
2. `@nestjs/jwt` — генерация токенов
3. `JwtAuthGuard` — Guard для защиты эндпоинтов
4. `@Public()` декоратор через `SetMetadata` — для открытых роутов

```ts
@UseGuards(AuthGuard('jwt'))
@Get('profile')
getProfile(@Request() req) { return req.user; }
```

---

## Lifecycle

**Q: Какие lifecycle hooks есть в NestJS?**

1. `onModuleInit` — после создания модуля (подключение к БД)
2. `onApplicationBootstrap` — всё приложение поднялось
3. `onModuleDestroy` — при остановке модуля (отключение)
4. `onApplicationShutdown` — при остановке приложения (получает сигнал)

Для graceful shutdown нужно включить: `app.enableShutdownHooks()`.

---

## Микросервисы

**Q: Как NestJS поддерживает микросервисы?**

Встроенные транспорты: TCP, Redis, NATS, Kafka, gRPC, RabbitMQ. Два паттерна:
- `@MessagePattern` — request-response (ждёт ответа)
- `@EventPattern` — fire-and-forget (без ответа)

Hybrid application — HTTP + микросервис в одном процессе.

---

**Q: `@MessagePattern` vs `@EventPattern` — в чём разница?**

- `@MessagePattern('cmd')` — клиент отправляет запрос и ждёт ответ (`client.send()`)
- `@EventPattern('event')` — клиент эмитит событие без ожидания ответа (`client.emit()`)

---

## Прочее

**Q: Что такое ConfigModule и зачем?**

Управление конфигурацией: чтение `.env`, валидация переменных окружения (через Joi), типизированный доступ через `ConfigService`. С `isGlobal: true` — не нужно импортировать в каждый модуль.

---

**Q: Как генерировать Swagger-документацию?**

`@nestjs/swagger` — автоматическая генерация OpenAPI спецификации из декораторов:
- `@ApiTags`, `@ApiOperation`, `@ApiResponse` — на контроллерах
- `@ApiProperty` — на DTO полях

```ts
SwaggerModule.setup('api/docs', app, document);
```
