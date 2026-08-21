# NestJS

## Что такое NestJS

NestJS — фреймворк для Node.js на TypeScript. Вдохновлён Angular: модульная архитектура, декораторы, DI из коробки.

**Стек:** NestJS + TypeScript + Express (или Fastify) + TypeORM / Prisma

**NestJS vs Express:** Express — минималистичный, без структуры. NestJS даёт: организацию кода (модули, контроллеры, сервисы), встроенную валидацию, Guards/Interceptors/Pipes, поддержку микросервисов.

**Слоистая архитектура** — каждый слой отвечает за своё:

```
Controller       — принимает HTTP-запрос, извлекает параметры, возвращает ответ
    ↓
Service          — бизнес-логика, оркестрация
    ↓
Repository/ORM   — работа с БД (TypeORM, Prisma, MikroORM)
```

Controller не знает про БД. Service не знает про HTTP. Каждый слой зависит только от слоя ниже.

---

## Архитектура: модули

```
AppModule
├── UsersModule
│   ├── UsersController   (HTTP-обработчики)
│   ├── UsersService      (бизнес-логика)
│   └── UsersRepository   (работа с БД)
└── AuthModule
    ├── AuthController
    └── AuthService
```

Каждый модуль инкапсулирует свою функциональность. Зависимости между модулями явно объявляются через `imports/exports`.

---

## Module

```ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { User } from './user.entity';

@Module({
    imports: [TypeOrmModule.forFeature([User])],  // регистрация репозитория
    controllers: [UsersController],
    providers: [UsersService],
    exports: [UsersService],   // доступен для других модулей
})
export class UsersModule {}
```

**Поля `@Module`:**
- `imports` — другие модули, чьи exported providers нужны
- `controllers` — обработчики HTTP-запросов
- `providers` — сервисы, репозитории, guards, etc.
- `exports` — что модуль предоставляет другим

---

## Controller

```ts
import { Controller, Get, Post, Patch, Delete, Body, Param, Query, HttpCode, HttpStatus } from '@nestjs/common';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';

@Controller('users')  // prefix для всех роутов
export class UsersController {
    constructor(private readonly usersService: UsersService) {}

    @Get()
    findAll(@Query('role') role?: string) {
        return this.usersService.findAll(role);
    }

    @Get(':id')
    findOne(@Param('id') id: string) {
        return this.usersService.findOne(+id);
    }

    @Post()
    @HttpCode(HttpStatus.CREATED)
    create(@Body() createUserDto: CreateUserDto) {
        return this.usersService.create(createUserDto);
    }

    @Patch(':id')
    update(@Param('id') id: string, @Body() updateUserDto: UpdateUserDto) {
        return this.usersService.update(+id, updateUserDto);
    }

    @Delete(':id')
    @HttpCode(HttpStatus.NO_CONTENT)
    remove(@Param('id') id: string) {
        return this.usersService.remove(+id);
    }
}
```

---

## Provider / Service

Провайдер — любой класс с `@Injectable()`. Провайдер может участвовать в DI, Nest использует его metadata для разрешения зависимостей.

```ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './user.entity';
import { CreateUserDto } from './dto/create-user.dto';

@Injectable()
export class UsersService {
    constructor(
        @InjectRepository(User)
        private readonly usersRepository: Repository<User>,
    ) {}

    async findAll(role?: string): Promise<User[]> {
        if (role) {
            return this.usersRepository.findBy({ role });
        }
        return this.usersRepository.find();
    }

    async findOne(id: number): Promise<User> {
        const user = await this.usersRepository.findOneBy({ id });
        if (!user) throw new NotFoundException(`User #${id} not found`);
        return user;
    }

    async create(dto: CreateUserDto): Promise<User> {
        const user = this.usersRepository.create(dto);
        return this.usersRepository.save(user);
    }

    async update(id: number, dto: UpdateUserDto): Promise<User> {
        const user = await this.findOne(id);
        Object.assign(user, dto);
        return this.usersRepository.save(user);
    }

    async remove(id: number): Promise<void> {
        const user = await this.findOne(id);
        await this.usersRepository.remove(user);
    }
}
```

Причём сам по себе @Injectable() не регистрирует класс в модуле. Он должен попасть в DI-контейнер через: 

```ts
@Module({
  providers: [UsersService],
})
```

---

## Dependency Injection (DI)

NestJS использует IoC-контейнер: ты объявляешь зависимости через конструктор, фреймворк создаёт и внедряет их.

**Зачем DI, а не `new` напрямую?** Без DI — `new UsersService(new Repository(new Database()))` в каждом месте. Проблемы: нельзя подменить зависимость на мок (тестирование), изменил конструктор → правь везде (связность), кто управляет lifecycle? DI решает всё это.

```ts
// Классический DI через конструктор — Nest сам создаёт и внедряет зависимости
@Injectable()
class AuthService {
    constructor(
        private usersService: UsersService,
        private jwtService: JwtService,
    ) {}
}
```

### Кастомные провайдеры

Иногда нужно внедрить не конкретный класс, а абстракцию — интерфейс или токен. Nest поддерживает несколько способов создания провайдеров.

**Пример: абстракция платёжного провайдера.** Сервис не знает, работает он со Stripe или PayPal — он зависит от интерфейса.

```ts
// payment.interface.ts
interface PaymentProvider {
    pay(amount: number): Promise<void>;
}

// stripe.provider.ts
@Injectable()
class StripePaymentProvider implements PaymentProvider {
    async pay(amount: number) {
        console.log(`Stripe: charging ${amount}`);
    }
}

// payments.service.ts — зависит от абстракции, не от конкретного класса
const PAYMENT_PROVIDER = 'PAYMENT_PROVIDER';

@Injectable()
class PaymentsService {
    constructor(
        @Inject(PAYMENT_PROVIDER)
        private readonly provider: PaymentProvider,
    ) {}
}
```

А в модуле решаем, какую реализацию подставить:

```ts
// useExisting — ссылка на уже зарегистрированный провайдер
@Module({
    providers: [
        StripePaymentProvider,
        {
            provide: PAYMENT_PROVIDER,
            useExisting: StripePaymentProvider,
        },
    ],
})
export class PaymentsModule {}

// useClass — Nest создаст новый экземпляр указанного класса
{
    provide: PAYMENT_PROVIDER,
    useClass: StripePaymentProvider, // можно легко заменить на PayPalProvider
}

// useFactory — когда нужна логика при создании (например, зависимость от конфига)
{
    provide: PAYMENT_PROVIDER,
    useFactory: (config: ConfigService) => {
        return config.get('PAYMENT') === 'stripe'
            ? new StripePaymentProvider()
            : new PayPalPaymentProvider();
    },
    inject: [ConfigService],
}

// useValue — для статических значений, конфигов, моков в тестах
{
    provide: 'APP_CONFIG',
    useValue: { dbUrl: process.env.DATABASE_URL },
}
```

**Разница useExisting vs useClass:** `useExisting` создаёт алиас — оба токена указывают на один и тот же экземпляр. `useClass` создаёт новый экземпляр, даже если такой класс уже зарегистрирован.

---

## Scope провайдеров

Каждый provider в NestJS имеет scope — правило, определяющее когда Nest создаёт экземпляр и сколько их существует.

### DEFAULT (singleton)

По умолчанию все провайдеры — синглтоны. Один экземпляр на всё приложение, все запросы используют его.

```ts
@Injectable() // scope: DEFAULT по умолчанию
export class UsersService {}
```

```
new UsersService()

request 1 ─┐
request 2 ─┼→ UsersService #1
request 3 ─┘
```

### REQUEST

Новый экземпляр создаётся для каждого входящего HTTP-запроса.

```ts
@Injectable({ scope: Scope.REQUEST })
export class RequestContext {
    userId: string;
    correlationId: string;
}
```

```
Request A → RequestContext #1 (userId = "123")
Request B → RequestContext #2 (userId = "456")
```

Каждый запрос получает изолированный контекст — данные не пересекаются. Полезно для хранения request-specific данных: текущий пользователь, tenant ID, correlation ID, locale.

### TRANSIENT

Каждый **потребитель** получает свой экземпляр провайдера. Не привязан к запросу — привязан к тому, кто инжектит.

```ts
@Injectable({ scope: Scope.TRANSIENT })
export class LoggerService {}

@Injectable()
class UsersService {
    constructor(private logger: LoggerService) {} // Logger #1
}

@Injectable()
class OrdersService {
    constructor(private logger: LoggerService) {} // Logger #2
}
```

```
DEFAULT (singleton):
UsersService ──┐
OrdersService ─┼→ Logger #1
PaymentsService┘

TRANSIENT:
UsersService ───→ Logger #1
OrdersService ──→ Logger #2
PaymentsService → Logger #3
```

### Почему не делать всё REQUEST?

Потому что это дорого. Singleton — один экземпляр на 1000 запросов. REQUEST scope — 1000 экземпляров на 1000 запросов.

Кроме того, **scope заразителен** — если request-scoped сервис зависит от других провайдеров, вся цепочка зависимостей тоже становится request-scoped:

```
Controller → RequestService (REQUEST) → SomeService → Repository
                                         ↑ тоже станет REQUEST-scoped
```

Поэтому по умолчанию провайдеры — singleton, и менять scope стоит только когда действительно нужна изоляция на уровне запроса или потребителя.

---

## Lifecycle hooks

Scope отвечает на вопрос «сколько экземпляров и для кого?». Lifecycle — какие стадии проходит провайдер от создания до уничтожения.

```ts
@Injectable()
export class DatabaseService implements OnModuleInit, OnModuleDestroy {
    onModuleInit() {
        // Подключение к БД при старте модуля
    }

    onModuleDestroy() {
        // Закрытие соединения при уничтожении модуля
    }
}
```

Порядок вызова хуков:

```
constructor()
    ↓
onModuleInit()           — модуль инициализирован
    ↓
onApplicationBootstrap() — всё приложение загружено
    ↓
... приложение работает ...
    ↓
onModuleDestroy()        — модуль уничтожается
    ↓
onApplicationShutdown()  — приложение завершается
```

Типичные применения: подключение/закрытие соединений (БД, Redis, RabbitMQ), запуск background workers, graceful shutdown.

---

## DTO (Data Transfer Object)

DTO — класс, описывающий форму входных данных. Используется с `class-validator` для валидации.

**Зачем DTO, если TypeScript уже проверяет типы?** TypeScript проверяет типы **только при компиляции** — в рантайме его нет. HTTP-запрос — это обычный JSON, TypeScript ничего не проверит. DTO + `class-validator` + `ValidationPipe` — это **runtime-валидация**.

```ts
import { IsString, IsEmail, IsInt, Min, Max, IsOptional } from 'class-validator';
import { Transform } from 'class-transformer';

export class CreateUserDto {
    @IsString()
    name: string;

    @IsEmail()
    email: string;

    @IsInt()
    @Min(0)
    @Max(120)
    age: number;

    @IsOptional()
    @IsString()
    role?: string;
}

// UpdateUserDto — все поля опциональны (Partial)
import { PartialType } from '@nestjs/mapped-types';
export class UpdateUserDto extends PartialType(CreateUserDto) {}
```

---

## Pipes (Validation & Transformation)

Pipe применяется **перед** вызовом handler'а. Занимается двумя вещами: валидацией и трансформацией входных данных.

```
GET /users/123

"123"          ← строка из URL
   ↓
ParseIntPipe   ← pipe трансформирует
   ↓
123            ← число в handler
   ↓
Controller
```

```ts
// ParseIntPipe — конвертирует строку в число, кидает 400 если не число
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) { ... }

// ValidationPipe — проверяет DTO через class-validator
@Post()
create(@Body() dto: CreateUserDto) { ... }

// Глобальный ValidationPipe — валидирует все DTO автоматически
app.useGlobalPipes(new ValidationPipe({
    whitelist: true,             // убирает лишние поля из запроса
    forbidNonWhitelisted: true,  // ошибка при лишних полях
    transform: true,             // автоконвертация типов (string → number)
}));

// Кастомный Pipe
@Injectable()
export class TrimPipe implements PipeTransform {
    transform(value: string): string {
        return value.trim();
    }
}
```

---

## Interceptors

Interceptor **оборачивает** выполнение handler'а — может выполнить код **до** и **после**. Этим он отличается от Pipe (только до) и Guard (только до).

```
Interceptor (до)
   ↓
   Controller → Service
   ↓
Interceptor (после) ← имеет доступ к результату
```

Отлично подходит для: логирования времени, трансформации ответа, кэширования, tracing.

```ts
// Логирование времени выполнения — код до и после handler'а
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
    intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
        const req = context.switchToHttp().getRequest();
        const start = Date.now();

        return next.handle().pipe(
            tap(() => {
                console.log(`${req.method} ${req.url} — ${Date.now() - start}ms`);
            })
        );
    }
}

// Трансформация ответа — оборачиваем в { data: ... }
@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, { data: T }> {
    intercept(context: ExecutionContext, next: CallHandler): Observable<{ data: T }> {
        return next.handle().pipe(map(data => ({ data })));
    }
}

// Применение на контроллер или глобально
@UseInterceptors(LoggingInterceptor)
@Controller('users')
export class UsersController { ... }

app.useGlobalInterceptors(new LoggingInterceptor());
```

---

## Guards

Guard отвечает на один вопрос: **можно ли этому запросу попасть дальше?** Если Guard возвращает `false` — запрос не дойдёт до Controller.

```
Request
  ↓
Middleware
  ↓
Guard ❌ → 403 Forbidden (запрос отклонён)
  ↓ ✅
Controller
```

JWT authentication — типичный use case Guard. Guard работает в Nest `ExecutionContext` и имеет доступ к метаданным handler'а (через `Reflector`), поэтому через него удобно реализовать `@Roles('admin')`, `@Public()` и т.д.

```ts
@Injectable()
export class JwtAuthGuard implements CanActivate {
    constructor(private jwtService: JwtService) {}

    canActivate(context: ExecutionContext): boolean {
        const req = context.switchToHttp().getRequest();
        const token = req.headers.authorization?.split(' ')[1];
        if (!token) return false;

        try {
            req.user = this.jwtService.verify(token);
            return true;
        } catch {
            return false;
        }
    }
}

// Применение — если JWT невалидный, до handler'а запрос не дойдёт
@UseGuards(JwtAuthGuard)
@Get('profile')
getProfile(@Request() req) {
    return req.user;
}
```

---

## Exception Filters

```ts
// Встроенные исключения
throw new NotFoundException('User not found');
throw new BadRequestException('Invalid data');
throw new UnauthorizedException('Not authenticated');
throw new ForbiddenException('Access denied');
throw new ConflictException('Email already exists');
throw new InternalServerErrorException('Server error');

// Кастомный фильтр
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
    catch(exception: HttpException, host: ArgumentsHost) {
        const ctx = host.switchToHttp();
        const response = ctx.getResponse();
        const request = ctx.getRequest();

        response.status(exception.getStatus()).json({
            statusCode: exception.getStatus(),
            timestamp: new Date().toISOString(),
            path: request.url,
            message: exception.message,
        });
    }
}
```

---

## Entity (TypeORM)

```ts
import { Entity, PrimaryGeneratedColumn, Column, CreateDateColumn, ManyToOne } from 'typeorm';

@Entity('users')
export class User {
    @PrimaryGeneratedColumn()
    id: number;

    @Column({ length: 100 })
    name: string;

    @Column({ unique: true })
    email: string;

    @Column({ default: 'user' })
    role: string;

    @Column({ nullable: true })
    age?: number;

    @CreateDateColumn()
    createdAt: Date;
}
```

---

## Swagger / OpenAPI

```ts
// main.ts
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';

const config = new DocumentBuilder()
    .setTitle('Users API')
    .setDescription('CRUD API for users')
    .setVersion('1.0')
    .addBearerAuth()
    .build();

const document = SwaggerModule.createDocument(app, config);
SwaggerModule.setup('api/docs', app, document);

// В DTO
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

export class CreateUserDto {
    @ApiProperty({ example: 'Alice', description: 'Full name' })
    @IsString()
    name: string;

    @ApiPropertyOptional({ example: 25 })
    @IsOptional()
    @IsInt()
    age?: number;
}

// В контроллере
@ApiTags('users')
@Controller('users')
export class UsersController {
    @ApiOperation({ summary: 'Create user' })
    @ApiResponse({ status: 201, type: User })
    @Post()
    create(@Body() dto: CreateUserDto) { ... }
}
```

---

## Logger

```ts
import { Logger, Injectable } from '@nestjs/common';

@Injectable()
export class UsersService {
    private readonly logger = new Logger(UsersService.name);

    async findOne(id: number) {
        this.logger.log(`Finding user #${id}`);
        this.logger.debug(`Debug info`);
        this.logger.warn(`Something unusual`);
        this.logger.error(`Error occurred`, error.stack);
        // ...
    }
}
```

---

## Запуск и CLI

```bash
# Создание проекта
nest new my-project

# Генерация ресурсов
nest g module users
nest g controller users
nest g service users
nest g resource users   # CRUD модуль целиком (module + controller + service + dto)

# Запуск
npm run start           # обычный
npm run start:dev       # с hot-reload (watch mode)
npm run start:debug     # с дебаггером

# Сборка
npm run build           # компиляция TypeScript → dist/
```

---

## Типичный main.ts

```ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe, Logger } from '@nestjs/common';
import { AppModule } from './app.module';

async function bootstrap() {
    const app = await NestFactory.create(AppModule);
    const logger = new Logger('Bootstrap');

    // Глобальный префикс
    app.setGlobalPrefix('api/v1');

    // Валидация
    app.useGlobalPipes(new ValidationPipe({
        whitelist: true,
        transform: true,
    }));

    // CORS
    app.enableCors({ origin: process.env.FRONTEND_URL });

    const port = process.env.PORT ?? 3000;
    await app.listen(port);
    logger.log(`Application running on port ${port}`);
}

bootstrap();
```

---

## Middleware

Middleware работает **самым первым** в pipeline — ещё до Guards и Interceptors. Он знает только `req`, `res`, `next` и не имеет доступа к Nest execution context.

Типичные задачи: логирование, correlation ID, работа с cookies/headers, CORS.

```ts
@Injectable()
export class LoggerMiddleware implements NestMiddleware {
    use(req: Request, res: Response, next: NextFunction) {
        console.log(`[${req.method}] ${req.url}`);
        next();
    }
}

// Применение в модуле
export class AppModule implements NestModule {
    configure(consumer: MiddlewareConsumer) {
        consumer
            .apply(LoggerMiddleware)
            .forRoutes('users');  // или { path: '*', method: RequestMethod.ALL }
    }
}
```

---

## Порядок выполнения запроса

```
Request
  ↓
Middleware        ← самый ранний, знает только req/res/next
  ↓
Guards            ← можно ли пустить дальше? (авторизация)
  ↓
Interceptors (до) ← код перед handler'ом
  ↓
Pipes             ← валидация и трансформация входных данных
  ↓
Controller        ← handler
  ↓
Service           ← бизнес-логика
  ↓
Interceptors (после) ← код после handler'а (timing, transform response)
  ↓
Exception Filters ← при ошибке на любом этапе
  ↓
Response
```

### Guard vs Middleware

Middleware работает на уровне HTTP — он знает `req`, `res`, `next`, но **не знает** какой handler будет вызван.

Guard работает в Nest `ExecutionContext` — может получить информацию о handler/class и метаданные (через `Reflector`). Поэтому авторизацию вроде `@Roles('admin')` логичнее делать через Guard.

### Interceptor vs Middleware

Middleware видит только входящий запрос и вызывает `next()` — у него нет доступа к результату handler'а.

Interceptor оборачивает handler через `next.handle()` и получает результат — поэтому timing, response transformation, caching удобнее делать через Interceptor.

```
Middleware:              Interceptor:
req → middleware → next  interceptor → next.handle() → result → interceptor
     (нет доступа              (доступ к результату)
      к результату)
```

---

## Authentication (Passport.js + JWT)

```bash
npm install @nestjs/passport passport passport-jwt @nestjs/jwt
npm install -D @types/passport-jwt
```

```ts
// auth/jwt.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
    constructor(config: ConfigService) {
        super({
            jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
            secretOrKey: config.get('JWT_SECRET'),
        });
    }

    async validate(payload: { sub: number; email: string }) {
        return { id: payload.sub, email: payload.email };  // → req.user
    }
}

// auth/auth.module.ts
@Module({
    imports: [
        UsersModule,
        JwtModule.registerAsync({
            imports: [ConfigModule],
            inject: [ConfigService],
            useFactory: (config: ConfigService) => ({
                secret: config.get('JWT_SECRET'),
                signOptions: { expiresIn: config.get('JWT_EXPIRES_IN', '15m') },
            }),
        }),
        PassportModule,
    ],
    providers: [AuthService, JwtStrategy],
    controllers: [AuthController],
})
export class AuthModule {}

// auth/auth.service.ts
@Injectable()
export class AuthService {
    constructor(
        private usersService: UsersService,
        private jwtService: JwtService,
    ) {}

    async login(email: string, password: string) {
        const user = await this.usersService.findByEmail(email);
        if (!user || !await bcrypt.compare(password, user.passwordHash)) {
            throw new UnauthorizedException('Invalid credentials');
        }
        const payload = { sub: user.id, email: user.email };
        return {
            access_token: this.jwtService.sign(payload),
        };
    }
}

// Применение guard
@UseGuards(AuthGuard('jwt'))   // встроенный Passport guard
@Get('profile')
getProfile(@Request() req) {
    return req.user;  // { id, email } из JwtStrategy.validate()
}

// Глобально + публичные роуты
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
    canActivate(context: ExecutionContext) {
        const request = context.switchToHttp().getRequest();
        if (Reflect.getMetadata('isPublic', context.getHandler())) return true;
        return super.canActivate(context);
    }
}

// Декоратор @Public()
export const Public = () => SetMetadata('isPublic', true);

@Public()
@Post('login')
login(@Body() dto: LoginDto) { ... }
```

---

## ConfigModule / ConfigService

```ts
// app.module.ts
import { ConfigModule } from '@nestjs/config';

@Module({
    imports: [
        ConfigModule.forRoot({
            isGlobal: true,           // не нужно импортировать в каждый модуль
            envFilePath: '.env',      // файл переменных (по умолчанию)
            validationSchema: Joi.object({  // валидация через @hapi/joi
                PORT: Joi.number().default(3000),
                DATABASE_URL: Joi.string().required(),
                JWT_SECRET: Joi.string().min(32).required(),
            }),
        }),
    ],
})
export class AppModule {}

// Использование в сервисах
@Injectable()
export class AppService {
    constructor(private config: ConfigService) {}

    getDbUrl(): string {
        return this.config.get<string>('DATABASE_URL');
    }
    getPort(): number {
        return this.config.get<number>('PORT', 3000);  // 3000 — дефолт
    }
}

// Типизированная конфигурация
export default () => ({
    database: {
        url: process.env.DATABASE_URL,
        pool: parseInt(process.env.DB_POOL ?? '10'),
    },
    jwt: {
        secret: process.env.JWT_SECRET,
        expiresIn: process.env.JWT_EXPIRES ?? '15m',
    },
});

// В модуле:
ConfigModule.forRoot({ load: [configuration] })
// Использование:
this.config.get('database.url')
```

---

## Lifecycle Hooks

NestJS вызывает хуки в определённом порядке при старте/остановке приложения.

```ts
import {
    OnModuleInit, OnApplicationBootstrap,
    OnModuleDestroy, OnApplicationShutdown,
    Injectable
} from '@nestjs/common';

@Injectable()
export class DatabaseService
    implements OnModuleInit, OnApplicationBootstrap, OnModuleDestroy, OnApplicationShutdown
{
    // 1. После создания модуля (DI завершён, но другие модули могут быть не готовы)
    async onModuleInit() {
        await this.connect();
        console.log('DB connected');
    }

    // 2. После того как всё приложение поднялось (все модули инициализированы)
    async onApplicationBootstrap() {
        await this.runMigrations();
    }

    // 3. При остановке модуля (graceful shutdown)
    async onModuleDestroy() {
        await this.disconnect();
    }

    // 4. При остановке приложения (получает сигнал: SIGTERM, SIGINT)
    async onApplicationShutdown(signal: string) {
        console.log(`Shutting down on signal: ${signal}`);
    }
}

// Graceful shutdown нужно включить явно:
// main.ts
app.enableShutdownHooks();
await app.listen(3000);
```

**Порядок вызова:**
```
onModuleInit        → для каждого модуля
onApplicationBootstrap → для каждого модуля
[ приложение работает ]
onModuleDestroy     → для каждого модуля (обратный порядок)
onApplicationShutdown → для каждого модуля
```

---

## Microservices

NestJS поддерживает несколько транспортов для микросервисной архитектуры.

```bash
npm install @nestjs/microservices
```

### TCP Transport

```ts
// microservice/main.ts — запуск как микросервиса
import { NestFactory } from '@nestjs/core';
import { Transport, MicroserviceOptions } from '@nestjs/microservices';

const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
    transport: Transport.TCP,
    options: { host: 'localhost', port: 3001 },
});
await app.listen();

// Обработчик сообщений
@Controller()
export class MathController {
    @MessagePattern('sum')               // команда с ответом
    accumulate(data: number[]): number {
        return data.reduce((a, b) => a + b, 0);
    }

    @EventPattern('user.created')        // event (без ответа)
    handleUserCreated(data: UserCreatedEvent) {
        console.log('User created:', data);
    }
}

// Клиент — вызов микросервиса
@Module({
    imports: [
        ClientsModule.register([{
            name: 'MATH_SERVICE',
            transport: Transport.TCP,
            options: { host: 'localhost', port: 3001 },
        }]),
    ],
})
export class AppModule {}

@Injectable()
export class AppService {
    constructor(@Inject('MATH_SERVICE') private client: ClientProxy) {}

    sum(numbers: number[]): Observable<number> {
        return this.client.send('sum', numbers);     // request-response
    }

    notifyUserCreated(user: User): void {
        this.client.emit('user.created', user);      // fire-and-forget
    }
}
```

### Kafka Transport

```ts
// Producer
ClientsModule.registerAsync([{
    name: 'KAFKA_SERVICE',
    useFactory: (config: ConfigService) => ({
        transport: Transport.KAFKA,
        options: {
            client: { brokers: [config.get('KAFKA_BROKER')] },
            consumer: { groupId: 'api-gateway' },
        },
    }),
    inject: [ConfigService],
}])

// Consumer
@Controller()
export class OrdersConsumer {
    @MessagePattern('orders.created')
    @Payload() message: KafkaMessage,
    handleOrder(@Payload() data: OrderCreatedEvent, @Ctx() context: KafkaContext) {
        const { offset, partition } = context.getMessage();
        console.log(`Order received: partition=${partition}, offset=${offset}`);
    }
}
```

### Hybrid Application (HTTP + Microservice)

```ts
// main.ts — HTTP и микросервис в одном процессе
const app = await NestFactory.create(AppModule);

app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.KAFKA,
    options: { client: { brokers: ['localhost:9092'] } },
});

await app.startAllMicroservices();
await app.listen(3000);
```
