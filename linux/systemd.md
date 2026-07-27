# Linux — systemd и логи

## Что такое systemd

systemd — система инициализации (PID 1) и менеджер сервисов в большинстве современных Linux-дистрибутивов (Ubuntu, Debian, RHEL, Arch...). Заменяет SysV init.

**Что делает:**
- Запускает сервисы при старте системы
- Управляет зависимостями между сервисами
- Перезапускает упавшие сервисы
- Собирает логи (journald)
- Управляет монтированием, таймерами, сокетами

---

## Unit

Unit — базовая единица конфигурации systemd. Описывает управляемый ресурс.

**Типы units:**

| Тип | Расширение | Описание |
|---|---|---|
| Service | `.service` | Фоновый сервис (nginx, postgresql) |
| Socket | `.socket` | Сокет-активация |
| Timer | `.timer` | Аналог cron |
| Mount | `.mount` | Точка монтирования |
| Target | `.target` | Группа units (аналог runlevel) |

Unit-файлы находятся в `/etc/systemd/system/` (пользовательские) и `/lib/systemd/system/` (системные).

### Пример unit-файла

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Python Application
After=network.target postgresql.service

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/venv/bin/python app.py
Restart=always
RestartSec=5
Environment=ENV=production

[Install]
WantedBy=multi-user.target
```

---

## systemctl — управление сервисами

```bash
# Статус
systemctl status nginx           # статус сервиса
systemctl is-active nginx        # active / inactive
systemctl is-enabled nginx       # включён ли автозапуск

# Управление
systemctl start nginx            # запустить
systemctl stop nginx             # остановить
systemctl restart nginx          # перезапустить (stop + start)
systemctl reload nginx           # перечитать конфиг (SIGHUP, без остановки)
systemctl kill nginx             # послать SIGTERM

# Автозапуск при старте системы
systemctl enable nginx           # включить автозапуск (создать symlink)
systemctl disable nginx          # выключить автозапуск
systemctl enable --now nginx     # включить + сразу запустить

# Просмотр
systemctl list-units             # все активные units
systemctl list-units --failed    # упавшие
systemctl list-unit-files        # все unit-файлы и их статус

# После редактирования unit-файла
systemctl daemon-reload          # перечитать файлы без перезапуска
```

---

## journalctl — просмотр логов

systemd-journald собирает логи из всех сервисов в бинарный журнал.

```bash
journalctl                        # все логи (с начала)
journalctl -n 100                 # последние 100 строк
journalctl -f                     # следить в реальном времени (как tail -f)

# Фильтрация по сервису
journalctl -u nginx               # логи конкретного сервиса
journalctl -u nginx -f            # следить за nginx
journalctl -u nginx --since today # только за сегодня

# Фильтрация по времени
journalctl --since "2024-01-01 10:00:00"
journalctl --since "1 hour ago"
journalctl --until "2024-01-01 12:00:00"

# Фильтрация по уровню
journalctl -p err                 # только ошибки (emerg, alert, crit, err)
journalctl -p warning             # warning и выше

# По PID
journalctl _PID=1234

# Очистить старые логи
journalctl --vacuum-size=500M     # оставить последние 500 МБ
journalctl --vacuum-time=7d       # оставить за последние 7 дней
```

---

## Традиционные логи

Параллельно с journald многие сервисы пишут в `/var/log/`:

```bash
/var/log/syslog          # системный лог (Ubuntu)
/var/log/messages        # системный лог (RHEL)
/var/log/auth.log        # аутентификация, sudo, SSH
/var/log/kern.log        # сообщения ядра
/var/log/nginx/          # логи nginx (access.log, error.log)
/var/log/postgresql/     # логи PostgreSQL

# Сообщения ядра
dmesg                    # буфер сообщений ядра
dmesg | tail -50         # последние 50
dmesg -T                 # с человекочитаемыми временными метками
dmesg | grep -i error    # поиск ошибок
```
