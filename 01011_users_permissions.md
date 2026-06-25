# Тема 11: Пользователи, группы и права доступа

## 11.1. Real, Effective, Saved UID

Три вида UID у каждого процесса:
```
RUID (real)      — кто реально запустил процесс (твой uid)
EUID (effective) — от чьего имени процесс работает прямо сейчас (ядро проверяет это)
SUID (saved)     — сохранённый, для возврата привилегий
```

Аналогия:
```
RUID = паспорт (кто ты на самом деле)
EUID = бейджик (какие права у тебя сейчас)
```

Ядро проверяет EUID при каждом обращении к файлу — не RUID.

### Пример с passwd:
```bash
ls -la /usr/bin/passwd
# -rwsr-xr-x root root /usr/bin/passwd  ← SUID бит

# ты (uid=1000) запускаешь passwd:
RUID = 1000   # ты
EUID = 0      # root (из-за SUID бита)
# passwd может писать в /etc/shadow потому что EUID=root
# но passwd проверяет RUID — позволяет менять только твой пароль
```

### Saved UID:
Процесс может временно сбросить привилегии и вернуться:
```
EUID=0 → сбросить до EUID=1000 → поработать без привилегий → вернуть EUID=0 (из saved)
```

---

## 11.2. Права rwx для файлов и директорий

```bash
ls -la
# -rwxr-xr-- 1 god staff file.txt
# drwxr-x--- 2 god staff mydir/
```

Три роли всегда в порядке: **владелец → группа → остальные**

### Для файлов:
```
r — читать содержимое
w — изменять содержимое
x — выполнять как программу
```

### Для директорий — другая логика:
```
r — читать список файлов (ls)
w — создавать/удалять файлы внутри
x — входить (cd) и обращаться к файлам внутри
```

`x` на директории критичен — без него вообще ничего не можешь делать внутри:
```bash
chmod 600 /mydir   # убрали x
cd /mydir          # → Permission denied
ls /mydir          # → Permission denied даже с r!
```

Интересная комбинация:
```bash
chmod 111 /mydir   # только x, без r
cd /mydir          # можно войти
ls /mydir          # нельзя (нет r)
cat /mydir/file    # можно если знаешь имя файла!
```

---

## 11.3. Добавить пользователя в группу

```bash
usermod -aG groupname username
# -a = append (без этого заменит ВСЕ группы!)
# -G = supplementary Groups
```

```bash
usermod -aG docker god    # добавить в docker
usermod -aG sudo god      # добавить в sudo

groups god                # список групп
id god                    # uid, gid и все группы

newgrp docker             # применить без перелогина
```

### usermod флаги:
```bash
-a        # append — добавить (всегда с -G)
-G        # supplementary Groups
-g        # primary group — основная группа
-d        # home Directory
-m        # move — переместить home (с -d)
-s        # shell — изменить оболочку
-l        # login — изменить имя пользователя
-L        # Lock — заблокировать
-U        # Unlock — разблокировать
-e        # expire — дата истечения
-c        # comment — полное имя
```

### Связанные утилиты:
```bash
useradd username      # создать пользователя
userdel username      # удалить
userdel -r username   # удалить + домашнюю директорию
passwd username       # сменить пароль
groupadd groupname    # создать группу
```

---

## 11.4. Sticky Bit

```bash
ls -la /tmp
# drwxrwxrwt   ← t = sticky bit
```

Sticky bit на директории — удалять файлы может только владелец файла, владелец директории, или root:

```bash
chmod +t /shared      # установить
chmod 1777 /shared    # то же числом (1 = sticky)
```

```
drwxrwxrwt — t = sticky + x есть
drwxrwxrwT — T = sticky, x нет
```

Важно: sticky не запрещает редактировать чужие файлы — только удалять. Редактирование контролируется правами самого файла.

```
права директории → создание/удаление файлов внутри
права файла      → чтение/запись самого файла
```

---

## 11.5. ACL — Access Control List

Расширенные права — позволяют задать права конкретному пользователю/группе помимо стандартных трёх ролей.

```bash
getfacl file.txt
# user::rw-
# user:vasya:r--    ← конкретный пользователь
# group::r--
# mask::r--
# other::r--
```

### Синтаксис ACL записи:
```
u:vasya:rw   # user — конкретный пользователь
g:dev:rx     # group — конкретная группа
o::r         # other — все остальные
m::rw        # mask — потолок прав для ACL записей
u::rw        # владелец файла (без имени)
g::r         # группа файла (без имени)
```

### setfacl флаги:
```bash
-m   # modify — добавить/изменить
-x   # remove — удалить запись
-b   # blank — удалить все ACL
-k   # удалить только default ACL
-d   # default ACL (для директорий, наследуется новыми файлами)
-R   # recursive
```

```bash
setfacl -m u:vasya:rw file.txt        # дать vasya права rw
setfacl -m g:dev:rx file.txt          # дать группе dev права rx
setfacl -x u:vasya file.txt           # убрать ACL для vasya
setfacl -b file.txt                   # убрать все ACL
setfacl -Rm u:vasya:rw /shared        # рекурсивно
setfacl -d -m u:vasya:rw /shared      # default ACL для новых файлов
```

### Когда нужны ACL:
Стандартные права — только три роли. ACL нужен когда:
- Разные пользователи имеют разные права
- Нельзя решить через группы
- Нужны права для конкретного пользователя без добавления в группу

---

## 11.6. PAM — Pluggable Authentication Modules

Модульная система аутентификации. Программы не знают как аутентифицировать — спрашивают PAM.

```
ssh → PAM → /etc/pam.d/sshd → список модулей
```

Каждый сервис имеет свой конфиг в `/etc/pam.d/`:
```
/etc/pam.d/sshd    — SSH
/etc/pam.d/sudo    — sudo
/etc/pam.d/login   — консоль
/etc/pam.d/passwd  — смена пароля
```

### Четыре типа модулей:
```
auth     — аутентификация (проверка пароля)
account  — проверка аккаунта (не истёк ли)
password — смена пароля
session  — действия при входе/выходе
```

### Флаги модулей:
```
required   — должен пройти, но продолжает остальные (для timing атак)
requisite  — провал → сразу стоп
sufficient — прошёл → достаточно, остальные не нужны
optional   — необязательный
```

`required` вместо `requisite` для безопасности — одинаковое время ответа при любой ошибке, сложнее определить причину отказа. Плюс все модули выполняются (например pam_tally2 считает попытки).

Добавить двухфакторку без изменения SSH:
```bash
# /etc/pam.d/sshd
auth required pam_unix.so
auth required pam_google_authenticator.so
```

---

## 11.7. umask

Маска вычитается из максимальных прав при создании файла:

```
максимум файла:      666 (без x — файлы не исполняемые по умолчанию)
максимум директории: 777

umask 027:
файл:      666 - 027 = 640 (rw-r-----)
директория: 777 - 027 = 750 (rwxr-x---)

дефолт umask 022:
файл:      666 - 022 = 644 (rw-r--r--)
директория: 777 - 022 = 755 (rwxr-xr-x)
```

Почему файлы максимум 666 (без x) — файлы по умолчанию не должны быть исполняемыми. `x` добавляется только явно через `chmod +x`.

```bash
umask              # посмотреть текущий
umask 027          # установить для сессии
# постоянно — в ~/.bashrc или /etc/profile
```

---

## 11.8. ulimit — ограничения ресурсов

```bash
ulimit -a          # все лимиты
ulimit -n 1024     # открытые файлы
ulimit -u 100      # процессы
ulimit -Sn 1024    # soft лимит
ulimit -Hn 4096    # hard лимит
```

### Soft vs Hard:
```
soft — текущий лимит, пользователь может увеличить до hard
hard — потолок, только root может увеличить
```

Пользователь: soft → увеличить до hard ✅, hard → только уменьшить.

Постоянно в `/etc/security/limits.conf`:
```
god  soft  nofile  1024
god  hard  nofile  4096
*    soft  nproc   100    # для всех
```

Ядро проверяет лимиты при каждом syscall (`open()`, `fork()`).

---

## 11.9. SUID, SGID, Sticky bit

```
SUID (4) — на файле: EUID = владелец файла при запуске
SGID (2) — на файле: EGID = группа файла при запуске
           на директории: новые файлы наследуют группу директории
Sticky(1) — на директории: удалять может только владелец файла
```

```bash
chmod u+s file    # SUID (u = user/владелец)
chmod g+s file    # SGID (g = group)
chmod +t dir      # sticky

# числами (первая цифра):
chmod 4755 file   # SUID
chmod 2755 dir    # SGID
chmod 1777 dir    # sticky
```

В `ls`:
```
-rwsr-xr-x   # SUID — s вместо x у владельца
-rwxr-sr-x   # SGID — s вместо x у группы
drwxrwxrwt   # sticky — t вместо x у остальных
-rwSr-xr-x   # SUID есть, но x нет (S заглавная — редко, обычно баг)
```

### SUID на скриптах не работает:
Ядро игнорирует SUID на скриптах (bash, python) — только на скомпилированных ELF бинарниках. Скрипт интерпретируется другой программой — SUID сработал бы на интерпретаторе, это дыра в безопасности.

### Пример SUID утилиты:
```bash
ls -la /usr/bin/passwd
# -rwsr-xr-x root root /usr/bin/passwd
```

### Принцип минимальных привилегий:
SUID даёт повышенные права только конкретной программе на конкретное действие — не всем пользователям root права.

---

## 11.10. Блокировка учётной записи

```bash
usermod -L god        # Lock — заблокировать
usermod -U god        # Unlock — разблокировать
passwd -l god         # то же через passwd
passwd -u god         # разблокировать
```

В `/etc/shadow` добавляется `!` перед хэшем:
```
god:$6$abc123$хэш:...    # разблокирован
god:!$6$abc123$хэш:...   # заблокирован
```

**Важно:** вход по SSH ключу работает даже при `-L` — ключи не через пароль!

Полная блокировка:
```bash
usermod -L god                    # заблокировать пароль
usermod -s /sbin/nologin god      # запретить интерактивный вход
```

`/sbin/nologin` — не shell, просто выводит "account not available" и выходит.

Но даже с `/sbin/nologin` можно выполнить команду:
```bash
ssh god@server "ls /home/god"   # без интерактивного входа работает
```

---

## 11.11. ACL vs стандартные права

Стандартные права — только три роли (владелец, группа, остальные).

ACL нужен когда стандартных трёх ролей недостаточно:
```bash
# нельзя через стандартные права:
# vasya читает, petya пишет, все остальные ничего

# через ACL:
setfacl -m u:vasya:r file.txt
setfacl -m u:petya:rw file.txt
setfacl -m o::--- file.txt
```

---

## 11.12. sudoers — конкретная команда без пароля

```bash
visudo   # безопасный редактор /etc/sudoers (проверяет синтаксис)
visudo -f /etc/sudoers.d/god   # отдельный файл
```

Синтаксис:
```
пользователь  хост=(от_кого)  команда
god           ALL=(root)      NOPASSWD: /bin/systemctl restart nginx
%developers   ALL=(root)      NOPASSWD: /bin/systemctl restart nginx
# % перед именем = группа
```

```bash
sudo -l   # посмотреть что можешь делать
```

### /etc/sudoers.d/ — директория фрагментов:
Вместо одного большого файла — много маленьких:
```
/etc/sudoers.d/
├── god
├── developers
└── docker
```

В `/etc/sudoers` есть `#includedir /etc/sudoers.d` — sudo читает все файлы из директории.

Паттерн `.d` (directory) везде в Linux:
```
/etc/sudoers.d/
/etc/cron.d/
/etc/apt/sources.list.d/
/etc/nginx/conf.d/
```

Идея — вместо одного большого файла, много маленьких фрагментов.

---

## /etc/passwd и /etc/shadow

```bash
cat /etc/passwd
# god:x:1000:1000::/home/god:/bin/bash
# имя:пароль:uid:gid:комментарий:home:shell
# x = пароль в /etc/shadow
```

```bash
cat /etc/shadow
# god:$6$abc123$хэш:19000:0:99999:7:::
# имя:хэш_пароля:дата_смены:...
```

`/etc/passwd` — читают все (нужен для uid→имя).
`/etc/shadow` — только root (хэши паролей).

Раньше хэши были в `/etc/passwd` — любой мог брутфорсить оффлайн. Вынесли в shadow.
