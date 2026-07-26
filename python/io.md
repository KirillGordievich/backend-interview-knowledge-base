# Python — Ввод/Вывод и сериализация

## Файловые объекты

Файловый объект предоставляет API (`read`, `write`, ...) для доступа к ресурсу: файл на диске, сокет, буфер в памяти и т.д. Также называются **потоками**. Все файловые объекты — контекстные менеджеры.

**Три вида** (модуль `io`):
- `TextIOWrapper` — текстовые файлы (кодировки, переносы строк)
- `RawIOBase` — небуферизированные бинарные
- `BufferedIOBase` — буферизированные бинарные

---

## open()

```python
open(file, mode='r', buffering=-1, encoding=None, errors=None, newline=None)
```

### Режимы (mode)

| Символ | Значение |
|---|---|
| `r` | чтение (по умолчанию) |
| `w` | запись (очищает файл) |
| `x` | исключительное создание (ошибка если файл существует) |
| `a` | добавление в конец |
| `t` | текстовый режим (по умолчанию) |
| `b` | бинарный режим |
| `+` | чтение и запись одновременно |

```python
open('f.txt', 'r')    # читать текст
open('f.txt', 'w')    # писать текст (перезаписать)
open('f.txt', 'a')    # добавлять в конец
open('f.bin', 'rb')   # читать бинарный
open('f.bin', 'wb')   # писать бинарный
open('f.txt', 'r+')   # читать и писать (файл должен существовать)
```

Всегда закрывайте файл или используйте `with`:

```python
# Правильно — файл закроется автоматически даже при исключении
with open('data.txt', encoding='utf-8') as f:
    content = f.read()

# Методы чтения
f.read()        # всё содержимое как строка
f.read(100)     # первые 100 символов
f.readline()    # одна строка
f.readlines()   # список всех строк

# Итерация построчно (экономит память)
with open('big_file.txt') as f:
    for line in f:
        process(line.rstrip('\n'))

# Запись
f.write('hello\n')
f.writelines(['line1\n', 'line2\n'])
```

---

## tell() и seek()

```python
with open('data.txt') as f:
    f.read(5)         # прочитать 5 символов
    f.tell()          # → 5 (текущая позиция)

    f.seek(0)         # вернуться в начало
    f.seek(0, 2)      # перейти в конец (whence=2)
    f.seek(-10, 2)    # 10 символов до конца
    f.seek(5, 1)      # +5 от текущей позиции (whence=1)
```

`whence`: `0` — начало, `1` — текущая позиция, `2` — конец.

---

## StringIO и BytesIO

Файлоподобные объекты в памяти. Используются когда API ожидает файл, но данные уже в строке/байтах.

```python
from io import StringIO, BytesIO

# StringIO — текстовый буфер
buf = StringIO()
buf.write('hello ')
buf.write('world')
buf.getvalue()   # 'hello world'
buf.seek(0)
buf.read()       # 'hello world'

# BytesIO — бинарный буфер
img_buf = BytesIO(b'\x89PNG...')
img_buf.seek(0)
data = img_buf.read()
```

---

## Сериализация

### json

Работает только с базовыми типами: `dict`, `list`, `str`, `int`, `float`, `bool`, `None`.

```python
import json

data = {'name': 'Alice', 'age': 30, 'scores': [95, 87, 92]}

# Строка ↔ объект
json_str = json.dumps(data)                          # объект → строка
json_str = json.dumps(data, indent=2, ensure_ascii=False)  # красиво, юникод
obj      = json.loads(json_str)                      # строка → объект

# Файл ↔ объект
with open('data.json', 'w', encoding='utf-8') as f:
    json.dump(data, f, indent=2, ensure_ascii=False) # объект → файл

with open('data.json', encoding='utf-8') as f:
    obj = json.load(f)                               # файл → объект
```

Для нестандартных типов — реализуйте свой `JSONEncoder`:

```python
class DateEncoder(json.JSONEncoder):
    def default(self, obj):
        import datetime
        if isinstance(obj, datetime.date):
            return obj.isoformat()
        return super().default(obj)

json.dumps({'date': datetime.date.today()}, cls=DateEncoder)
```

### pickle

Сериализует практически любые Python-объекты (функции, классы, лямбды). Формат бинарный и Python-специфичный — не для обмена между языками.

```python
import pickle

data = {'key': [1, 2, 3], 'obj': MyClass()}

# Байты ↔ объект
raw   = pickle.dumps(data)    # объект → bytes
obj   = pickle.loads(raw)     # bytes → объект

# Файл ↔ объект (бинарный режим!)
with open('data.pkl', 'wb') as f:
    pickle.dump(data, f)

with open('data.pkl', 'rb') as f:
    obj = pickle.load(f)
```

**Никогда не десериализуйте pickle из ненадёжных источников** — это эквивалентно выполнению произвольного кода.

| | json | pickle |
|---|---|---|
| Формат | текст (читаемый) | бинарный |
| Типы | базовые Python-типы | почти все Python-объекты |
| Скорость | медленнее | быстрее |
| Безопасность | безопасен | небезопасен из ненадёжных источников |
| Кросс-языковой | да | нет |
