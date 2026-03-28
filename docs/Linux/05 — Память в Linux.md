---
tags: [linux, devops, memory, ram, swap, buffer, cache, oom, virtual-memory]
---

# Память в Linux

## Виды памяти

Linux использует сложную иерархию памяти:

```
CPU Registers → L1 Cache → L2 Cache → L3 Cache → RAM → Swap (диск)
   быстрее ←                                              → медленнее
```

---

## Команда free: разбор вывода

```bash
free -m    # в мегабайтах
free -h    # human-readable
free -s 2  # обновлять каждые 2 секунды
```

```
              total    used    free   shared  buff/cache  available
Mem:           6930    3598     843      183        2489       2919
Swap:         15999       4   15995
```

| Поле | Описание | Формула |
|------|----------|---------|
| `total` | Вся физическая RAM | — |
| `used` | Занята процессами | `total - free - buff/cache` |
| `free` | Не используется совсем | — |
| `shared` | Разделяемая память (tmpfs, IPC) | — |
| `buff/cache` | Буферы + кэш файловой системы | — |
| `available` | Доступно для новых процессов | `free + освобождаемый кэш` |

### Почему available > free?

`available` = `free` + часть `buff/cache`, которую ядро готово отдать.

Ядро активно использует свободную память под кэш файлов (Page Cache), ускоряя доступ к файлам. При нехватке памяти — кэш освобождается в первую очередь.

---

## Buffer и Cache

### Page Cache (cached)

Ядро кэширует содержимое файлов в RAM. При повторном обращении к файлу — данные берутся из памяти, а не с диска.

```bash
# Посмотреть использование Page Cache
cat /proc/meminfo | grep -E 'Cached|Buffers'

# Принудительно сбросить кэш (для тестов)
sync && echo 3 > /proc/sys/vm/drop_caches
# 1 = только page cache
# 2 = dentries и inodes
# 3 = всё вышеперечисленное
```

### Buffers

Буферы ядра — страницы памяти, зарезервированные для операций ввода-вывода. Метаданные файловой системы, блочные устройства.

---

## Swap (Раздел подкачки)

Swap — область на диске, куда ядро вытесняет неиспользуемые страницы памяти (page out).

### Зачем нужен swap

- Позволяет системе не падать при исчерпании RAM
- Ядро вытесняет редко используемые страницы
- Освобождает RAM для активных процессов

### Swap не замена RAM

Диск в 100-1000 раз медленнее RAM. Активное использование swap = деградация производительности.

### Настройка swap

```bash
# Посмотреть swap
swapon --show
free -h

# Swappiness — агрессивность использования swap (0-100)
cat /proc/sys/vm/swappiness   # обычно 60
sysctl vm.swappiness=10        # применить сейчас
echo "vm.swappiness=10" >> /etc/sysctl.conf  # постоянно

# 0  = использовать swap только при критическом недостатке RAM
# 10 = рекомендуется для серверов
# 60 = значение по умолчанию
# 100 = активно использовать swap
```

### Создание swap-файла

```bash
# Создать файл
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# Добавить в fstab
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

---

## /proc/meminfo

Детальная информация о памяти:

```bash
cat /proc/meminfo
```

Ключевые поля:

| Поле | Описание |
|------|----------|
| `MemTotal` | Вся RAM |
| `MemFree` | Свободная |
| `MemAvailable` | Доступна для новых процессов |
| `Buffers` | Буферы ядра |
| `Cached` | Page Cache |
| `SwapTotal/Free` | Swap |
| `Dirty` | Страницы, ожидающие записи на диск |
| `Writeback` | Страницы, записываемые прямо сейчас |
| `Slab` | Аллокации ядра (dcache, inode cache) |
| `HugePages_Total` | Huge Pages |

---

## OOM Killer (Out of Memory Killer)

Когда RAM и swap исчерпаны, ядро запускает OOM Killer — убивает процесс с наибольшим `oom_score`.

```bash
# Посмотреть oom_score процессов
cat /proc/PID/oom_score

# Защитить процесс от OOM Killer
echo -1000 > /proc/PID/oom_score_adj   # min = -1000 (защищён)

# Настройка поведения при OOM
# 0 = OOM Killer (по умолчанию)
# 1 = ядро паникует
cat /proc/sys/vm/panic_on_oom

# Логи OOM
dmesg | grep -i 'oom\|killed process'
journalctl -k | grep -i oom
```

---

## Virtual Memory и адресное пространство

Каждый процесс работает в **виртуальном адресном пространстве** (изоляция, 64-битная = 128 ТБ).

```bash
# Карта памяти процесса
cat /proc/PID/maps
pmap -x PID

# Общая память процесса
ps -o pid,vsz,rss,comm -p PID
# VSZ = виртуальный размер
# RSS = реальный размер в RAM (Resident Set Size)
```

### VSZ vs RSS

- **VSZ** — виртуальная память (включая незагруженные страницы, shared libs)
- **RSS** — реально занятая физическая RAM

---

## Huge Pages

Обычные страницы памяти = 4 КБ. Huge Pages = 2 МБ или 1 ГБ.

Используются для: базы данных (PostgreSQL, Oracle), виртуализации (KVM/QEMU).

```bash
# Посмотреть Huge Pages
cat /proc/meminfo | grep Huge
hugeadm --pool-list

# Настройка (2 МБ Huge Pages)
sysctl vm.nr_hugepages=1024
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```

---

## Чеклист при утечке памяти / нехватке RAM

```bash
# 1. Общая картина
free -h
top / htop

# 2. Топ процессов по памяти
ps aux --sort=-%mem | head -20
ps -eo pid,ppid,%mem,rss,vsz,comm --sort=-%mem | head -20

# 3. Детали конкретного процесса
pmap -x PID | tail -1   # итого по процессу

# 4. OOM события
dmesg | grep -i oom
journalctl -k -b | grep -i oom

# 5. Утечки slab (ядро)
cat /proc/meminfo | grep Slab
slabtop

# 6. Детальный meminfo
cat /proc/meminfo

# 7. Если нужно срочно освободить кэш
sync && echo 1 > /proc/sys/vm/drop_caches
```
