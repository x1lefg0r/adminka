# Тема 12: Безопасность — AppArmor, SELinux, auditd

## 12.1. AppArmor vs SELinux

Оба — MAC (Mandatory Access Control). Работают по-разному:

### AppArmor:
- Политики основаны на **путях к файлам**
- Проще в настройке
- Профиль для каждой программы
- Дефолт на Ubuntu/Debian

```bash
# профиль живёт в:
/etc/apparmor.d/usr.sbin.nginx

# nginx может:
/etc/nginx/** r,         # читать конфиги
/var/log/nginx/** w,     # писать логи
# nginx не может:
# /etc/shadow — не прописано → запрещено
```

### SELinux:
- Политики основаны на **контекстах безопасности** (labels)
- Сложнее но мощнее
- Дефолт на RHEL/CentOS

```bash
ls -Z /etc/shadow
# system_u:object_r:shadow_t:s0 /etc/shadow
```

### Главное различие:
```
AppArmor — путь к файлу (/etc/shadow)
SELinux  — контекст файла (shadow_t) — не зависит от пути
```

Переместил файл — AppArmor потеряет (путь изменился). SELinux не потеряет (контекст остался).

---

## 12.2. DAC vs MAC

**DAC (Discretionary Access Control)** — дискреционный:
```
владелец сам решает кому давать доступ
chmod, chown, ACL — всё это DAC
```

Проблема:
```
скомпрометированный nginx работает как root
→ читает /etc/shadow (DAC разрешает — root же)
→ данные уже утекли до того как заметили
```

**MAC (Mandatory Access Control)** — мандатный:
```
система устанавливает правила, владелец не может их изменить
SELinux, AppArmor — MAC
```

```
nginx скомпрометирован
→ пытается читать /etc/shadow
→ SELinux: httpd_t не может читать shadow_t → отказ
→ данные не утекли
```

Два уровня защиты:
```
DAC (unix права) → MAC (SELinux/AppArmor) → доступ
оба должны разрешить
```

MAC не про то что root бессилен — про **least privilege**: nginx не должен иметь доступ к /etc/shadow вообще, зачем давать?

---

## 12.3. auditd — классы событий

```bash
# добавить правило:
auditctl -w /etc/passwd -p rwxa -k passwd_changes
# -w = watch (файл)
# -p = permissions (r=read, w=write, x=execute, a=attribute)
# -k = key (метка для поиска)

# посмотреть логи:
ausearch -k passwd_changes
ausearch -m avc -ts recent   # SELinux события
aureport --summary            # сводный отчёт
```

Логи в `/var/log/audit/audit.log`.

Классы событий:
```
syscall  — системные вызовы
file     — доступ к файлам
user     — логин, sudo
network  — соединения
process  — создание/завершение
```

Для сетевых соединений удобнее:
```bash
ss -tnp      # TCP соединения с процессами
lsof -i      # все сетевые соединения
tcpdump      # перехват пакетов
```

Заблокировать изменение правил (immutable mode):
```bash
auditctl -e 2   # даже root не может остановить без перезагрузки
```

---

## 12.4. auditd vs стандартное журналирование

```
journald/syslog:
- приложения сами решают что логировать
- можно пропустить важные события
- процесс может скрыть свои действия

auditd:
- работает на уровне ядра — перехватывает syscall-ы
- процесс не может скрыть действия
- tamper-resistant
- кто/что/когда с точностью до пользователя
```

```bash
# journald — что приложение решило записать:
journalctl -u nginx
# → "nginx started", "GET /index.html 200"

# auditd — что реально происходило на уровне ядра:
ausearch -f /etc/shadow
# → кто открывал, когда, какой процесс, какой uid
```

### Как обойти auditd:
- Обычный процесс — не может
- Root — может выключить, но это залогируется
- Kernel rootkit — может обойти (работает до auditd)

**Rootkit** — вредоносный модуль ядра который подменяет syscall table:
```c
// подменить sys_open своей функцией:
syscall_table[__NR_open] = evil_open;

// evil_open скрывает нужные файлы от auditd
```

Защита — логи на отдельный сервер в реальном времени + `auditctl -e 2`.

Обнаружение rootkit:
```bash
rkhunter --check    # rootkit hunter
chkrootkit          # проверка известных rootkit-ов
# надёжнее — загрузиться с внешнего носителя
```

---

## 12.5. Четыре компонента контекста SELinux (*)

```
system_u : object_r : shadow_t : s0
   ↑           ↑         ↑       ↑
 юзер        роль      тип    уровень
```

- **user** — SELinux пользователь. `system_u`, `unconfined_u`
- **role** — роль. `object_r` для файлов, `system_r` для процессов
- **type** — **самое важное**. Определяет политику доступа
- **level** — уровень секретности (MLS). Обычно `s0`

На практике смотришь только на **type**.

Политика = огромная матрица:
```
кто (тип процесса) × что (тип файла) × действие = разрешено/запрещено

httpd_t → shadow_t → read = ЗАПРЕЩЕНО
httpd_t → httpd_sys_content_t → read = РАЗРЕШЕНО
passwd_t → shadow_t → write = РАЗРЕШЕНО
```

---

## 12.6. Переключить в режим логирования без блокировки (*)

```bash
# SELinux:
setenforce 0      # permissive — логирует но не блокирует
setenforce 1      # enforcing — блокирует
getenforce        # текущий режим
sestatus          # подробный статус

# AppArmor:
aa-complain /usr/sbin/nginx   # complain mode
aa-enforce /usr/sbin/nginx    # enforce mode
```

---

## 12.7. Enforce vs Complain в AppArmor (*)

```
enforce  — нарушения блокируются
complain — нарушения только логируются
```

Complain используют при написании нового профиля — смотришь что блокируется, добавляешь правила.

```bash
aa-status                   # статус всех профилей
aa-genprof /usr/sbin/nginx  # сгенерировать профиль автоматически
journalctl | grep apparmor  # логи нарушений
```

---

## 12.8. Восстановить контексты SELinux (*)

```bash
restorecon -Rv /home/username
# -R = recursive
# -v = verbose
```

Самая частая причина проблем с SELinux — файл скопирован из другого места и получил неправильный контекст. `restorecon` исправляет.

---

## 12.9. Правило аудита для /etc/passwd (*)

```bash
auditctl -w /etc/passwd -p wa -k passwd_watch
# -w = watch
# -p wa = write + attribute change
# -k = key
```

---

## 12.10. Почему веб-сервер не подключается к нестандартному порту (*)

SELinux контролирует какие порты может использовать процесс:

```bash
semanage port -l | grep http
# http_port_t  tcp  80, 443, 8080

# разрешить нестандартный порт:
semanage port -a -t http_port_t -p tcp 8888
```

Файрвол разрешает → но SELinux блокирует на уровне типа порта. Два уровня проверки — оба должны разрешить.

---

## Основные команды

```bash
# SELinux:
getenforce                    # текущий режим
setenforce 0/1               # permissive/enforcing
sestatus                      # статус
ls -Z file                    # контекст файла
ps -Z                         # контекст процессов
restorecon -Rv /path          # восстановить контексты
semanage port -l              # разрешённые порты
ausearch -m avc -ts recent    # что заблокировал SELinux
audit2allow -a                # сгенерировать правило из логов

# AppArmor:
aa-status                     # статус профилей
aa-enforce /usr/sbin/nginx    # enforce mode
aa-complain /usr/sbin/nginx   # complain mode
aa-genprof /usr/sbin/nginx    # сгенерировать профиль

# auditd:
auditctl -w /file -p rwxa -k key  # добавить правило
auditctl -l                        # список правил
auditctl -e 2                      # immutable mode
ausearch -k key                    # поиск по ключу
ausearch -f /etc/shadow            # поиск по файлу
aureport --summary                 # сводный отчёт
```
