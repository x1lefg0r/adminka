# Тема 9: Система инициализации systemd

## 9.1. Unit в systemd

**Unit** — базовая единица управления в systemd. Файл конфигурации который описывает что запустить, как запустить и когда.

### Типы юнитов:
```
service  — сервис/демон (nginx.service, sshd.service)
target   — группа юнитов, аналог runlevel (multi-user.target)
timer    — планировщик задач, аналог cron (backup.timer)
socket   — сокет для socket activation (nginx.socket)
mount    — точка монтирования (/home.mount)
automount— автоматическое монтирование
path     — отслеживание изменений файлов/директорий
device   — устройство
slice    — группа процессов для управления ресурсами (cgroups)
scope    — группа процессов запущенных извне systemd
```

Файлы юнитов живут в:
```
/lib/systemd/system/    — системные юниты (пакеты)
/etc/systemd/system/    — пользовательские юниты (приоритет выше)
```

---

## 9.2. Wants= vs Requires=

```ini
[Unit]
Wants=network.target      # мягкая зависимость
Requires=network.target   # жёсткая зависимость
```

- **`Wants=`** — запустить зависимость если возможно. Если зависимость не запустилась — сервис всё равно стартует
- **`Requires=`** — жёсткая зависимость. Если зависимость не запустилась — сервис тоже не запустится

```ini
# пример — nginx хочет сеть но не требует:
[Unit]
Wants=network-online.target
After=network-online.target   # запускаться ПОСЛЕ сети
```

`After=` — порядок запуска (не зависимость). `Before=` — наоборот.

---

## 9.3. Изменить цель загрузки по умолчанию

```bash
systemctl set-default multi-user.target    # многопользовательский без GUI
systemctl set-default graphical.target     # с GUI
systemctl get-default                      # посмотреть текущую
```

Временно переключить прямо сейчас:
```bash
systemctl isolate multi-user.target
```

---

## 9.4. Socket activation

Механизм при котором systemd слушает сокет вместо сервиса. Сервис запускается только когда приходит первое соединение.

```
обычно:
сервис запущен → слушает сокет → ждёт соединений (потребляет ресурсы вхолостую)

socket activation:
systemd слушает сокет → сервис не запущен → пришло соединение → systemd запускает сервис → передаёт соединение
```

Преимущества:
- Сервисы не запущены пока не нужны → меньше потребление памяти
- Параллельная загрузка — systemd открывает все сокеты сразу, сервисы стартуют параллельно
- Автоматический перезапуск — сокет остаётся открытым пока сервис перезапускается

Файлы:
```
nginx.socket   — описывает сокет
nginx.service  — описывает сервис
```

---

## 9.5. Просмотр журнала сервиса

```bash
journalctl -u nginx.service -n 50          # последние 50 записей
journalctl -u nginx.service -n 50 --no-pager  # без пейджера
journalctl -u nginx.service -f             # follow, в реальном времени
journalctl -u nginx.service --since "1 hour ago"
journalctl -u nginx.service -p err         # только ошибки
```

stderr сервиса тоже попадает в journald автоматически.

---

## 9.6. systemd-tmpfiles

Управляет временными директориями и файлами при загрузке:
- Создаёт директории (`/tmp`, `/run`, `/var/run`)
- Удаляет старые файлы по правилам
- Устанавливает права и владельца

```bash
systemd-tmpfiles --create    # создать файлы/директории по конфигу
systemd-tmpfiles --clean     # удалить устаревшие файлы
```

Конфиги в `/etc/tmpfiles.d/` и `/usr/lib/tmpfiles.d/`:
```
# тип путь режим владелец группа возраст
d /var/log/myapp 0755 myapp myapp -      # создать директорию
D /tmp/cache     0755 root  root  1d     # создать и чистить старше 1 дня
```

---

## 9.7. Полностью заблокировать сервис

```bash
systemctl mask nginx.service    # замаскировать — нельзя запустить вообще
systemctl unmask nginx.service  # снять маску
```

Разница:
```
systemctl disable nginx  # отключить автозапуск, но можно запустить вручную
systemctl mask nginx     # создать симлинк на /dev/null — нельзя запустить никак
```

---

## 9.8. Restart= и ограничение частоты перезапусков

```ini
[Service]
Restart=always          # всегда перезапускать
Restart=on-failure      # только при ошибке (ненулевой код возврата)
Restart=on-abnormal     # при сигналах, таймаутах
Restart=no              # не перезапускать (дефолт)

RestartSec=5s           # пауза между перезапусками

# ограничение частоты:
StartLimitBurst=5       # максимум 5 перезапусков
StartLimitIntervalSec=10s # за 10 секунд
# если превысили — сервис переходит в failed, не перезапускается
```

---

## 9.9. Targets vs уровни выполнения SysVinit

**SysVinit runlevels** — старая система:
```
0 — выключение
1 — однопользовательский (восстановление)
2 — многопользовательский без сети
3 — многопользовательский с сетью
5 — графический
6 — перезагрузка
```

**systemd targets** — современная система:
```
poweroff.target      ≈ runlevel 0
rescue.target        ≈ runlevel 1
multi-user.target    ≈ runlevel 3
graphical.target     ≈ runlevel 5
reboot.target        ≈ runlevel 6
```

Отличия:
- Target — просто группа юнитов, можно создать свои
- Несколько targets могут быть активны одновременно
- Зависимости между targets явные и гибкие
- Параллельный запуск юнитов внутри target

---

## 9.10. Статус сервиса с журналом

```bash
systemctl status ssh.service
```

Показывает:
- Статус (active/inactive/failed)
- PID, время запуска
- Последние строки журнала автоматически

---

## 9.11. Таймер systemd — каждый день в 3 ночи

Нужно два файла:

**`/etc/systemd/system/backup.service`:**
```ini
[Unit]
Description=Daily backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

**`/etc/systemd/system/backup.timer`:**
```ini
[Unit]
Description=Run backup daily at 3am

[Timer]
OnCalendar=*-*-* 03:00:00    # каждый день в 3:00
Persistent=true               # запустить если пропустили (система была выключена)

[Install]
WantedBy=timers.target
```

```bash
systemctl enable --now backup.timer   # включить и запустить
systemctl list-timers                 # список всех таймеров
```

---

## 9.12. Включить автозапуск и запустить сейчас

```bash
systemctl enable --now nginx
# enable — добавить в автозапуск
# --now   — запустить прямо сейчас не дожидаясь перезагрузки
```

Раздельно:
```bash
systemctl enable nginx    # только автозапуск
systemctl start nginx     # только запустить сейчас
```

---

## 9.13. Иерархия зависимостей юнита

```bash
systemctl list-dependencies network.target
# показывает дерево зависимостей

systemctl list-dependencies --reverse network.target
# кто зависит от этого юнита
```

---

## Основные команды systemctl

```bash
systemctl start nginx      # запустить
systemctl stop nginx       # остановить
systemctl restart nginx    # перезапустить
systemctl reload nginx     # перечитать конфиг (SIGHUP)
systemctl status nginx     # статус
systemctl enable nginx     # автозапуск
systemctl disable nginx    # отключить автозапуск
systemctl mask nginx       # заблокировать полностью
systemctl unmask nginx     # разблокировать
systemctl is-active nginx  # активен ли
systemctl is-enabled nginx # включён ли автозапуск
systemctl daemon-reload    # перечитать файлы юнитов после изменений
```

---

## journalctl — основные команды

```bash
journalctl                          # весь журнал
journalctl -f                       # follow, в реальном времени
journalctl -u nginx                 # только nginx
journalctl -n 50                    # последние 50 записей
journalctl -p err                   # только ошибки (err, warning, info, debug)
journalctl --since "2026-06-25"     # с определённой даты
journalctl --since "1 hour ago"     # за последний час
journalctl -b                       # с последней загрузки
journalctl -b -1                    # с предыдущей загрузки
journalctl --disk-usage             # сколько места занимает журнал
journalctl -p err -b                # ошибки с последней загрузки (вопрос 15.5)
journalctl -o json-pretty           # вывод в JSON
```

---

## 9.14. systemd и cgroups

systemd использует cgroups (control groups) для:
- Отслеживания всех процессов сервиса (включая форкнутые)
- Ограничения ресурсов (CPU, память, I/O)
- Гарантированного завершения всех процессов при остановке

```ini
[Service]
MemoryLimit=512M
CPUQuota=50%
```

При двойном форке (демонизация) — процесс выходит из-под контроля традиционных систем. systemd через cgroups отслеживает все процессы в группе независимо от форков.

При `systemctl stop` — systemd посылает SIGTERM всей cgroup, потом SIGKILL если не завершились. Ни один процесс не ускользнёт.
