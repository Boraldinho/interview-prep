---
tags: [linux, devops, boot, grub, kernel, systemd, initramfs, bios, uefi]
---

# Загрузка Linux

## Полная последовательность загрузки

```
Power ON
    │
    ▼
BIOS / UEFI (firmware)
    │  POST: тест железа, поиск загрузчика
    ▼
MBR / GPT + Bootloader (GRUB2)
    │  Загружает ядро и initramfs
    ▼
Kernel (vmlinuz)
    │  Инициализация железа, монтирование rootfs
    ▼
initramfs (временная rootfs в памяти)
    │  Монтирование настоящего /
    ▼
init / systemd (PID 1)
    │  Запуск сервисов, цели (targets)
    ▼
Login prompt / Display Manager
```

---

## Этап 1: BIOS / UEFI

### BIOS (Legacy)

- **POST** (Power-On Self Test): проверка RAM, CPU, видеокарты, дисков
- Ищет MBR (Master Boot Record) на загрузочном диске
- MBR = первые 512 байт диска, содержит первичный загрузчик

### UEFI (современный стандарт)

- Хранится в NVRAM (энергонезависимая память)
- Поддерживает GPT-разделы (до 128 разделов, диски > 2 ТБ)
- **ESP (EFI System Partition)** — специальный FAT32-раздел с загрузчиками (`/boot/efi`)
- **Secure Boot** — проверяет подпись загрузчика и ядра
- Быстрее BIOS, поддерживает сетевую загрузку (PXE)

```bash
# Определить BIOS или UEFI
ls /sys/firmware/efi    # если существует — UEFI

# Управление UEFI записями
efibootmgr -v
```

---

## Этап 2: GRUB2 (Загрузчик)

**GRUB2 (Grand Unified Bootloader v2)** — стандартный загрузчик Linux.

### Расположение файлов

```
/boot/grub2/grub.cfg       # основной конфиг (не редактировать напрямую)
/etc/default/grub          # параметры для генерации конфига
/etc/grub.d/               # скрипты генерации меню
/boot/vmlinuz-*            # образы ядра
/boot/initramfs-*          # образы initramfs
```

### Управление GRUB

```bash
# Обновить конфиг после изменений
grub2-mkconfig -o /boot/grub2/grub.cfg   # RHEL
update-grub                               # Debian/Ubuntu

# Параметры в /etc/default/grub
GRUB_DEFAULT=0              # загружать первое меню
GRUB_TIMEOUT=5              # ждать 5 секунд
GRUB_CMDLINE_LINUX="quiet splash"  # параметры ядра
```

### Что делает GRUB

1. Показывает меню выбора ядра
2. Загружает ядро (`vmlinuz`) в память
3. Загружает `initramfs` в память
4. Передаёт управление ядру

---

## Этап 3: Ядро Linux (Kernel)

### Что делает ядро при старте

1. Распаковывает себя (сжатый образ)
2. Обнаруживает и инициализирует оборудование
3. Настраивает подсистему памяти (MMU, paging)
4. Монтирует `initramfs` как временную rootfs
5. Запускает `/init` или `/sbin/init` из initramfs

### Параметры ядра

Передаются через GRUB в строке `GRUB_CMDLINE_LINUX`:

| Параметр | Описание |
|----------|----------|
| `quiet` | Минимальный вывод при загрузке |
| `ro` | Смонтировать root только для чтения |
| `init=/bin/bash` | Запустить bash вместо init (emergency) |
| `single` / `1` | Однопользовательский режим |
| `nomodeset` | Отключить mode setting (проблемы с видео) |
| `mem=2G` | Ограничить используемую RAM |

```bash
# Текущие параметры ядра
cat /proc/cmdline

# Версия ядра
uname -r
uname -a

# Загруженные модули ядра
lsmod
modprobe module_name   # загрузить модуль
modprobe -r module_name  # выгрузить
```

---

## Этап 4: initramfs

**initramfs** (initial RAM filesystem) — временная мини-файловая система, загружаемая в память.

### Зачем нужна

Представь ситуацию: ядро загружено в память и хочет смонтировать диск, чтобы запустить систему. Но чтобы работать с диском — нужен драйвер. А драйвер лежит на том самом диске, который ещё не смонтирован. Замкнутый круг.

Ещё сложнее если диск зашифрован (LUKS), или это LVM-том, или программный RAID — там нужны дополнительные инструменты, которых в ядре нет.

**initramfs разрывает этот круг:**

```
GRUB загружает в память:
  1. ядро (vmlinuz)
  2. initramfs — маленький архив с нужными драйверами и утилитами

Ядро стартует → монтирует initramfs как временный / в памяти
→ находит нужные драйверы → монтирует настоящий диск
→ переключает / на настоящий диск (pivot_root)
→ запускает systemd
→ initramfs выбрасывается из памяти
```

Это как взять с собой маленький рюкзак с инструментами чтобы открыть склад, где лежат все остальные инструменты.

### Что содержит initramfs

- Необходимые модули ядра (драйверы дисков, ФС)
- `udev` для обнаружения устройств
- Скрипты монтирования LVM/RAID/LUKS
- Минимальный userspace (`busybox`)

```bash
# Список файлов в initramfs
lsinitrd /boot/initramfs-$(uname -r).img   # RHEL
unmkinitramfs /boot/initrd.img /tmp/initrd  # Debian

# Пересоздать initramfs
dracut --force                              # RHEL
update-initramfs -u                         # Debian
mkinitcpio -P                              # Arch
```

---

## Этап 5: systemd (PID 1)

После монтирования настоящей rootfs ядро запускает `/sbin/init` (обычно симлинк на `systemd`).

### Systemd targets (аналог runlevels)

| Target | SysV runlevel | Описание |
|--------|---------------|----------|
| `poweroff.target` | 0 | Выключение |
| `rescue.target` | 1 | Однопользовательский режим |
| `multi-user.target` | 2,3,4 | Многопользовательский без GUI |
| `graphical.target` | 5 | Многопользовательский с GUI |
| `reboot.target` | 6 | Перезагрузка |

```bash
# Текущий target
systemctl get-default
systemctl list-units --type=target

# Изменить target по умолчанию
systemctl set-default multi-user.target

# Переключиться сейчас (без перезагрузки)
systemctl isolate rescue.target

# Аварийный режим
systemctl isolate emergency.target
```

### Порядок запуска systemd

```
systemd
    │
    ├─ sysinit.target (монтирование ФС, запуск udev)
    │
    ├─ basic.target (базовые сервисы)
    │
    ├─ multi-user.target
    │   ├─ network.target
    │   ├─ sshd.service
    │   ├─ nginx.service
    │   └─ ...
    │
    └─ graphical.target (если нужен GUI)
```

### Время загрузки

```bash
# Общее время загрузки
systemd-analyze

# Разбивка по сервисам
systemd-analyze blame

# Граф зависимостей
systemd-analyze plot > boot.svg
```

---

## Однопользовательский режим и аварийное восстановление

```bash
# Перезагрузить в rescue (нужен пароль root)
systemctl reboot --boot-loader-entry=rescue

# Через GRUB: добавить в строку ядра
# init=/bin/bash
# или: systemd.unit=rescue.target

# Если ФС повреждена: в initramfs shell
# Добавить rd.break к параметрам ядра в GRUB
```

### Сброс пароля root (через GRUB)

1. В меню GRUB нажать `e`
2. В строке `linux` добавить `rd.break` (RHEL) или `init=/bin/bash` (универсально)
3. Нажать `Ctrl+X`
4. В initramfs shell:
```bash
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel    # SELinux: перемаркировка
exit && reboot
```

---

## Журналы загрузки

```bash
# Системный журнал
journalctl -b            # текущая загрузка
journalctl -b -1         # предыдущая загрузка
journalctl -b -p err     # только ошибки

# Сообщения ядра
dmesg
dmesg | grep -i error
dmesg -T               # с временными метками

# Старый syslog
/var/log/syslog        # Debian
/var/log/messages      # RHEL
```
