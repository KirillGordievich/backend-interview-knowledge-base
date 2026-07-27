# Linux — Диагностика и troubleshooting

## Команды мониторинга

### CPU

```bash
top                            # интерактивный мониторинг (q — выход)
htop                           # улучшенный top
ps aux --sort=-%cpu | head     # топ процессов по CPU
mpstat 1                       # статистика по CPU (нужен sysstat)
```

В `top`: нажать `P` — сортировать по CPU, `M` — по памяти, `k` — убить процесс.

### Память

```bash
free -h                        # RAM и swap (human-readable)
vmstat 1 5                     # статистика каждую секунду, 5 раз
ps aux --sort=-%mem | head     # топ процессов по памяти
cat /proc/meminfo              # детальная информация
```

### Диск

```bash
df -h                          # использование дисков (по файловым системам)
df -ih                         # использование inodes
du -sh /var/log/*              # размер директорий в /var/log
du -sh /* 2>/dev/null | sort -rh | head  # топ директорий по размеру
iostat -x 1                    # I/O статистика (нужен sysstat)
lsof | grep deleted            # удалённые файлы, удерживаемые процессами
```

---

## Типичные сценарии

### Закончилось место на диске

```bash
df -h                          # 1. Проверить заполненность разделов
du -sh /* 2>/dev/null | sort -rh | head -20   # 2. Найти крупнейшие директории
du -sh /var/log/* | sort -rh | head           # 3. Проверить логи

# Очистить логи systemd
journalctl --vacuum-size=500M

# Найти крупные файлы
find / -type f -size +1G 2>/dev/null

# Файл удалён, но место не освободилось?
lsof | grep deleted    # процесс держит файл открытым
# → перезапустить процесс
```

### Процесс грузит 100% CPU

```bash
top                            # 1. Определить PID
ps aux --sort=-%cpu | head     # альтернатива

# 2. Что делает процесс
strace -p 1234                 # системные вызовы
cat /proc/1234/status          # статус процесса
ls -la /proc/1234/fd/          # открытые файловые дескрипторы

# 3. Стек потоков Python
kill -SIGUSR1 1234    # если приложение поддерживает
# или
py-spy top --pid 1234          # профилировщик для Python
```

### Сервер начал тормозить

```bash
uptime                         # load average (нагрузка за 1/5/15 минут)
# load average > числа CPU → перегрузка

top                            # CPU, память, swap
free -h                        # не закончилась ли память?
df -h                          # не кончилось ли место?
iostat -x 1                    # I/O wait высокий?

journalctl -p err --since "1 hour ago"  # ошибки за последний час
dmesg | tail -50               # ошибки ядра
```

**Load average:** три числа — средняя нагрузка за 1, 5, 15 минут. На 4-ядерной системе нормально до ~4.0. Постоянно высокий → CPU-bound или много I/O-wait.

### Порт не открывается / сервис недоступен

```bash
# 1. Сервис запущен?
systemctl status myapp

# 2. Слушает нужный порт?
ss -tlnp | grep 8080
lsof -i :8080

# 3. Firewall блокирует?
iptables -L -n | grep 8080
ufw status

# 4. Сервис привязан к 127.0.0.1 вместо 0.0.0.0?
ss -tlnp | grep 8080    # смотреть Local Address

# 5. Проверить с самого сервера
curl localhost:8080
```

### Приложение внезапно завершилось

```bash
# 1. Посмотреть логи сервиса
journalctl -u myapp --since "30 minutes ago"
journalctl -u myapp -n 200

# 2. OOM Kill?
dmesg | grep -i "out of memory"
dmesg | grep -i oom
journalctl | grep -i oom

# 3. Сигнал? Код выхода?
systemctl status myapp    # смотреть "Main PID ... code=killed, signal=KILL"

# 4. Сегфолт?
dmesg | grep segfault
```

### Приложение не завершается по `docker stop`

Причина: приложение — PID 1 в контейнере, не обрабатывает SIGTERM. Docker ждёт 10 секунд, затем SIGKILL.

```bash
docker inspect mycontainer | grep -i pid   # PID 1 внутри контейнера

# Решение 1: tini
ENTRYPOINT ["/usr/bin/tini", "--", "python", "app.py"]

# Решение 2: обработать SIGTERM в приложении
import signal, sys
signal.signal(signal.SIGTERM, lambda *_: sys.exit(0))

# Решение 3: увеличить таймаут
docker stop --time=30 mycontainer
```

### Поиск процесса, съедающего память

```bash
ps aux --sort=-%mem | head -10
cat /proc/$(pgrep myapp)/status | grep VmRSS   # реальная RAM процесса

# Для Python
pip install memory-profiler
python -m memory_profiler app.py

# tracemalloc (встроен в Python)
import tracemalloc
tracemalloc.start()
# ... код ...
snapshot = tracemalloc.take_snapshot()
for stat in snapshot.statistics('lineno')[:10]:
    print(stat)
```

---

## Шпаргалка: что смотреть первым

| Симптом | Первые команды |
|---|---|
| Сервер тормозит | `uptime`, `top`, `free -h`, `df -h` |
| Нет места на диске | `df -h`, `du -sh /* \| sort -rh`, `lsof \| grep deleted` |
| 100% CPU | `top`, `ps aux --sort=-%cpu` |
| Много памяти | `free -h`, `ps aux --sort=-%mem` |
| Сервис упал | `systemctl status`, `journalctl -u service -n 100` |
| Порт недоступен | `ss -tlnp`, `systemctl status`, `iptables -L` |
| Приложение убито без причины | `dmesg \| grep oom`, `journalctl \| grep oom` |
