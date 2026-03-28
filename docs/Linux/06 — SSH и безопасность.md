---
tags: [linux, devops, ssh, security, sshd, keys, firewall, permissions, hardening]
---

# SSH и безопасность в Linux

## SSH: основы

**SSH (Secure Shell)** — протокол для безопасного удалённого управления сервером поверх зашифрованного канала.

### Конфигурационные файлы

| Файл | Описание |
|------|----------|
| `/etc/ssh/sshd_config` | Конфигурация **сервера** SSH |
| `/etc/ssh/ssh_config` | Конфигурация **клиента** SSH (глобальная) |
| `~/.ssh/config` | Конфигурация клиента (пользователя) |
| `~/.ssh/authorized_keys` | Публичные ключи, разрешённые для входа |
| `~/.ssh/known_hosts` | Известные хосты (защита от MITM) |
| `~/.ssh/id_rsa` | Приватный ключ пользователя |
| `~/.ssh/id_rsa.pub` | Публичный ключ пользователя |

---

## sshd_config: важные параметры

```bash
# Открыть конфигурацию
vim /etc/ssh/sshd_config

# Применить изменения
systemctl reload sshd
```

### Отключение аутентификации по паролю

```ini
# /etc/ssh/sshd_config
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM no
```

> Обязательно добавьте свой публичный ключ в `~/.ssh/authorized_keys` перед отключением паролей!

### Базовый харденинг sshd_config

```ini
# Порт (скрытость от автосканеров)
Port 2222

# Только IPv4/IPv6
AddressFamily inet

# Запрет root-логина
PermitRootLogin no

# Только ключи
PasswordAuthentication no
PubkeyAuthentication yes

# Ограничение по пользователям/группам
AllowUsers deploy
AllowGroups sshusers

# Таймаут бездействия
ClientAliveInterval 300
ClientAliveCountMax 2

# Отключить X11 и агентский форвардинг если не нужны
X11Forwarding no
AllowAgentForwarding no

# Ограничить попытки аутентификации
MaxAuthTries 3

# Banner
Banner /etc/ssh/banner.txt
```

---

## SSH ключи

### Генерация ключей

```bash
# Современный алгоритм (рекомендуется)
ssh-keygen -t ed25519 -C "comment" -f ~/.ssh/id_ed25519

# RSA (совместимость со старыми системами)
ssh-keygen -t rsa -b 4096 -C "comment"

# С passphrase для защиты приватного ключа
ssh-keygen -t ed25519 -N "mypassphrase"
```

### Копирование ключа на сервер

```bash
ssh-copy-id user@host
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host

# Вручную
cat ~/.ssh/id_ed25519.pub | ssh user@host "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

### Права доступа для SSH

SSH строгий к правам:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
```

### ~/.ssh/config: алиасы и настройки

```ini
Host myserver
    HostName 192.168.1.100
    User deploy
    Port 2222
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 60

Host bastion
    HostName bastion.example.com
    User ubuntu

Host internal
    HostName 10.0.0.5
    User ubuntu
    ProxyJump bastion
```

```bash
ssh myserver    # использует настройки из config
```

---

## SSH туннели

```bash
# Local Port Forwarding (доступ к удалённому сервису локально)
ssh -L 8080:remote-db:5432 user@bastion
# localhost:8080 → bastion → remote-db:5432

# Remote Port Forwarding (публикация локального сервиса)
ssh -R 8080:localhost:3000 user@server
# server:8080 → localhost:3000

# Dynamic (SOCKS proxy)
ssh -D 1080 user@server
```

---

## Права доступа к файлам (Linux permissions)

### Формат прав

```
-rwxr-xr--  1 user group  4096 Jan 1 00:00 file
│└──┬──┘└──┬──┘└──┬──┘
│   │      │      └── Остальные (others)
│   │      └───────── Группа (group)
│   └──────────────── Владелец (user)
└──────────────────── Тип файла
```

### Числовое представление

| Право | Значение | Файл | Директория |
|-------|----------|------|-----------|
| `r` | 4 | Чтение | Просмотр содержимого |
| `w` | 2 | Запись | Создание/удаление файлов |
| `x` | 1 | Выполнение | Вход в директорию |

```bash
# Примеры
chmod 644 file    # rw-r--r--  (файл: владелец читает/пишет, остальные читают)
chmod 755 dir     # rwxr-xr-x  (директория: все могут войти, только владелец писать)
chmod 600 secret  # rw-------  (только владелец)
chmod 400 key.pem # r--------  (только чтение владельцем)

# Рекурсивно
chmod -R 755 /var/www

# Буквенное задание
chmod u+x script.sh      # добавить execute для владельца
chmod g-w file           # убрать write у группы
chmod o= file            # убрать все права у остальных
chmod a+r file           # добавить read для всех
```

### Специальные биты

```bash
# SUID (4000) — запуск от имени владельца файла
chmod u+s /usr/bin/passwd
ls -la /usr/bin/passwd   # -rwsr-xr-x

# SGID (2000) — запуск от имени группы файла / новые файлы наследуют группу
chmod g+s /shared/dir

# Sticky bit (1000) — только владелец может удалить свой файл
chmod +t /tmp
ls -la /tmp   # drwxrwxrwt
```

---

## Пользователи и сессии

```bash
# Кто сейчас в системе
w            # детальная информация: время, нагрузка, откуда зашли
who          # краткий список
last         # история входов
lastb        # неудачные попытки входа
lastlog      # последний вход каждого пользователя

# Информация о пользователе
id username
groups username
```

---

## Управление пользователями

```bash
# Создание пользователя
useradd -m -s /bin/bash -G sudo username
passwd username

# Изменение
usermod -aG docker username    # добавить в группу
usermod -s /bin/bash username  # изменить shell
usermod -L username            # заблокировать
usermod -U username            # разблокировать

# Удаление
userdel -r username            # с домашней директорией

# Важные файлы
/etc/passwd    # пользователи
/etc/shadow    # хэши паролей (только root)
/etc/group     # группы
/etc/sudoers   # sudo права
```

### sudo

```bash
# Редактировать sudoers безопасно
visudo

# Разрешить пользователю без пароля
username ALL=(ALL) NOPASSWD: ALL

# Разрешить конкретные команды
deploy ALL=(ALL) NOPASSWD: /bin/systemctl restart nginx, /bin/systemctl status *
```

---

## Firewall: iptables и nftables

### iptables

```bash
# Посмотреть правила
iptables -L -n -v

# Разрешить SSH
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Разрешить установленные соединения
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Закрыть всё остальное
iptables -A INPUT -j DROP

# Сохранить правила
iptables-save > /etc/iptables/rules.v4
```

### ufw (Ubuntu)

```bash
ufw enable
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw deny 3306/tcp
ufw status verbose
```

### firewalld (RHEL/CentOS)

```bash
firewall-cmd --state
firewall-cmd --list-all
firewall-cmd --add-service=http --permanent
firewall-cmd --add-port=8080/tcp --permanent
firewall-cmd --reload
```

---

## Fail2ban

Автоматически блокирует IP-адреса с подозрительной активностью (брутфорс SSH, nginx и др.).

```bash
# Установка
apt install fail2ban

# Статус
fail2ban-client status
fail2ban-client status sshd

# Разбанить IP
fail2ban-client set sshd unbanip 1.2.3.4
```

Конфиг `/etc/fail2ban/jail.local`:
```ini
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 600
```
