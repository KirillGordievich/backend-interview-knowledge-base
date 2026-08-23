# PostgreSQL — WAL (Write-Ahead Log)

## Что такое WAL

WAL (Write-Ahead Log, журнал с опережающей записью) — механизм надёжности PostgreSQL. **Принцип:** прежде чем изменить страницу данных, запись об изменении должна попасть в WAL-файл на диске.

---

## Зачем нужен WAL

**Durability (надёжность ACID):**
После `COMMIT` запись гарантированно находится в WAL на диске. При сбое питания — PostgreSQL восстановит их при следующем запуске, применив WAL к последнему checkpoint.

**Производительность:**
Запись в WAL последовательная (append-only) — на порядок быстрее случайной записи по всей БД.

**Репликация:**
WAL передаётся на standby-реплики — основа streaming replication.

**Point-in-Time Recovery (PITR):**
Архивируя WAL, можно восстановить БД на любой момент времени.

---

## Как работает WAL

```
Client                PostgreSQL                    Disk
  │                        │                         │
  │── UPDATE orders ... ──►│                         │
  │                        │── записать в WAL ───────►│ pg_wal/000001
  │── COMMIT ─────────────►│                         │
  │                        │── fsync WAL ────────────►│ (гарантия на диске)
  │◄── OK ─────────────────│                         │
  │                        │                         │
  │                  (checkpoint позже)              │
  │                        │── записать dirty pages ─►│ base/ (данные)
```

---

## Что такое checkpoint

Момент, когда PostgreSQL сбрасывает изменённые страницы (dirty pages) из shared_buffers на диск.

После checkpoint WAL до этой точки больше не нужен для восстановления — можно удалить или переиспользовать.

```sql
CHECKPOINT;  -- принудительный checkpoint

-- postgresql.conf
checkpoint_timeout = 5min    -- максимальное время между checkpoint
max_wal_size = 1GB           -- при превышении — внеплановый checkpoint
min_wal_size = 80MB          -- минимум WAL-файлов на диске
```

**При сбое:** PostgreSQL читает последний checkpoint из pg_control, затем применяет все WAL-записи после него — база восстановлена в консистентное состояние.

---

## Как WAL используется для репликации

WAL-записи можно передавать на standby-реплики — это основа streaming replication в PostgreSQL:

```
Primary ──── WAL stream ────► Standby (replica)
                                  └── apply WAL → replica data
```

```sql
-- postgresql.conf (primary)
wal_level = replica          -- включить данные для репликации
max_wal_senders = 5          -- максимум standby-подключений
wal_keep_size = 256MB        -- сколько WAL держать для реплик

-- pg_hba.conf (primary)
host  replication  replicator  192.168.1.0/24  scram-sha-256

-- recovery.conf / postgresql.conf (standby)
primary_conninfo = 'host=primary port=5432 user=replicator'
```

**Синхронная vs асинхронная репликация:**
- `synchronous_commit = on` — COMMIT ждёт подтверждения от реплики (0 потерь, медленнее)
- `synchronous_commit = off` — COMMIT не ждёт (возможна потеря ~wal_writer_delay данных, быстрее)

---

## Какие бывают уровни WAL (wal_level)

Параметр `wal_level` определяет, сколько информации пишется в WAL. Чем выше уровень, тем больше данных и больше возможностей:

```sql
-- postgresql.conf
wal_level = minimal    -- только для crash recovery (нельзя реплицировать)
wal_level = replica    -- для streaming replication (по умолчанию)
wal_level = logical    -- для logical replication (CDC, Debezium)
```

---

## Какие команды полезны для работы с WAL

Основные диагностические запросы для мониторинга WAL и репликации:

```sql
-- Текущая позиция в WAL
SELECT pg_current_wal_lsn();

-- Отставание реплики
SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes
FROM pg_stat_replication;

-- Список WAL-файлов
SELECT * FROM pg_ls_waldir() ORDER BY modification DESC LIMIT 10;

-- Размер pg_wal на диске
SELECT pg_size_pretty(sum(size)) FROM pg_ls_waldir();
```
