# Тема 14: SSH и удалённый доступ

## 14.1. Симметричное vs асимметричное шифрование

### Симметричное:
```
один ключ для шифрования и расшифровки
проблема — как безопасно передать ключ?
плюс — быстрое
```

### Асимметричное:
```
пара математически связанных ключей:
приватный — только у тебя, никому не давать
публичный — можно всем раздавать

зашифровано публичным → расшифровать только приватным
подписано приватным → проверить только публичным
```

Зная публичный ключ — нельзя восстановить приватный (задача факторизации больших чисел).

### Как SSH использует оба:
```
1. Подключение → асимметричное (обмен Диффи-Хеллмана)
   → генерируется временный симметричный ключ сессии

2. Весь трафик → симметричное (AES) — быстрее

3. Аутентификация → асимметричное
   → сервер проверяет твой приватный ключ через публичный
```

---

## 14.2. Отключить аутентификацию по паролю

```bash
# /etc/ssh/sshd_config:
PasswordAuthentication no
PermitRootLogin no           # запретить вход под root
PubkeyAuthentication yes     # разрешить вход по ключу
Port 2222                    # нестандартный порт
MaxAuthTries 3               # максимум попыток
AllowUsers god vasya         # только эти пользователи
AllowGroups sshusers         # только эта группа
ClientAliveInterval 300      # keepalive каждые 5 минут

systemctl reload sshd        # применить без разрыва соединений
```

**Важно:** перед отключением убедиться что SSH ключ работает!

---

## 14.3. ssh-agent

Хранит приватные ключи в памяти в расшифрованном виде:

```bash
eval $(ssh-agent)         # запустить agent
ssh-add ~/.ssh/id_rsa     # добавить ключ (passphrase один раз)
ssh-add -l                # список загруженных ключей
ssh-add -d ~/.ssh/id_rsa  # удалить ключ
ssh-add -D                # удалить все ключи
```

Без agent — вводить passphrase при каждом подключении.
С agent — один раз, дальше автоматически.

Ключи только в памяти — при завершении сессии исчезают.

---

## 14.4 и 14.10. Проброс портов (туннели)

```bash
# Local (-L) — локальный порт → удалённый сервис:
ssh -L 5432:db.internal:5432 user@bastion
# localhost:5432 → bastion → db.internal:5432

# Remote (-R) — удалённый порт → локальный сервис:
ssh -R 8080:localhost:3000 user@server
# server:8080 → твой localhost:3000

# Dynamic (-D) — SOCKS прокси:
ssh -D 1080 user@server
# браузер → SOCKS5 localhost:1080 → весь трафик через server
```

Конкретно 8080→80 через промежуточный сервер:
```bash
ssh -L 8080:remote_host:80 user@bastion
# или через ProxyJump:
ssh -J bastion -L 8080:remote_host:80 user@remote_host
```

---

## 14.5. known_hosts

```bash
~/.ssh/known_hosts   # публичные ключи известных серверов
```

При первом подключении — SSH спрашивает подтвердить fingerprint сервера → сохраняет в known_hosts.

При следующих подключениях — проверяет ключ сервера:
```
совпадает → всё ок
изменился → WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

Изменившийся ключ = переустановили сервер (нормально) или MITM атака (опасно).

```bash
ssh-keygen -R server.com   # удалить запись (после переустановки)
```

### Защита от MITM:
```
без known_hosts:
ты → злоумышленник (притворяется сервером) → видит весь трафик

с known_hosts:
ключ изменился → WARNING → проверить перед вводом пароля
```

---

## 14.6. Клиентский блок конфигурации

```bash
# ~/.ssh/config:
Host myserver
    HostName server.example.com
    Port 2222
    User god
    IdentityFile ~/.ssh/id_rsa_server

Host bastion
    HostName 1.2.3.4
    User admin
    IdentityFile ~/.ssh/bastion_key

Host internal
    HostName 10.0.0.5
    ProxyJump bastion         # подключиться через bastion

Host *
    ServerAliveInterval 60    # keepalive каждые 60 секунд
    ServerAliveCountMax 3
```

```bash
ssh myserver   # вместо: ssh -p 2222 -i ~/.ssh/id_rsa_server god@server.example.com
```

---

## 14.7. Риски agent forwarding

```bash
ssh -A user@server   # включить agent forwarding
```

Твой агент доступен на удалённом сервере — можно подключаться дальше без пароля.

**Риск:**
```
скомпрометированный сервер → root использует твой агент → подключается к другим серверам от твоего имени
приватный ключ не виден, но агент можно использовать
```

Безопаснее — `ProxyJump`:
```bash
ssh -J bastion internal   # агент не форвардится на bastion
```

---

## 14.8. Скопировать публичный ключ на сервер

```bash
ssh-copy-id user@server
ssh-copy-id -i ~/.ssh/id_rsa.pub user@server

# вручную:
cat ~/.ssh/id_rsa.pub | ssh user@server "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

Права которые должны быть:
```
~/.ssh/              — 700
~/.ssh/authorized_keys — 600
```

SSH откажется работать если права слишком открытые.

---

## 14.9. Аутентификация по SSH ключам

### Клиентская сторона:
```
~/.ssh/id_rsa        — приватный ключ
~/.ssh/id_rsa.pub    — публичный ключ
~/.ssh/config        — конфигурация подключений
~/.ssh/known_hosts   — известные серверы
```

### Серверная сторона:
```
~/.ssh/authorized_keys — публичные ключи разрешённых пользователей
/etc/ssh/sshd_config   — конфигурация SSH сервера
```

### Процесс аутентификации:
```
1. клиент подключается
2. сервер смотрит authorized_keys
3. сервер шифрует случайное число публичным ключом клиента
4. клиент расшифровывает приватным ключом
5. клиент отправляет хэш расшифрованного числа
6. сервер проверяет → совпало → доступ разрешён
```

```bash
# создать пару ключей:
ssh-keygen -t ed25519 -C "god@mypc"   # современный (рекомендуется)
ssh-keygen -t rsa -b 4096             # классический RSA
```

---

## 14.11. Ограничить SSH доступ по IP

```bash
# /etc/ssh/sshd_config:
AllowUsers god@192.168.1.100   # только с этого IP

# Match блок:
Match User god Address 192.168.1.100
    AllowTcpForwarding yes

# TCP Wrappers:
# /etc/hosts.allow:
sshd: 192.168.1.100
# /etc/hosts.deny:
sshd: ALL

# iptables — надёжнее всего:
iptables -A INPUT -p tcp --dport 22 -s 192.168.1.100 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j DROP
```
