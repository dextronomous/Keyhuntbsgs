# KeyHunt: режим `-m bsgs` (BSGS) — документация по ключам (RU)

Этот документ описывает **все ключи командной строки, которые применимы к режиму** `-m bsgs` в текущей версии `keyhunt` (включая новые ключи `-D` и `-w`), а также ограничения и примеры.

> Примечание: в примерах ниже используйте `./keyhunt` (Linux/WSL) или `keyhunt.exe` (Windows).

---

## 1) Быстрый старт

Минимальная команда (BSGS по умолчанию = `sequential`, диапазон берётся из `-b`):

```bash
./keyhunt -m bsgs -f tests/125.txt -b 125
```

Типовая команда для скорости/удобства:

```bash
./keyhunt -m bsgs -f tests/125.txt -b 125 -t 20 -k 64 -s 1 -q
```

---

## 2) Входные данные `-f` (обязательно)

### `-f <file>`
Файл целей для BSGS — это текстовый файл, где **каждая строка содержит публичный ключ в hex**:

- **сжатый pubkey**: 66 символов hex (`02...` или `03...`)
- **несжатый pubkey**: 130 символов hex (`04...`)

В строке допускаются пробелы/доп. токены (берётся первый токен), поэтому можно хранить комментарии справа.

Пример файла `test.txt`:

```text
02f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9
04....(130 hex символов)....
03abcd...  # comment
```

Пример запуска:

```bash
./keyhunt -m bsgs -f test.txt -b 125
```

---

## 3) Подрежимы BSGS (как сканируется диапазон)

### `-B <mode>`
Выбор подрежима BSGS:

- `sequential` — последовательный проход диапазона (по умолчанию)
- `backward` — проход «назад»
- `both` — комбинированный режим
- `random` — случайный перебор внутри диапазона
- `dance` — режим dance

Примеры:

```bash
./keyhunt -m bsgs -f test.txt -b 125 -B sequential
./keyhunt -m bsgs -f test.txt -b 125 -B backward
./keyhunt -m bsgs -f test.txt -b 125 -B both
./keyhunt -m bsgs -f test.txt -b 125 -B random
./keyhunt -m bsgs -f test.txt -b 125 -B dance
```

### `-R`
Включает **Random mode** и принудительно выставляет BSGS-подрежим `random`.

Пример:

```bash
./keyhunt -m bsgs -f test.txt -b 125 -R
```

---

## 4) Как задаётся диапазон поиска

BSGS работает по диапазону приватных ключей.

### `-r <start[:end]>`
Явно задаёт диапазон в hex.

- `-r <start>` — от `start` до `N` (порядка кривой)
- `-r <start>:<end>` — от `start` до `end`

Примеры:

```bash
./keyhunt -m bsgs -f test.txt -r 1:FFFFFFFFFFFFFFFF
./keyhunt -m bsgs -f test.txt -r 4000000000000000:8000000000000000
./keyhunt -m bsgs -f test.txt -r 20000000000000000
```

### `-b <bits>`
Задаёт **битовый диапазон** (1..256). Это удобный способ для puzzle-диапазонов.

Пример (puzzle 125):

```bash
./keyhunt -m bsgs -f tests/125.txt -b 125
```

### `-D <deep_file>` (новое)
Включает чтение диапазонов из `deep`-файла. Формат: `start_hex:end_hex` по одному диапазону в строке.

Поведение:

- Если `-m bsgs` и `-B sequential`, то программа **пройдёт все диапазоны** из файла (run-all), по очереди.
- Для других режимов (не BSGS, либо BSGS не `sequential`) используется «старое» поведение: берётся первый валидный диапазон.

Пример `deep.txt`:

```text
4000000000000000:7fffffffffffffff
8000000000000000:ffffffffffffffff
```

Пример запуска:

```bash
./keyhunt -m bsgs -f test.txt -B sequential -D deep.txt -t 16 -k 64 -s 1 -q
```

После каждого диапазона в режиме run-all запись добавляется в `checked-deep.txt`.

### `-w <WIF_mask>` (новое)
Автоматическая генерация `deep.txt` по **маске WIF**, где символ `_` означает «неизвестный символ» (wildcard).

Ограничения:

- Работает **только** с `-m bsgs` и `-B sequential`
- Нельзя использовать вместе с `-r` или `-b`

Пример:

```bash
./keyhunt -m bsgs -f test.txt -w KwrFSkr6KgLDmbRn6_f3uri5sDyWxXC5437__________aCR1At2 -k 64 -B sequential -s 1 -t 20
```

Что происходит:

- создаётся `deep.txt` с диапазонами
- затем запускается обработка всех диапазонов из `deep.txt`
- в прогрессе подсвечиваются символы WIF на позициях, где в маске стоял `_`

---

## 5) Производительность и память

### `-k <factor>`
K-фактор для BSGS. Как правило, **увеличение `-k` ускоряет**, но может влиять на память/время подготовки таблиц.

Пример:

```bash
./keyhunt -m bsgs -f test.txt -b 125 -k 64
```

### `-n <N>`
Задаёт параметр `N` для BSGS (длина цикла/область работы алгоритма).

Допустимые форматы:

- `-n 1099511627776` (десятичное)
- `-n 0x100000000000` (hex)

Важные требования:

- `N` должен иметь **точный квадратный корень** (иначе программа завершится с ошибкой)
- `M = sqrt(N)` должен быть кратен **1024** (иначе будет ошибка)

Пример:

```bash
./keyhunt -m bsgs -f test.txt -b 125 -n 0x100000000000 -k 64 -t 20
```

### `-t <threads>`
Количество потоков.

Пример:

```bash
./keyhunt -m bsgs -f test.txt -b 125 -t 20
```

### `-z <multiplier>`
Множитель размера bloom-фильтров.

- `-z 1` — по умолчанию
- увеличение может уменьшить вероятность ложных срабатываний, но увеличит память/файлы

Пример:

```bash
./keyhunt -m bsgs -f test.txt -b 125 -z 2 -k 64
```

---

## 6) Вывод, прогресс, файлы

### `-s <seconds>`
Период вывода статистики.

- `-s 0` — отключить статистику

Примеры:

```bash
./keyhunt -m bsgs -f test.txt -b 125 -s 1
./keyhunt -m bsgs -f test.txt -b 125 -s 10
./keyhunt -m bsgs -f test.txt -b 125 -s 0
```

### `-q`
Quiet: отключает «болтливый» вывод потоков (полезно при большом числе потоков).

Пример:

```bash
./keyhunt -m bsgs -f test.txt -b 125 -q -s 1
```

### `-M`
Matrix screen (экран/формат вывода матрицей). Удобно для визуального мониторинга.

Пример:

```bash
./keyhunt -m bsgs -f test.txt -b 125 -M
```

### `-S`
Read/Save режим для файлов: при запуске программа старается **читать** заранее сохранённые файлы (bloom/table), и при необходимости **сохранять** их.

Пример:

```bash
./keyhunt -m bsgs -f test.txt -b 125 -S -k 64
```

### `-6`
Пропустить проверку SHA256 checksum при чтении файлов.

Пример:

```bash
./keyhunt -m bsgs -f test.txt -b 125 -S -6
```

### `-d`
Включить debug-вывод.

Пример:

```bash
./keyhunt -m bsgs -f test.txt -b 125 -d
```

---

## 7) Ключи, которые НЕ совместимы с BSGS (важно)

### `-e`
Endomorphism **не работает** с BSGS (программа завершится с ошибкой).

Пример (так делать нельзя):

```bash
./keyhunt -m bsgs -f test.txt -b 125 -e
```

### `-I <stride>`
Stride **не работает** с BSGS (программа завершится с ошибкой).

Пример (так делать нельзя):

```bash
./keyhunt -m bsgs -f test.txt -b 125 -I 2
```

### `-l <compress|uncompress|both>`
Для BSGS этот ключ **не является основным параметром** поиска: сжатость берётся из входных публичных ключей (`-f`).

Если хочешь искать по сжатому pubkey — положи в файл `-f` сжатый pubkey.

---

## 8) Какие файлы создаёт BSGS

В процессе работы/подготовки BSGS создаёт и читает файлы (названия зависят от `N`, `k`, и внутренних параметров):

- `keyhunt_bsgs_4_*.blm` (bloom)
- `keyhunt_bsgs_6_*.blm` (bloom)
- `keyhunt_bsgs_7_*.blm` (3rd bloom)
- `keyhunt_bsgs_2_*.tbl` (таблица bP)

Дополнительно для deep/wif-mask:

- `deep.txt` — диапазоны `start:end`
- `checked-deep.txt` — журнал обработанных диапазонов (start/end + WIF + pubkeys)

---

## 9) Примеры «по задачам»

### Поиск по одному диапазону (bitrange)

```bash
./keyhunt -m bsgs -f tests/125.txt -b 125 -t 20 -k 64 -s 1 -q
```

### Random BSGS по bitrange

```bash
./keyhunt -m bsgs -f tests/125.txt -b 125 -R -t 20 -k 64 -s 1 -q
```

### Сканирование набора диапазонов (deep.txt)

```bash
./keyhunt -m bsgs -f test.txt -B sequential -D deep.txt -t 20 -k 64 -s 1 -q
```

### Сканирование по маске WIF (генерация deep.txt автоматически)

```bash
./keyhunt -m bsgs -f test.txt -w KwrFSkr6KgLDmbRn6_f3uri5sDyWxXC5437__________aCR1At2 -B sequential -k 64 -t 20 -s 1 -q
```
