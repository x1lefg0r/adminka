# Тема 15: Планировщики задач и журналирование

## 15.1. Cron vs таймеры systemd

### Cron — классический планировщик:
```bash
crontab -e   # редактировать задачи текущего пользователя
crontab -l   # посмотреть задачи

# формат строки cron:
# минута час день_месяца месяц день_недели команда
  0      3   *           *     *           /usr/local/bin/backup.sh
# каждый день в 3:00

# спецсимволы:
# * = любое значение
# */5 = каждые 5 единиц
# 1,3,5 = конкретные значения
# 1-5 = диапазон
```

### Таймеры systemd:
```bash
systemctl list-timers          # список всех таймеров
systemctl status backup.timer  # статус конкретного
```

### Сравнение:

| | Cron | systemd timer |
|--|------|--------------|
| Точность | минута | секунда и меньше |
| Логирование | отдельно (mail/syslog) | journald автоматически |
| Зависимости | нет | полные systemd зависимости |
| Пропущенные задачи | теряются | Persistent=true догонит |
| Сложность | проще | сложнее |

Cron проще для простых задач. systemd timer — когда нужна интеграция с systemd, зависимости, точное логирование.

---

## 15.2. Монотонное vs календарное планирование в systemd

### Календарное (realtime) — привязано к реальному времени:
```ini
[Timer]
OnCalendar=*-*-* 03:00:00    # каждый день в 3:00
OnCalendar=Mon *-*-* 00:00   # каждый понедельник
OnCalendar=weekly            # раз в неделю
OnCalendar=daily             # раз в день
```

### Монотонное — привязано к событию:
```ini
[Timer]
OnBootSec=5min          # через 5 минут после загрузки
OnUnitActiveSec=1h      # каждый час после последнего запуска
OnStartupSec=10min      # через 10 минут после старта systemd
```

**Монотонное** — не зависит от системного времени. Если изменить время — не повлияет.
**Календарное** — привязано к часам. Изменили время → задача запустится в нужный момент.

Реальный сценарий:
```
резервное копирование в 3:00 → OnCalendar (нужно конкретное время)
проверка после загрузки → OnBootSec (нужно после события)
```

---

## 15.3. anacron vs cron

**Проблема cron:** если машина была выключена в момент запуска задачи — задача пропускается навсегда.

```
cron: backup каждый день в 3:00
машина выключена в 3:00 → backup не выполнился → потеряли бэкап
```

**anacron** решает это:
```
anacron проверяет при загрузке — когда последний раз выполнялась задача?
если давно — выполняет сразу
```

```bash
# /etc/anacrontab:
# период  задержка  имя_задачи  команда
  1       5         backup      /usr/local/bin/backup.sh
# период 1 = ежедневно
# задержка 5 = запустить через 5 минут после загрузки
```

anacron не умеет запускать задачи в конкретное время — только с периодичностью (ежедневно, еженедельно). Для серверов которые работают 24/7 — cron. Для ноутбуков/десктопов — anacron.

---

## 15.4. journald — RAM vs диск

**journald** — демон журналирования systemd. Собирает логи со всех сервисов автоматически.

### Хранение по умолчанию — только в RAM:
```
/run/log/journal/   — volatile (энергозависимое), в RAM
при перезагрузке → логи исчезают
```

### Включить постоянное хранение:
```bash
mkdir -p /var/log/journal
systemd-tmpfiles --create --prefix /var/log/journal
systemctl restart systemd-journald
```

Или в конфиге:
```bash
# /etc/systemd/journald.conf:
Storage=persistent    # всегда на диск
Storage=volatile      # всегда в RAM
Storage=auto          # на диск если /var/log/journal существует (дефолт)
```

```bash
journalctl --disk-usage   # сколько места занимает журнал
journalctl --vacuum-size=500M   # оставить только 500MB
journalctl --vacuum-time=2weeks # оставить только за 2 недели
```

---

## 15.5. journalctl — критические ошибки с последней загрузки

```bash
journalctl -p err -b
# -p = priority (приоритет)
# err = уровень ошибок и выше
# -b = boot (с последней загрузки)
```

### Уровни приоритетов (от высшего к низшему):
```
0 = emerg    — система неработоспособна
1 = alert    — немедленное действие
2 = crit     — критические условия
3 = err      — ошибки
4 = warning  — предупреждения
5 = notice   — нормальные но важные
6 = info     — информационные
7 = debug    — отладка
```

`-p err` показывает err + всё выше (crit, alert, emerg).

---

## 15.6. logrotate

**logrotate** — утилита ротации логов. Предотвращает бесконечный рост лог файлов.

```bash
# /etc/logrotate.d/nginx:
/var/log/nginx/*.log {
    daily           # ротировать ежедневно
    missingok       # не ошибка если файл отсутствует
    rotate 14       # хранить 14 старых файлов
    compress        # сжимать старые логи (gzip)
    delaycompress   # не сжимать последний ротированный
    notifempty      # не ротировать если файл пустой
    create 0640 nginx adm  # создать новый файл с этими правами
    sharedscripts   # выполнить скрипты один раз для всех файлов
    postrotate
        nginx -s reopen   # сказать nginx открыть новые файлы
    endscript
}
```

### postrotate:
Блок команд которые выполняются ПОСЛЕ ротации. Нужен потому что:
```
logrotate переименовал nginx.log → nginx.log.1
создал новый пустой nginx.log
но nginx всё ещё пишет в старый файл (держит fd открытым)
postrotate: nginx -s reopen → nginx открывает новый nginx.log
```

Без postrotate — nginx продолжал бы писать в переименованный файл.

```bash
logrotate -f /etc/logrotate.d/nginx   # принудительно запустить
logrotate -d /etc/logrotate.d/nginx   # dry-run (симуляция)
```

---

## 15.7. rsyslog — аутентификационные логи в отдельный файл

**rsyslog** — демон системного журналирования. Старше journald, работает параллельно.

```bash
# /etc/rsyslog.d/auth.conf:
auth,authpriv.*  /var/log/auth.log
```

Синтаксис:
```
facility.severity  destination

facility (источник):
  auth/authpriv — аутентификация
  kern          — ядро
  daemon        — демоны
  mail          — почта
  cron          — планировщик
  *             — все

severity (важность):
  emerg, alert, crit, err, warning, notice, info, debug, *

destination:
  /var/log/file.log   — файл
  @server:514         — UDP на сервер
  @@server:514        — TCP на сервер
```

```bash
systemctl restart rsyslog   # применить после изменений
```

---

## 15.8. Запланированные задачи текущего пользователя

```bash
crontab -l   # список задач текущего пользователя
```

Системные задачи:
```bash
cat /etc/crontab           # системный crontab
ls /etc/cron.d/            # задачи пакетов
ls /etc/cron.daily/        # ежедневные скрипты
ls /etc/cron.weekly/       # еженедельные
ls /etc/cron.hourly/       # ежечасные
```

---

## 15.9. journald — почему логи теряются после перезагрузки

По умолчанию `Storage=auto` — journald пишет в RAM (`/run/log/journal/`).

`/run` — tmpfs (в памяти), очищается при перезагрузке. Поэтому логи исчезают.

Если существует `/var/log/journal/` — journald автоматически начинает писать туда (на диск). Постоянное хранение.

```bash
# проверить текущий режим:
journalctl --header | grep "File Path"

# логи прошлых загрузок (если persistent):
journalctl -b -1   # предыдущая загрузка
journalctl -b -2   # позапрошлая
journalctl --list-boots  # список всех загрузок
```

---

## Основные команды journalctl

```bash
journalctl                          # весь журнал
journalctl -f                       # follow (в реальном времени)
journalctl -u nginx                 # только nginx
journalctl -n 50                    # последние 50 записей
journalctl -p err                   # только ошибки и выше
journalctl -p err -b                # ошибки с последней загрузки
journalctl --since "2026-06-25"     # с даты
journalctl --since "1 hour ago"     # за последний час
journalctl -b                       # с последней загрузки
journalctl -b -1                    # с предыдущей загрузки
journalctl --list-boots             # список загрузок
journalctl --disk-usage             # размер журнала
journalctl -o json-pretty           # вывод в JSON
journalctl -o json-pretty -u nginx | grep "_PID"  # конкретные поля
```

## Основные команды crontab

```bash
crontab -e   # редактировать
crontab -l   # список
crontab -r   # удалить все задачи (осторожно!)
crontab -u username -l  # задачи другого пользователя (root)
```

### Примеры cron выражений:
```
*/5 * * * *        — каждые 5 минут
0 3 * * *          — каждый день в 3:00
0 3 * * 1          — каждый понедельник в 3:00
0 3 1 * *          — первого числа каждого месяца в 3:00
0 3 * * 1-5        — по будням в 3:00
@reboot            — при загрузке системы
@daily             — раз в день (= 0 0 * * *)
@weekly            — раз в неделю
@monthly           — раз в месяц
```
