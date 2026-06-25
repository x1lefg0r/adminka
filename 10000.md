# Тема 16: Контейнеризация — Docker и изоляция

## 16.1. Контейнеры vs виртуальные машины

### Виртуальная машина (гипервизор):
```
[железо]
[гипервизор (VMware, KVM, VirtualBox)]
[гостевая ОС 1] [гостевая ОС 2] [гостевая ОС 3]
[приложение]    [приложение]    [приложение]
```
- Каждая VM имеет своё ядро
- Полная изоляция
- Тяжёлые — гигабайты памяти, минуты на запуск

### Контейнер (ОС-виртуализация):
```
[железо]
[ядро Linux (одно на всех)]
[контейнер 1] [контейнер 2] [контейнер 3]
[приложение]  [приложение]  [приложение]
```
- Все контейнеры используют одно ядро хоста
- Изоляция через namespaces и cgroups
- Лёгкие — мегабайты памяти, секунды на запуск

### Ключевое различие:
```
VM  — полная виртуализация железа, своё ядро
контейнер — изоляция процессов, общее ядро
```

Контейнер = процесс на хосте с иллюзией изолированной среды.

---

## 16.2. ENTRYPOINT vs CMD в Dockerfile

```dockerfile
FROM ubuntu

ENTRYPOINT ["/usr/bin/nginx"]   # основная команда, не перезаписывается
CMD ["-g", "daemon off;"]       # аргументы по умолчанию, перезаписываются
```

### Как взаимодействуют:
```bash
# дефолтный запуск:
docker run myimage
# → /usr/bin/nginx -g "daemon off;"

# переопределить CMD:
docker run myimage -c /etc/nginx/custom.conf
# → /usr/bin/nginx -c /etc/nginx/custom.conf

# переопределить ENTRYPOINT:
docker run --entrypoint /bin/bash myimage
# → /bin/bash
```

### Форматы:
```dockerfile
# exec форма (рекомендуется) — прямой запуск, PID 1:
ENTRYPOINT ["/usr/bin/nginx"]
CMD ["-g", "daemon off;"]

# shell форма — запускается через /bin/sh -c:
ENTRYPOINT nginx
CMD -g daemon off;
# проблема: сигналы (SIGTERM) не доходят до процесса — идут в sh
```

**Правило:** всегда используй exec форму (массив) — сигналы доходят правильно.

---

## 16.3. Запустить контейнер в фоне с пробросом порта

```bash
docker run -d -p 8080:80 --rm nginx
# -d = detach (фоновый режим, daemon)
# -p host_port:container_port (проброс порта)
# --rm = удалить контейнер после остановки
```

Полезные флаги `docker run`:
```bash
-d                    # фоновый режим
-p 8080:80            # проброс порта хост:контейнер
-v /host:/container   # bind mount
--name mycontainer    # имя контейнера
--rm                  # удалить после остановки
-e VAR=value          # переменная окружения
--network mynet       # сеть
-m 512m               # лимит памяти
--cpus 1.5            # лимит CPU
--restart always      # политика перезапуска
```

---

## 16.4. Именованные тома vs bind-монтирования

### Именованный том (named volume):
```bash
docker volume create mydata
docker run -v mydata:/app/data nginx
```
- Управляется Docker (`/var/lib/docker/volumes/`)
- Переносимый между контейнерами
- Лучше для продакшн данных
- Не зависит от структуры хоста

### Bind-монтирование:
```bash
docker run -v /host/path:/container/path nginx
```
- Монтирует конкретную директорию хоста
- Хост и контейнер видят одни файлы
- Удобно для разработки (изменения на хосте видны в контейнере)
- Зависит от структуры хоста — менее переносимо

```
named volume → продакшн, данные БД, постоянные данные
bind mount   → разработка, конфиги, логи доступные с хоста
```

---

## 16.5. Список запущенных контейнеров

```bash
docker ps              # запущенные контейнеры
docker ps -a           # все контейнеры включая остановленные
docker ps -q           # только ID контейнеров
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Вывод `docker ps`:
```
CONTAINER ID  IMAGE   COMMAND   CREATED   STATUS   PORTS              NAMES
abc123        nginx   nginx...  1h ago    Up 1h    0.0.0.0:80->80/tcp myserver
```

---

## 16.6. PID namespace и проблема PID 1

### PID namespace:
Каждый контейнер имеет своё пространство PID — процессы контейнера не видят процессы хоста.

```
хост:  PID 1=systemd, PID 100=nginx, PID 200=docker
контейнер: PID 1=nginx (внутри контейнера)
           хостовый PID nginx может быть 5678
```

### Проблема PID 1 в контейнере:
PID 1 в Linux имеет особую роль — усыновляет осиротевшие процессы и обрабатывает сигналы.

Обычный процесс как PID 1 в контейнере:
```
docker stop контейнер → SIGTERM → nginx (PID 1)
nginx не обрабатывает SIGTERM как PID 1 → 10 секунд → SIGKILL
```

Решения:
```dockerfile
# 1. exec форма в Dockerfile (процесс становится PID 1 напрямую):
ENTRYPOINT ["/usr/bin/nginx", "-g", "daemon off;"]

# 2. tini — минимальный init для контейнеров:
RUN apt install -y tini
ENTRYPOINT ["/usr/bin/tini", "--", "/usr/bin/nginx"]

# 3. --init флаг:
docker run --init nginx
```

---

## 16.7. Ограничение памяти и CPU

```bash
docker run -m 512m --cpus 1.5 nginx
# -m 512m    = максимум 512MB памяти
# --cpus 1.5 = максимум 1.5 ядра CPU
```

Под капотом Docker использует cgroups:
```bash
# посмотреть лимиты контейнера:
docker inspect mycontainer | grep -i memory
docker stats mycontainer   # реальное потребление в реальном времени
```

`docker stats` показывает: CPU%, MEM Usage/Limit, NET I/O, BLOCK I/O.

---

## 16.8. Сетевые драйверы Docker

### bridge (дефолт):
```
контейнеры в изолированной сети docker0
NAT для выхода в интернет
контейнеры общаются между собой по имени
```

### host:
```
контейнер использует сеть хоста напрямую
нет изоляции сети
максимальная производительность
docker run --network host nginx
```

### none:
```
нет сети вообще
полная сетевая изоляция
docker run --network none nginx
```

```bash
docker network ls                    # список сетей
docker network create mynet          # создать сеть
docker run --network mynet nginx     # подключить к сети
docker network inspect mynet         # подробности
```

---

## 16.9. Запустить shell в работающем контейнере

```bash
docker exec -it mycontainer /bin/bash
# exec  = выполнить команду в работающем контейнере
# -i    = interactive (держать stdin открытым)
# -t    = tty (псевдотерминал)
# -it   = интерактивный терминал вместе

# если bash нет (минимальный образ):
docker exec -it mycontainer /bin/sh
```

Другие полезные exec:
```bash
docker exec mycontainer ls /etc/nginx     # выполнить без интерактива
docker exec mycontainer nginx -t          # проверить конфиг nginx
docker exec -u root mycontainer whoami   # от конкретного пользователя
```

---

## 16.10. Шесть типов namespaces Linux

Namespaces — механизм изоляции процессов в ядре Linux. Docker использует все шесть:

```
PID     — изоляция процессов (свои PID, не видят хост)
Network — изоляция сети (свои интерфейсы, таблица маршрутизации)
Mount   — изоляция файловой системы (своё дерево директорий)
UTS     — изоляция hostname и domainname (своё имя хоста)
IPC     — изоляция межпроцессного взаимодействия (shared memory, очереди)
User    — изоляция пользователей (root внутри = не root снаружи)
```

```bash
# посмотреть namespaces процесса:
ls -la /proc/1234/ns/
```

cgroups (control groups) — дополняют namespaces: не изолируют, а **ограничивают** ресурсы (CPU, память, I/O).

```
namespaces = что процесс ВИДИТ
cgroups    = сколько ресурсов процесс ПОЛУЧАЕТ
```

---

## 16.11. nginx с пробросом порта в фоне

```bash
docker run -d -p 8080:80 nginx
```

Проверить:
```bash
curl http://localhost:8080   # должен ответить nginx
docker ps                    # видим запущенный контейнер
docker logs mycontainer      # логи контейнера
```

---

## 16.12. User namespaces

Без user namespaces:
```
root внутри контейнера = root на хосте
прорыв из контейнера → root на хосте
```

С user namespaces:
```
UID 0 (root) внутри → UID 65534 (nobody) на хосте
прорыв из контейнера → непривилегированный пользователь на хосте
```

```bash
# включить в Docker:
# /etc/docker/daemon.json:
{
  "userns-remap": "default"
}

systemctl restart docker
```

После включения Docker создаёт пользователя `dockremap` и маппит UID:
```
внутри контейнера: UID 0-65535
на хосте: UID 100000-165535
```

---

## Основные команды Docker

```bash
# образы:
docker images                    # список образов
docker pull nginx                # скачать образ
docker build -t myapp .          # собрать образ
docker rmi myimage               # удалить образ
docker image prune               # удалить неиспользуемые

# контейнеры:
docker run -d -p 80:80 nginx     # запустить
docker ps                        # запущенные
docker ps -a                     # все
docker stop mycontainer          # остановить (SIGTERM → SIGKILL)
docker kill mycontainer          # убить (SIGKILL сразу)
docker rm mycontainer            # удалить остановленный
docker rm -f mycontainer         # удалить принудительно
docker logs -f mycontainer       # логи в реальном времени
docker exec -it mycontainer bash # войти в контейнер
docker inspect mycontainer       # полная информация
docker stats                     # потребление ресурсов

# тома:
docker volume ls                 # список томов
docker volume create mydata      # создать том
docker volume rm mydata          # удалить том
docker volume prune              # удалить неиспользуемые

# сети:
docker network ls                # список сетей
docker network create mynet      # создать
docker network rm mynet          # удалить

# система:
docker system prune              # удалить всё неиспользуемое
docker system df                 # использование диска
```

---

## 16.14. Многоэтапный Dockerfile (*)

```dockerfile
# Этап 1 — сборка:
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Этап 2 — продакшн образ:
FROM node:18-alpine
WORKDIR /app

# копируем только собранное из этапа сборки:
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

# непривилегированный пользователь:
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# корректная пересылка сигналов:
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]

CMD ["node", "dist/index.js"]
```

Плюсы многоэтапной сборки:
```
финальный образ не содержит:
- инструменты сборки (компилятор, npm dev зависимости)
- исходный код
- временные файлы
→ меньше размер, меньше attack surface
```

### Почему tini для сигналов:
```
docker stop → SIGTERM → PID 1 (node)
node не всегда правильно обрабатывает SIGTERM как PID 1
→ 10 секунд ожидания → SIGKILL → некорректное завершение

с tini:
docker stop → SIGTERM → tini (PID 1) → node → корректное завершение
```
