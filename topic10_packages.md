# Тема 10: Управление пакетами (APT и RPM/YUM/DNF)

## 10.1. Низкоуровневые vs высокоуровневые инструменты

### Низкоуровневые — работают с конкретным файлом напрямую:
```bash
dpkg -i package.deb    # Debian/Ubuntu
rpm -i package.rpm     # RHEL/CentOS
```

Не знают о зависимостях — просто распаковывают и устанавливают.

### Высокоуровневые — работают с репозиториями:
```bash
apt install nginx      # Debian/Ubuntu
dnf install nginx      # RHEL/CentOS
```

Под капотом apt делает:
1. Идёт в репозиторий (список в `/etc/apt/sources.list`)
2. Смотрит индекс пакетов
3. Скачивает nginx + все зависимости
4. Устанавливает через dpkg

```
apt/dnf = менеджер который идёт в репо, берёт нужное и всё что к нему прилагается
dpkg/rpm = просто установщик конкретного файла
```

### Когда нужны низкоуровневые:
- Нет в репозитории (скачал .deb с сайта разработчика)
- Нет интернета (air-gapped сервер)
- Конкретная старая версия которой нет в репо
- Диагностика — посмотреть что внутри пакета

```bash
# если dpkg не установил зависимости:
dpkg -i myapp.deb      # упало
apt install -f         # -f = fix, доустановить зависимости

# перекинуть на другую машину без интернета:
apt download nginx     # скачать .deb без установки
scp nginx.deb user@machineB:/tmp/
dpkg -i /tmp/nginx.deb
```

### Полезные команды dpkg:
```bash
dpkg -l nginx          # статус пакета (ii=installed, rc=removed+configs, un=not installed)
dpkg -L nginx          # какие файлы установил пакет
dpkg -S /usr/bin/nginx # какому пакету принадлежит файл
```

---

## 10.2. Роль GPG ключей

GPG ключи защищают от подделки пакетов:
```
разработчик подписывает пакет приватным ключом
apt проверяет подпись публичным ключом
если не совпадает → отказывается устанавливать
```

Ключ выглядит как текстовый блок в base64:
```
-----BEGIN PGP PUBLIC KEY BLOCK-----
mQENBF6at6oBCAC6RUGH...
-----END PGP PUBLIC KEY BLOCK-----
```

Не каждое приложение — каждый **репозиторий** имеет свой ключ:
```
Ubuntu repo   → один ключ для всех пакетов
Docker repo   → свой ключ
Nginx repo    → свой ключ
```

### Добавить GPG ключ:
```bash
# современный способ:
curl -fsSL https://nginx.org/keys/nginx_signing.key | gpg --dearmor -o /etc/apt/trusted.gpg.d/nginx.gpg
```

Флаги curl:
```
-f   # fail — не выводить HTML ошибки
-s   # silent — без прогресса
-S   # Show error — показать ошибку если -s включён
-L   # Location — следовать редиректам
```

`gpg --dearmor` — конвертировать из текстового base64 формата в бинарный (apt хочет бинарный).

Ключи хранятся в `/etc/apt/trusted.gpg.d/`.

---

## 10.3 и 10.9. Найти пакет по файлу

```bash
# Debian/Ubuntu:
dpkg -S /usr/bin/nginx
dpkg -S /etc/ssh/sshd_config
# → openssh-server: /etc/ssh/sshd_config

# через apt-file (ищет даже в неустановленных):
apt-file search /usr/bin/nginx

# RHEL/CentOS:
rpm -qf /usr/bin/nginx
rpm -qf /etc/ssh/sshd_config
```

`-S` = search, `-q` = query, `-f` = file.

---

## 10.4. Закрепление (pinning/holding) пакетов

```bash
# Debian/Ubuntu:
apt-mark hold nginx        # заморозить версию
apt-mark unhold nginx      # разморозить
apt-mark showhold          # список замороженных

# RHEL/CentOS:
dnf versionlock add nginx
dnf versionlock delete nginx
```

### Зачем на продакшне:
```
обновление сломало конфиг → откатили → заморозили
→ apt upgrade не обновит снова автоматически
→ починили → разморозили
```

### Откатить версию:
```bash
apt-cache policy nginx              # посмотреть доступные версии
apt install nginx=1.18.0-0ubuntu1   # установить конкретную версию
```

Если версии нет в репо — нужен .deb файл скачанный заранее.

---

## 10.5. Удалить ненужные зависимости

```bash
apt autoremove           # Debian/Ubuntu
apt autoremove --dry-run # симуляция
dnf autoremove           # RHEL/CentOS
```

Удаляет пакеты установленные автоматически как зависимости но больше не нужные:
```
установили nginx → автоматически libpcre3
удалили nginx → libpcre3 висит
apt autoremove → удалит libpcre3
```

---

## 10.6. Remove vs Purge

```bash
apt remove nginx    # удалить пакет, конфиги остаются в /etc/nginx/
apt purge nginx     # удалить пакет + все конфиги
```

```bash
dpkg -l nginx
# ii — installed
# rc — removed but configs remain
# un — not installed
```

`remove` — если планируешь переустановить (конфиги сохранятся).
`purge` — чистая переустановка или удаление навсегда.

---

## 10.7. Симуляция обновления

```bash
apt upgrade --dry-run    # Debian/Ubuntu
apt upgrade -s           # то же самое

dnf upgrade --assumeno   # RHEL/CentOS
```

"Dry run" — холостой прогон, репетиция без реального применения. Паттерн везде в IT:
```bash
rsync --dry-run
terraform plan
ansible --check
```

---

## 10.8. Локальный кэш пакетов

```
/var/cache/apt/archives/  — кэш .deb файлов (не критично если повреждён)
/var/lib/apt/lists/       — индексы репозиториев (критично если повреждён)
```

### Индексы — что это:
Текстовые файлы со списком всех пакетов в репозитории:
```
Package: nginx
Version: 1.24.0
Depends: libc6, libpcre3
Filename: pool/main/n/nginx/nginx_1.24.0_amd64.deb
SHA256: abc123...
```

`apt update` скачивает эти файлы. Без них apt не знает что вообще существует.

### Повреждение и починка:
```bash
# симптом:
apt install nginx
# E: Unable to locate package nginx

# починить:
rm -rf /var/lib/apt/lists/*
apt update
# или просто:
apt clean && apt update
```

```bash
apt clean       # удалить весь кэш .deb (освободить место)
apt autoclean   # удалить только устаревшие версии из кэша
```

Повреждается при: прерванном `apt update`, проблемах с диском, переполненном диске.

---

## 10.10. Список пакетов начинающихся с "python3-"

```bash
# Debian/Ubuntu:
dpkg -l 'python3-*'
apt list --installed | grep python3-

# RHEL/CentOS:
rpm -qa 'python3-*'
dnf list installed 'python3-*'
```

---

## Репозитории

У каждого дистрибутива свой репозиторий — веб-сервер с каталогом пакетов:
```
Ubuntu  → archive.ubuntu.com
Debian  → deb.debian.org
CentOS  → mirror.centos.org
```

`/etc/apt/sources.list` — список репозиториев:
```
deb https://archive.ubuntu.com/ubuntu jammy main restricted universe
```

Секции Ubuntu репо:
```
main       — официальные пакеты
universe   — community пакеты
restricted — проприетарные драйверы
multiverse — проприетарное ПО
```

Популярные приложения держат свои репо (Docker, Nginx, Chrome) — там всегда свежие версии.

## Основные команды apt:
```bash
apt update              # обновить индексы
apt install nginx       # установить
apt remove nginx        # удалить (конфиги остаются)
apt purge nginx         # удалить + конфиги
apt autoremove          # удалить ненужные зависимости
apt upgrade             # обновить все пакеты
apt upgrade --dry-run   # симуляция обновления
apt search nginx        # поиск
apt show nginx          # информация о пакете
apt list --installed    # список установленных
apt-cache policy nginx  # доступные версии
apt-mark hold nginx     # заморозить версию
apt clean               # очистить кэш
```
