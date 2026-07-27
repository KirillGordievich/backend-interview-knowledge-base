# NestJS

## Что такое NestJS

NestJS — фреймворк для Node.js на TypeScript. Вдохновлён Angular: модульная архитектура, декораторы, DI из коробки.

**Стек:** NestJS + TypeScript + Express (или Fastify) + TypeORM / Prisma

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

Провайдер — любой класс с `@Injectable()`. NestJS управляет его созданием и инъекцией.

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

---

## Dependency Injection (DI)

NestJS использует IoC-контейнер: ты объявляешь зависимости через конструктор, фреймворк создаёт и внедряет их.

```ts
// Классический DI через конструктор
@Injectable()
class AuthService {
    constructor(
        private usersService: UsersService,
        private jwtService: JwtService,
    ) {}
}

// Кастомный провайдер — useValue
const providers = [
    {
        provide: 'CONFIG',
        useValue: { dbUrl: process.env.DATABASE_URL }
    }
];
// Инъекция:
constructor(@Inject('CONFIG') private config: AppConfig) {}

// useFactory — создаёт провайдер через фабрику
{
    provide: 'CACHE',
    useFactory: (config: ConfigService) => new Redis(config.get('REDIS_URL')),
    inject: [ConfigService]
}
```

---

## DTO (Data Transfer Object)

DTO — класс, описывающий форму входных данных. Используется с `class-validator` для валидации.

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

Pipe обрабатывает входные данные перед хендлером.

```ts
// Глобальный ValidationPipe — валидирует все DTO
// main.ts
app.useGlobalPipes(new ValidationPipe({
    whitelist: true,        // убирает лишние поля из запроса
    forbidNonWhitelisted: true,  // ошибка при лишних полях
    transform: true,        // автоконвертация типов (string → number)
    transformOptions: {
        enableImplicitConversion: true
    }
}));

// ParseIntPipe — конвертирует строку в число
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) { ... }

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

Interceptor — перехватывает запрос/ответ. Используется для логирования, кэширования, трансформации ответа.

```ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map, tap } from 'rxjs/operators';

// Логирование времени выполнения
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
    intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
        const req = context.switchToHttp().getRequest();
        const start = Date.now();

        return next.handle().pipe(
            tap(() => {
                const ms = Date.now() - start;
                console.log(`${req.method} ${req.url} — ${ms}ms`);
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

// Применение
@UseInterceptors(LoggingInterceptor)
@Controller('users')
export class UsersController { ... }

// Глобально
app.useGlobalInterceptors(new LoggingInterceptor());
```

---

## Guards

Guard определяет, может ли запрос быть обработан (авторизация).

```ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';

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

// Применение
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

## Scopes провайдеров

```ts
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.DEFAULT })    // Singleton — один на всё приложение (по умолчанию)
class SingletonService {}

@Injectable({ scope: Scope.REQUEST })    // Новый экземпляр на каждый HTTP-запрос
class RequestScopedService {}

@Injectable({ scope: Scope.TRANSIENT })  // Новый экземпляр при каждой инъекции
class TransientService {}
```

**Важно:** REQUEST scope поднимается по дереву зависимостей — все зависимости тоже станут REQUEST scope.

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

```ts
// Middleware выполняется до Guards и Interceptors
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
Incoming Request
    → Middleware
    → Guards
    → Interceptors (before)
    → Pipes (validation/transformation)
    → Controller/Route Handler
    → Interceptors (after)
    → Exception Filters (при ошибке)
    → Response
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
