# Linux — Shell

## Что такое shell

Shell — интерпретатор командной строки, посредник между пользователем и ядром ОС. Принимает команды, передаёт их ядру и возвращает результат.

**Bash vs Zsh:**

| | Bash | Zsh |
|---|---|---|
| По умолчанию | Ubuntu, большинство Linux | macOS (с 2019) |
| Автодополнение | Базовое | Расширенное (плагины) |
| Скриптинг | Стандарт де-факто | Совместим с bash |
| Конфиг | `~/.bashrc`, `~/.bash_profile` | `~/.zshrc` |

**Login shell** — запускается при входе в систему (SSH, TTY). Загружает `~/.bash_profile` или `~/.profile`. Обычный интерактивный shell (новая вкладка терминала) загружает `~/.bashrc`.

---

## Переменные окружения

```bash
# Просмотр
env                    # все переменные
printenv PATH          # конкретная переменная
echo $HOME

# Установка (только для текущей сессии)
MY_VAR="hello"

# Экспорт (доступна дочерним процессам)
export MY_VAR="hello"
export PATH="$PATH:/my/bin"   # добавить в PATH

# Постоянно — добавить в ~/.bashrc или ~/.bash_profile
echo 'export MY_VAR="hello"' >> ~/.bashrc
```

**PATH** — список директорий, где shell ищет исполняемые файлы при вводе команды. При `ls` shell ищет `/bin/ls`, `/usr/bin/ls` и т.д. по порядку.

```bash
which python3    # показать полный путь исполняемого файла
```

---

## Полезные возможности shell

```bash
# History
history            # список команд
history 10         # последние 10
!1997              # выполнить команду №1997
!1997:p            # показать команду №1997, не выполняя
!!                 # повторить последнюю команду
Ctrl+R             # интерактивный поиск по истории

# Алиасы (только для текущей сессии)
alias ll='ls -la'
alias gs='git status'
unalias ll

# Перенаправление
command > file.txt    # stdout в файл (перезапись)
command >> file.txt   # stdout в файл (добавление)
command 2> err.txt    # stderr в файл
command &> all.txt    # stdout + stderr в файл
command < input.txt   # stdin из файла
> file.txt            # обнулить файл (перенаправление пустого вывода)

# Пайп
ps aux | grep python | wc -l
```

---

## Ключевые команды

### Навигация и работа с файлами

Базовые команды для перемещения по файловой системе и операций с файлами. `cd -` удобен, когда нужно переключаться между двумя директориями.

```bash
# pwd (print working directory) — показать текущую директорию
pwd

# ls (list) — содержимое директории
ls -la            # с правами, скрытыми файлами и размерами (-l long, -a all)

# cd (change directory) — перейти в директорию
cd /var/log       # по абсолютному пути
cd ..             # на уровень вверх
cd -              # вернуться в предыдущую директорию

# cp (copy), mv (move), rm (remove) — операции с файлами
cp file.txt /dst/         # копировать файл
cp -r dir/ /dst/          # копировать директорию (-r recursive)
mv old.txt new.txt        # переименовать / переместить
rm file.txt               # удалить файл
rm -rf dir/               # удалить директорию рекурсивно (-r recursive, -f force)

# mkdir (make directory) — создать директорию
mkdir -p a/b/c            # с вложенными (-p parents)

# touch — создать пустой файл или обновить timestamp существующего
touch file.txt
```

### Просмотр файлов

Для логов особенно полезен `tail -f` — он показывает новые строки в реальном времени. `less` удобнее `cat` для больших файлов — можно листать, искать (`/pattern`), выходить по `q`.

```bash
# cat (concatenate) — вывести содержимое файла целиком
cat file.txt

# head / tail — первые или последние строки файла
head -n 20 file.txt       # первые 20 строк
tail -n 20 file.txt       # последние 20 строк
tail -f app.log           # следить за файлом в реальном времени (-f follow)

# less — постраничный просмотр (/ — поиск, q — выход)
less file.txt
```

### Поиск

Два основных инструмента: `grep` ищет **по содержимому** файлов, `find` ищет **сами файлы** по имени, типу, размеру, дате. Их часто комбинируют через пайп или `xargs`.

```bash
# grep (global regular expression print) — поиск строк по содержимому
grep "error" app.log                    # найти строки с "error"
grep -r "TODO" ./src/                   # рекурсивно по директории (-r recursive)
grep -i "error" app.log                 # без учёта регистра (-i ignore case)
grep -n "error" app.log                 # с номерами строк (-n number)
grep -v "DEBUG" app.log                 # инвертированный — строки БЕЗ совпадения (-v invert)
grep -E "error|warn" app.log            # расширенные regex (-E extended)

# find — поиск файлов по имени, типу, размеру, дате
find /var/log -name "*.log"             # по имени
find /tmp -type f -mtime +7             # файлы старше 7 дней (-mtime modification time)
find /tmp -type f -empty               # пустые файлы
find . -name "*.log" -size +100M       # файлы больше 100 МБ
find /var/log -name "*.log" -mtime +7 -delete   # найти и удалить
```

### Обработка текста

Классический набор Unix-утилит для работы с текстовыми данными. Сила в комбинировании через пайпы: `sort | uniq -c | sort -rn` — частотный анализ строк. `awk` и `sed` — мощнее, но для простых задач хватает `cut`, `sort`, `uniq`.

```bash
# sort — сортировка строк
sort file.txt                    # по алфавиту
sort -rn numbers.txt             # числовая сортировка по убыванию (-r reverse, -n numeric)

# uniq — убрать/подсчитать дубликаты (работает только с отсортированным вводом)
sort file.txt | uniq             # убрать дубликаты
sort file.txt | uniq -c          # подсчитать повторения

# wc (word count) — подсчёт строк, слов, символов
wc -l file.txt                   # количество строк
wc -w file.txt                   # количество слов

# cut — вырезать столбцы или символы из каждой строки
cut -d',' -f1,3 data.csv         # 1-й и 3-й столбцы CSV (-d разделитель, -f поля)
cut -c1-10 file.txt              # первые 10 символов каждой строки

# xargs — превращает stdin в аргументы команды
find . -name "*.tmp" | xargs rm
echo "file1 file2" | xargs ls -la

# awk (Aho, Weinberger, Kernighan) — обработка текста по столбцам
awk '{print $1, $3}' file.txt           # напечатать 1-й и 3-й столбцы
awk -F',' '{print $2}' data.csv         # 2-й столбец CSV (-F разделитель)
awk '$3 > 1000 {print $1}' data.txt     # фильтр по значению

# sed (stream editor) — потоковая замена и удаление текста
sed 's/old/new/g' file.txt              # заменить все вхождения (s — substitute, g — global)
sed -i 's/old/new/g' file.txt           # заменить прямо в файле (-i — in-place)
sed '/pattern/d' file.txt              # удалить строки с паттерном (d — delete)
```

### Сеть

`curl` — универсальный инструмент для HTTP-запросов из командной строки, полезен для отладки API. `wget` больше подходит для скачивания файлов. Подробнее о сетевых командах — в [networking.md](networking.md).

```bash
ping google.com           # проверить доступность хоста
curl https://example.com  # HTTP-запрос
curl -I https://example.com  # только заголовки
wget https://example.com/file.zip  # скачать файл
```

---

## Практические сценарии

Реальная сила shell — в комбинировании команд через пайпы. Ниже типичные задачи, которые бекенд-разработчик решает на проде.

### Анализ логов

```bash
# Топ-10 IP-адресов по количеству запросов (nginx access.log)
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head

# Распределение HTTP-статусов
awk '{print $9}' access.log | sort | uniq -c | sort -rn
# 15230 200
#  1023 304
#   512 404
#    87 500

# Последние 100 строк с 500-ми ошибками
grep " 500 " access.log | tail -100

# Следить за ошибками в реальном времени
tail -f app.log | grep --line-buffered "ERROR"

# Количество запросов за последние 1000 строк по эндпоинтам
tail -1000 access.log | awk '{print $7}' | sort | uniq -c | sort -rn | head

# Топ-10 самых медленных эндпоинтов (если время в последнем столбце)
awk '{print $NF, $7}' access.log | sort -rn | head
```

### Работа с процессами

```bash
# Найти процесс по имени и убить
kill $(pgrep -f "python app.py")

# Убить все процессы на порту 8080
kill $(lsof -t -i:8080)

# Запустить процесс, который переживёт закрытие SSH
nohup python app.py > app.log 2>&1 &

# Повторять команду каждые 2 секунды (мониторинг)
watch -n 2 'ss -tlnp | grep 8080'
```

### Работа с файлами и дисками

```bash
# Найти 10 самых больших файлов на сервере
find / -type f -exec du -h {} + 2>/dev/null | sort -rh | head

# Очистить логи старше 30 дней
find /var/log -name "*.log" -mtime +30 -delete

# Посчитать количество файлов в директории рекурсивно
find ./src -type f | wc -l

# Сравнить два конфига
diff nginx.conf nginx.conf.bak

# Архивировать и разархивировать
tar -czf backup.tar.gz /var/www/       # создать архив
tar -xzf backup.tar.gz                 # распаковать
tar -tzf backup.tar.gz                 # посмотреть содержимое без распаковки
```

### SSH и удалённая работа

```bash
# Подключиться к серверу
ssh user@192.168.1.100
ssh -i ~/.ssh/id_rsa user@server       # с конкретным ключом

# Скопировать файл на сервер / с сервера
scp app.log user@server:/tmp/          # на сервер
scp user@server:/var/log/app.log .     # с сервера

# rsync — умная синхронизация (копирует только изменения)
rsync -avz ./deploy/ user@server:/var/www/

# Выполнить команду на удалённом сервере без входа
ssh user@server 'df -h && free -h'

# Проброс порта — доступ к БД на сервере через localhost
ssh -L 5432:localhost:5432 user@server
# теперь psql -h localhost подключится к удалённой БД
```

### Быстрый анализ данных

```bash
# Уникальные значения из CSV-столбца
cut -d',' -f3 data.csv | sort -u

# JSON из API → извлечь поле (нужен jq)
curl -s https://api.example.com/users | jq '.[].email'

# Подсчитать уникальные ошибки в логе
grep "ERROR" app.log | awk -F'ERROR' '{print $2}' | sort | uniq -c | sort -rn
```
