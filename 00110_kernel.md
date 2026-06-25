# Тема 6: Устройство Linux — ядро, загрузка, системные вызовы

## 6.1. Пространство ядра vs пространство пользователя

```
kernel space — ядро, драйверы, полный доступ к железу
user space   — все пользовательские процессы, ограниченный доступ
```

Почему важно: если бы любой процесс мог напрямую обращаться к железу — один кривой процесс мог бы положить всю систему. Разделение гарантирует изоляцию.

Переход между ними — только через **системные вызовы** (syscall). Это контролируемые "двери" в ядро.

### Как выглядит переход:
```c
write(1, "hello", 5);
// 1. положить аргументы в регистры CPU
// 2. выполнить инструкцию syscall
// 3. CPU → kernel mode
// 4. ядро выполняет операцию
// 5. CPU → user mode
// 6. вернуть результат
```

```bash
strace ls   # посмотреть все syscall-ы которые делает ls
```

---

## 6.2. Последовательность загрузки

```
BIOS/UEFI → GRUB → ядро → initramfs → root ФС → systemd (PID 1)
```

**1. BIOS/UEFI** — POST (Power On Self Test): проверяет RAM, CPU, диски. Ищет загрузочное устройство.

**2. GRUB** (Grand Unified Bootloader) — загрузчик в начале диска. Показывает меню ОС, загружает ядро в память, передаёт параметры и управление ядру.

**3. Ядро** — инициализирует железо, драйверы, монтирует initramfs как временную корневую ФС.

**4. initramfs** — минимальная временная ФС в памяти. Содержит драйверы для доступа к настоящей корневой ФС (RAID, LVM, шифрование).

**5. Настоящая root ФС** — монтируется, initramfs размонтируется.

**6. systemd (PID 1)** — первый пользовательский процесс, запускает все сервисы.

### systemd — PID 1:
```
systemd (PID 1)
├── sshd
├── nginx
│   ├── worker
│   └── worker
└── bash
    └── твоя команда
```

Убить PID 1 нельзя — ядро игнорирует SIGKILL для PID 1. Выключение через:
```bash
systemctl poweroff   # корректное завершение всех сервисов
```

---

## 6.3. Параметры ядра при загрузке

```bash
cat /proc/cmdline
# → BOOT_IMAGE=/vmlinuz root=/dev/sda1 quiet splash
```

Параметры передаёт GRUB из своего конфига:
```bash
cat /etc/default/grub
# GRUB_CMDLINE_LINUX="quiet splash"
```

Примеры параметров:
```
root=/dev/sda1   # где корневая ФС
quiet            # не выводить сообщения ядра при загрузке
splash           # графическая заставка
init=/bin/bash   # запустить bash вместо systemd (восстановление)
nomodeset        # отключить драйверы видеокарты
ro               # смонтировать корневую ФС только для чтения
```

### Временно изменить через GRUB:
1. При загрузке зажать `Shift` (BIOS) или `Esc` (UEFI)
2. Выбрать пункт, нажать `e`
3. Найти строку с `linux`, изменить параметры
4. `Ctrl+X` — загрузиться (только для этой загрузки)

### Постоянно:
```bash
nano /etc/default/grub
# GRUB_CMDLINE_LINUX="quiet splash твой_параметр"
# GRUB_TIMEOUT=5     # показывать меню 5 секунд
# GRUB_TIMEOUT=0     # не показывать
# GRUB_TIMEOUT=-1    # ждать вечно

update-grub          # Debian/Ubuntu — перегенерировать grub.cfg
grub2-mkconfig -o /boot/grub2/grub.cfg   # RHEL/CentOS
```

---

## 6.4. Загружаемые модули ядра

Ядро не содержит все драйверы сразу — загружает по необходимости как модули (`.ko` файлы).

```bash
lsmod              # список загруженных модулей
modinfo bluetooth  # информация о модуле
```

### Ручная загрузка (`insmod`) — без разрешения зависимостей:
```bash
insmod /path/to/module.ko   # загрузить напрямую
rmmod module_name           # выгрузить
```

### Через штатные утилиты (`modprobe`) — умный, знает зависимости:
```bash
modprobe bluetooth    # загрузить + все зависимости автоматически
modprobe -r bluetooth # выгрузить + ненужные зависимости
```

### Зависимости модулей:
```
bluetooth
└── requires: rfkill
    └── requires: cfg80211
```

`insmod bluetooth` — ошибка если rfkill не загружен.
`modprobe bluetooth` — загрузит cfg80211 → rfkill → bluetooth автоматически.

Граф зависимостей:
```bash
/lib/modules/$(uname -r)/modules.dep
```

`uname -r` — версия текущего ядра. Модули компилируются под конкретную версию ядра.

### Автозагрузка:
```bash
echo "bluetooth" >> /etc/modules
```

---

## 6.5. strace — трассировка системных вызовов

```bash
strace -o strace.log /bin/ls
```

- `-o` (output) — куда писать вывод
- По умолчанию пишет в stderr (чтобы не мешать stdout программы)

```bash
strace ls                          # все syscall-ы в stderr
strace -p 1234                     # attach к живому процессу (-p = pid)
strace -e trace=open,read,write ls # фильтр по syscall-ам (-e = expression)
strace -e open,read,write ls       # то же самое, trace= дефолтный тип
strace -c ls                       # статистика вызовов (-c = count)
```

### `-e` мини-язык:
```bash
-e trace=open,read,write    # фильтр по syscall-ам
-e signal=SIGTERM           # фильтр по сигналам
-e read=3                   # показывать содержимое fd 3
-e trace=read,signal=SIGTERM # несколько типов вместе
```

```bash
ltrace ls   # вызовы функций библиотек (в отличие от strace — syscall-ов)
```

---

## 6.6. Виртуальная файловая система /proc

`/proc` — не реальная ФС на диске. Ядро генерирует содержимое на лету при чтении, при закрытии — данные нигде не сохраняются.

```bash
cat /proc/cpuinfo      # информация о CPU
cat /proc/meminfo      # использование памяти
cat /proc/uptime       # время работы системы
cat /proc/cmdline      # параметры загрузки
cat /proc/1234/maps    # карта памяти процесса
ls /proc/1234/fd/      # открытые файловые дескрипторы
```

Под капотом при `cat /proc/meminfo`:
1. `cat` делает `open("/proc/meminfo")`
2. Ядро перехватывает
3. Идёт в свои внутренние C структуры (`struct sysinfo`)
4. Форматирует в текст, отдаёт
5. Файла на диске нет

---

## 6.7. Системный вызов vs функция библиотеки C

```
пользовательский код → функция libc → системный вызов → ядро
```

**Функция libc** — обёртка в user space (`/lib/libc.so`):
```c
printf("hello")    // libc
fopen("file.txt")  // libc
malloc(1024)       // libc
```

**Системный вызов** — переход в kernel space:
```c
write(1, "hello", 5)   // syscall
open("file.txt", ...)  // syscall
brk(...)               // syscall
```

Связь:
```c
printf()  →  write()      // форматирование + буферизация → syscall
fopen()   →  open()       // обёртка над syscall
malloc()  →  brk()/mmap() // просит память у ядра
```

В конце цепочки всегда syscall — только ядро умеет работать с железом. Можно писать без libc напрямую через syscall-ы (Go, Rust, embedded).

```bash
strace ls   # только syscall-ы
ltrace ls   # только функции библиотек
```

---

## 6.8. Динамическое изменение параметров ядра

**sysctl** — system control:

```bash
sysctl -a                          # все параметры (-a = all)
sysctl net.ipv4.ip_forward         # прочитать
sysctl -w net.ipv4.ip_forward=1    # записать (-w = write)
sysctl -n net.ipv4.ip_forward      # только значение (-n = no name)
sysctl -p                          # применить из /etc/sysctl.conf (-p = apply)

# то же через /proc напрямую:
echo 1 > /proc/sys/net/ipv4/ip_forward
```

Структура параметра = путь в `/proc/sys/`:
```
net.ipv4.ip_forward → /proc/sys/net/ipv4/ip_forward
```

### Постоянно:
```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

### Основные параметры:

**Сеть:**
```bash
net.ipv4.ip_forward=1              # маршрутизация пакетов (нужно для Docker, VPN)
net.ipv4.tcp_syncookies=1          # защита от SYN flood
net.core.somaxconn=65535           # очередь входящих соединений
```

**Память:**
```bash
vm.swappiness=10                   # агрессивность использования swap (0-100)
vm.dirty_ratio=15                  # % RAM для грязных страниц
```

**Ядро:**
```bash
kernel.pid_max=65536               # максимальный PID
kernel.dmesg_restrict=1            # запретить обычным пользователям читать dmesg
```

**Файловая система:**
```bash
fs.file-max=2097152                # максимум открытых файлов в системе
fs.inotify.max_user_watches=524288 # максимум отслеживаемых файлов (для IDE)
```

---

## 6.9. Параметр `quiet` и изменение параметров через GRUB

**`quiet`** — подавляет сообщения ядра при загрузке. Без него — поток текста инициализации. С `quiet` — тишина или графическая заставка (`splash`).

Временно изменить — через редактор GRUB при загрузке (см. 6.3).
Постоянно — через `/etc/default/grub` + `update-grub`.

---

## 6.10. /proc vs /sys

```
/proc — что происходит (процессы, память, статистика ядра)
/sys  — на чём происходит (железо, устройства, драйверы)
```

**/proc:**
```bash
/proc/PID/       # всё о процессе
/proc/meminfo    # память
/proc/cpuinfo    # CPU
/proc/sys/       # параметры ядра (sysctl)
```

**/sys (sysfs)** — структурированный, появился позже как замена части /proc:
```bash
/sys/class/net/eth0/     # сетевой интерфейс
/sys/block/sda/          # блочное устройство
/sys/bus/pci/devices/    # PCI устройства

# пример — изменить яркость:
echo 500 > /sys/class/backlight/intel_backlight/brightness
```

Оба дают прямой доступ к ядру — просто к разным его частям. `/proc` исторически появился раньше, немного хаотичный. `/sys` структурированный, специально для устройств.
