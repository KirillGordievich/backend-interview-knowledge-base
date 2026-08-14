# Backend Interview Knowledge Base

Шпаргалки для подготовки к техническим интервью на позицию Python Backend-разработчика.

---

## Python

| Файл | Темы |
|---|---|
| [basics.md](python/basics.md) | Последовательности, множества, словари, хэширование, сложность операций, walrus `:=` |
| [functions.md](python/functions.md) | *args/**kwargs, замыкания, декораторы, `@foobar` vs `@foobar()`, functools |
| [iterators.md](python/iterators.md) | Iterable vs Iterator, генераторы, `yield`, `send()`, `yield from`, itertools |
| [oop.md](python/oop.md) | Классы, ABC, метаклассы, MRO, `__new__` vs `__init__`, дескрипторы, магические методы, миксины, `__slots__` |
| [async.md](python/async.md) | Event loop, корутины, `asyncio.gather`, `TaskGroup`, примитивы синхронизации, uvloop, Uvicorn |
| [concurrency.md](python/concurrency.md) | GIL, threading, multiprocessing, ProcessPoolExecutor, когда что использовать |
| [typing.md](python/typing.md) | Type hints, `Any/object/type`, TypeVar, Generic, Protocol vs ABC, TypedDict, ParamSpec, TypeGuard, Annotated + вопросы с ответами |
| [exceptions.md](python/exceptions.md) | Иерархия исключений, try/except/else/finally, цепочки, кастомные исключения |
| [internals.md](python/internals.md) | Управление памятью, reference counting, GC, интернирование, `perf_counter` |
| [modules.md](python/modules.md) | Модули, пакеты, `sys.path`, `__all__`, `importlib.reload` |
| [io.md](python/io.md) | Файловые объекты, режимы `open()`, `StringIO/BytesIO`, json, pickle |
| [tools.md](python/tools.md) | Ruff, flake8, mypy, Poetry, uv, pre-commit |

---

## SQL

| Файл | Темы |
|---|---|
| [basics.md](sql/basics.md) | SQL, нормализация, 1NF/2NF/3NF, DDL/DML/DCL, DELETE vs TRUNCATE |
| [transactions.md](sql/transactions.md) | Транзакции, ACID, уровни изоляции (READ UNCOMMITTED → SERIALIZABLE), аномалии (dirty/non-repeatable/phantom read, write skew), MVCC, SAVEPOINT |
| [joins.md](sql/joins.md) | INNER/LEFT/FULL OUTER JOIN, GROUP BY, HAVING, UNION/INTERSECT/EXCEPT |
| [indexes.md](sql/indexes.md) | Виды индексов, кластерный vs некластерный, когда планировщик не использует индекс, EXPLAIN |
| [constraints.md](sql/constraints.md) | PRIMARY KEY, FOREIGN KEY + CASCADE, UNIQUE, CHECK, отношения 1:1/1:N/M:N |
| [cte.md](sql/cte.md) | CTE (`WITH`), подзапросы, коррелированные подзапросы, рекурсивные CTE, EXISTS vs IN vs JOIN |
| [window_functions.md](sql/window_functions.md) | ROW_NUMBER/RANK/DENSE_RANK, LAG/LEAD, агрегаты с OVER, PARTITION BY, рамка окна |
| [views.md](sql/views.md) | Обычные и материализованные представления, REFRESH, обновляемые view |
| [locking.md](sql/locking.md) | Table-level блокировки (8 режимов), row-level блокировки (FOR UPDATE/SHARE), advisory locks, SELECT FOR UPDATE, NOWAIT, SKIP LOCKED, дедлоки, мониторинг блокировок |
| [orm.md](sql/orm.md) | ORM, проблема N+1, eager loading, SQL-инъекции, когда использовать raw SQL |

---

## Linux

| Файл | Темы |
|---|---|
| [shell.md](linux/shell.md) | Shell, bash vs zsh, PATH, env vars, grep/find/awk/sed/xargs и другие команды |
| [filesystem.md](linux/filesystem.md) | Inode, hard link vs symlink, что происходит при `rm`, структура /etc /var /proc /tmp |
| [permissions.md](linux/permissions.md) | rwx, chmod числовой/символьный, setuid/setgid/sticky bit |
| [processes.md](linux/processes.md) | Процесс vs программа, fork/exec, PID/PPID, zombie/orphan/daemon, сигналы, PID 1 в Docker |
| [memory.md](linux/memory.md) | Виртуальная память, страницы, page fault, swap, mmap, OOM Killer |
| [networking.md](linux/networking.md) | Сокеты, 0.0.0.0 vs 127.0.0.1, порты, ss vs netstat, DNS, что происходит при `curl google.com` |
| [systemd.md](linux/systemd.md) | systemd, unit-файлы, systemctl, journalctl |
| [troubleshooting.md](linux/troubleshooting.md) | Нет места, 100% CPU, утечка памяти, порт недоступен, OOM kill, docker stop — разбор сценариев |

---

## PostgreSQL

| Файл | Темы |
|---|---|
| [isolation.md](postgresql/isolation.md) | Уровни изоляции, аномалии (dirty read, phantom read) |
| [planner.md](postgresql/planner.md) | Query planner, EXPLAIN, статистика |
| [vacuum.md](postgresql/vacuum.md) | VACUUM, AUTOVACUUM, bloat |
| [wal.md](postgresql/wal.md) | WAL, checkpoint, streaming replication, PITR, wal_level |

---

## SQL — Дополнительно

| Файл | Темы |
|---|---|
| [advanced.md](sql/advanced.md) | OLTP vs OLAP, Star Schema, гонки данных, распределённые транзакции, 2PC, Saga |

---

## Web

| Файл | Темы |
|---|---|
| [protocols.md](web/protocols.md) | TCP/IP, HTTP/1.1/2/3, HTTPS, TLS handshake, WebSockets, RPC, gRPC (Protobuf, streaming), JSON-RPC, Zero Copy |
| [performance.md](web/performance.md) | TTFB, latency, throughput, RPS, percentile, event loop lag, sticky sessions |
| [auth.md](web/auth.md) | Authentication vs Authorization, Sessions, JWT, Access/Refresh токены, CORS |
| [api.md](web/api.md) | REST принципы, HTTP методы, OpenAPI/Swagger |
| [nginx.md](web/nginx.md) | Reverse proxy, load balancing, SSL termination, location, upstream, rate limiting, кэш |

---

## Архитектура

| Файл | Темы |
|---|---|
| [principles.md](architecture/principles.md) | SOLID, DRY, YAGNI, KISS, ООП (инкапсуляция/наследование/полиморфизм/абстракция), coupling & cohesion |
| [patterns.md](architecture/patterns.md) | IoC, DI, CQRS, DDD, направление зависимости vs поток данных, Unit of Work |
| [gof_patterns.md](architecture/gof_patterns.md) | 22 GoF паттерна: Creational (Abstract Factory, Builder, Factory Method, Prototype, Singleton), Structural (Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy), Behavioral (Chain of Responsibility, Command, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, Visitor) |
| [observability.md](architecture/observability.md) | Структурированные логи, контекст, метрики (Prometheus), трейсинг (OpenTelemetry), алертинг, ELK |
| [fundamentals.md](architecture/fundamentals.md) | Статическая vs динамическая типизация, биты/байты, безопасность памяти, ETL, DevOps |

---

## Тестирование

| Файл | Темы |
|---|---|
| [testing.md](testing/testing.md) | Unit/Integration/E2E, pytest, mocking/stubbing, фикстуры, CI/CD |

---

## Node.js

| Файл | Темы |
|---|---|
| [javascript.md](nodejs/javascript.md) | Типы, NaN, hoisting, замыкания, this, call/apply/bind, копирование объектов, array methods, деструктуризация, Event Loop, Promises, async/await, Map/Set/WeakMap, var/let/const scope, Symbol + Symbol.iterator, spread/rest, optional chaining `?.`/`??`, generators |
| [typescript.md](nodejs/typescript.md) | any/unknown, void/never, interface vs type vs abstract class, generics, utility types (Partial/Pick/Omit/Record), type assertion, enum, модификаторы доступа, mapped types, conditional types, decorators |
| [nestjs.md](nodejs/nestjs.md) | Модули, провайдеры, контроллеры, DI/IoC, DTO + валидация, pipes, interceptors, guards, exception filters, scopes, TypeORM entity, Swagger, middleware, порядок выполнения запроса, Auth (Passport.js + JWT), ConfigModule, lifecycle hooks, microservices (TCP/Kafka) |
| [nodejs.md](nodejs/nodejs.md) | Архитектура (V8 + libuv), фазы Event Loop, setImmediate vs nextTick vs setTimeout(0), CommonJS vs ESM, EventEmitter, Streams + pipeline, Buffer, Cluster, Worker Threads, process |

---

## Message Queue

| Файл | Темы |
|---|---|
| [concepts.md](mq/concepts.md) | Зачем MQ, Point-to-Point vs Pub/Sub, Consumer Groups, гарантии доставки, ACK, DLQ, идемпотентность, backpressure, ordering, TTL |
| [rabbitmq.md](mq/rabbitmq.md) | Exchange types (direct/fanout/topic), bindings, durable queues, prefetch, DLQ, retry с backoff, RPC поверх RabbitMQ |
| [kafka.md](mq/kafka.md) | Partitions, offsets, consumer groups, rebalancing, acks, exactly-once, retention, log compaction, consumer lag, Saga паттерн |

---

## System Design

| Файл | Темы |
|---|---|
| [caching.md](system-design/caching.md) | LRU/LFU/FIFO, Redis cache-aside/write-through/write-behind, инвалидация, cache stampede/penetration/avalanche |
| [distributed_systems.md](system-design/distributed_systems.md) | CAP теорема, PACELC, BASE vs ACID, 2PC, Saga (choreography/orchestration), Outbox pattern, Eventual consistency, идемпотентность |
| [rate_limiting.md](system-design/rate_limiting.md) | Token Bucket, Leaky Bucket, Fixed/Sliding Window, distributed rate limiting (Redis + Lua), API Gateway (Nginx), HTTP-заголовки, стратегии при превышении |
| [databases.md](system-design/databases.md) | SQL vs NoSQL, replication (master-slave/multi-master), sharding (range/hash/consistent hashing), Snowflake ID, partitioning, connection pooling (PgBouncer), read replicas, CQRS routing, индексы |
| [messaging_patterns.md](system-design/messaging_patterns.md) | Sync vs Async, Event-Driven Architecture, Event Sourcing, CQRS, choreography vs orchestration, backpressure, Transactional Outbox + CDC, идемпотентность, DLQ, Circuit Breaker, Service Mesh |

---

## Blockchain

| Файл | Темы |
|---|---|
| [evm.md](blockchain/evm.md) | EVM как машина состояний, смарт-контракты, gas (EIP-1559), статусы транзакций, ABI, Events, Proxy pattern, reentrancy |
| [transactions.md](blockchain/transactions.md) | Структура транзакции, nonce, жизненный цикл, mempool, PBS (Proposer-Builder Separation), MEV, sandwich attack, типы нод, RPC |
| [cryptography.md](blockchain/cryptography.md) | Требования к хешу, Merkle Tree, асимметричная криптография, ECDSA, ZK-SNARKs vs ZK-STARKs, ZK Rollups, HD wallets BIP-32/39 |
| [consensus.md](blockchain/consensus.md) | PoW, PoS, Ethereum validator/epoch/slot, slashing, finality, DPoS, PBFT, Tendermint, reorg, L2 (Optimistic/ZK Rollups), bridges |

---

## AI / LLM

| Файл | Темы |
|---|---|
| [context_window.md](ai/context_window.md) | Токены, контекстное окно, chunking (fixed/semantic/parent-child), sliding window, hierarchical summarization, retrieval before generation, context compression (LLMLingua), KV cache / prompt caching, типы памяти агента |
| [hallucinations.md](ai/hallucinations.md) | Виды галлюцинаций, почему возникают, RAG как заземление, structured outputs, function calling, few-shot, system prompt инструкции, verification chain, self-consistency, temperature, метрики RAGAS |
| [rag.md](ai/rag.md) | Embeddings, векторные БД (pgvector/Qdrant/Chroma), chunking стратегии, dense/sparse/hybrid retrieval, HyDE, multi-query, reranking (cross-encoder/Cohere), полный pipeline, RAGAS оценка, продвинутые паттерны |
| [agents.md](ai/agents.md) | Tool use / function calling, агентный цикл, ReAct, типы памяти, планирование (sequential/parallel/hierarchical), мультиагентные системы, LangGraph, надёжность (retry/guardrails), observability, оценка агентов |

---

## Docker

| Файл | Темы |
|---|---|
| [basics.md](docker/basics.md) | Контейнер vs VM, архитектура Docker, образы и слои, контейнеры, Dockerfile, ENTRYPOINT vs CMD, exec vs shell форма, .dockerignore, namespaces/cgroups |
| [compose.md](docker/compose.md) | Docker Compose, сервисы, depends_on + healthcheck, profiles, переменные, override-файлы |
| [networking.md](docker/networking.md) | Bridge/host/none/overlay, user-defined bridge, DNS, проброс портов, сети в Compose |
| [volumes.md](docker/volumes.md) | Volumes vs bind mounts vs tmpfs, named/anonymous volumes, backup, node_modules трюк |
| [optimization.md](docker/optimization.md) | Multi-stage builds, выбор базового образа, кэш слоёв, безопасность, HEALTHCHECK, сканирование |

---

## Kubernetes

| Файл | Темы |
|---|---|
| [basics.md](kubernetes/basics.md) | Архитектура кластера (Control Plane, Worker Node), Pod, ReplicaSet, Deployment, StatefulSet, DaemonSet, Job, Namespaces, Labels |
| [services.md](kubernetes/services.md) | Service (ClusterIP/NodePort/LoadBalancer), Headless Service, DNS, Ingress, Ingress Controller |
| [storage.md](kubernetes/storage.md) | Volumes, PV, PVC, StorageClass, dynamic provisioning, StatefulSet + volumeClaimTemplates |
| [configuration.md](kubernetes/configuration.md) | ConfigMap, Secret, resource requests/limits, QoS классы, Liveness/Readiness/Startup probes |
| [scaling.md](kubernetes/scaling.md) | RollingUpdate/Recreate, Blue-Green, Canary, HPA, VPA, Taints/Tolerations, Node Affinity, PDB |

---

## Другие разделы

- [algorithms/](algorithms/) — алгоритмы и структуры данных
- [git/](git/) — система контроля версий
