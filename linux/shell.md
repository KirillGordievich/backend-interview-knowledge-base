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

# Пайп
ps aux | grep python | wc -l
```

---

## Ключевые команды

### Навигация и файлы

```bash
pwd               # текущая директория
ls -la            # список с правами и скрытыми файлами
cd /var/log       # перейти по абсолютному пути
cd ..             # на уровень вверх
cd -              # вернуться в предыдущую директорию

cp file.txt /dst/         # копировать файл
cp -r dir/ /dst/          # копировать директорию
mv old.txt new.txt        # переименовать / переместить
rm file.txt               # удалить файл
rm -rf dir/               # удалить директорию рекурсивно (осторожно!)
mkdir -p a/b/c            # создать вложенные директории
touch file.txt            # создать файл / обновить timestamp
```

### Просмотр файлов

```bash
cat file.txt              # вывести всё содержимое
head -n 20 file.txt       # первые 20 строк
tail -n 20 file.txt       # последние 20 строк
tail -f app.log           # следить за файлом в реальном времени
less file.txt             # постраничный просмотр
```

### Поиск

```bash
# grep — поиск по содержимому
grep "error" app.log                    # найти строки с "error"
grep -r "TODO" ./src/                   # рекурсивно по директории
grep -i "error" app.log                 # без учёта регистра
grep -n "error" app.log                 # с номерами строк
grep -v "DEBUG" app.log                 # инвертированный (без "DEBUG")
grep -E "error|warn" app.log            # расширенные regex

# find — поиск файлов
find /var/log -name "*.log"             # по имени
find /tmp -type f -mtime +7             # файлы старше 7 дней
find /tmp -type f -empty               # пустые файлы
find . -name "*.log" -size +100M       # крупные файлы
find . -name "*.log" -delete           # найти и удалить

# Найти файлы старше 7 дней и удалить
find /var/log -name "*.log" -mtime +7 -delete
# Удалить все пустые файлы
find . -type f -empty -delete
```

### Обработка текста

```bash
# sort, uniq, wc
sort file.txt                    # сортировать строки
sort -rn numbers.txt             # числовая сортировка по убыванию
sort file.txt | uniq             # убрать дубликаты
sort file.txt | uniq -c          # подсчитать повторения
wc -l file.txt                   # количество строк
wc -w file.txt                   # количество слов

# cut — вырезать столбцы
cut -d',' -f1,3 data.csv         # 1-й и 3-й столбцы CSV
cut -c1-10 file.txt              # первые 10 символов каждой строки

# xargs — передать вывод как аргументы
find . -name "*.tmp" | xargs rm
echo "file1 file2" | xargs ls -la

# awk — обработка колонок
awk '{print $1, $3}' file.txt           # напечатать 1-й и 3-й столбцы
awk -F',' '{print $2}' data.csv         # 2-й столбец CSV
awk '$3 > 1000 {print $1}' data.txt     # фильтр по значению

# sed — потоковый редактор
sed 's/old/new/g' file.txt              # заменить все вхождения
sed -i 's/old/new/g' file.txt           # заменить в файле
sed '/pattern/d' file.txt              # удалить строки с паттерном
```

### Сеть

```bash
ping google.com           # проверить доступность хоста
curl https://example.com  # HTTP-запрос
curl -I https://example.com  # только заголовки
wget https://example.com/file.zip  # скачать файл
```

### Разное

```bash
sudo command              # выполнить от имени root
chmod 755 script.sh       # изменить права доступа
./script.sh               # выполнить скрипт в текущей директории
shred -u file.txt         # безопасно удалить (перезаписать случайными данными)
exit                      # завершить сессию shell
```
