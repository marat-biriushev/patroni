# Глава 3. Узел PostgreSQL: ядро, watchdog, пакеты Patroni

Всё в этой главе — **на всех psql**.

## 3.1 Память: vm.overcommit_memory

По умолчанию Linux обещает процессам больше памяти, чем есть (overcommit). Когда память
действительно кончается, приходит **OOM killer** и убивает «самый жирный» процесс — часто
postmaster. Для лидера это авария: PostgreSQL падает целиком.

С `vm.overcommit_memory = 2` ядро не раздаёт больше, чем может выполнить: процесс получит
ошибку выделения памяти (PostgreSQL обработает её в рамках одного запроса), а не SIGKILL.

Лимит тогда считается так: `CommitLimit = swap + RAM × overcommit_ratio / 100`.
При значении по умолчанию (50) и 7.5 ГБ RAM + 3 ГБ swap — ~6.7 ГБ, этого может не хватить.
Ставим 80.

```bash
cat > /etc/sysctl.d/90-patroni.conf <<'EOF'
vm.overcommit_memory = 2
vm.overcommit_ratio = 80
EOF
sysctl --system | grep overcommit
grep -E 'CommitLimit|Committed_AS' /proc/meminfo   # Committed_AS должен быть заметно меньше CommitLimit
```

## 3.2 Watchdog

Худший сценарий для HA — «зомби-лидер»: Patroni на лидере завис (или узел потерял связь с etcd),
реплики уже выбрали нового лидера, а старый PostgreSQL продолжает принимать записи.

Watchdog — это таймер в ядре. Patroni на лидере регулярно его «пинает». Если Patroni завис и
перестал пинать, по истечении таймаута ядро **перезагружает узел** — старый лидер гарантированно
перестаёт писать до того, как новый начнёт.

На виртуалках аппаратного watchdog обычно нет — используем программный `softdog`:

```bash
echo softdog > /etc/modules-load.d/softdog.conf
modprobe softdog

# /dev/watchdog должен открываться пользователем postgres (Patroni работает от него)
cat > /etc/udev/rules.d/99-patroni-watchdog.rules <<'EOF'
KERNEL=="watchdog", OWNER="postgres", GROUP="postgres", MODE="0600"
EOF
udevadm control --reload-rules
udevadm trigger --action=add --sysname-match=watchdog

ls -l /dev/watchdog      # crw------- postgres postgres
```

## 3.3 Пакеты Patroni

```bash
dnf install /root/offline/rpms/*.rpm
```

`dnf` поставит 6 пакетов из файлов и подтянет из AppStream/BaseOS/EPEL
`python3-click`, `python3-prettytable`, `python3-psutil`, `python3-wcwidth`.

```bash
patroni --version            # patroni 4.1.5
patronictl version
```

Зачем что:

| Пакет | Роль |
|---|---|
| `patroni` | сам Patroni и `patronictl` |
| `patroni-etcd` | метапакет: зависимости для работы с etcd |
| `python3.12-etcd`, `py-consul` | клиентские библиотеки DCS (требуются пакетами PGDG) |
| `python3-psycopg2` | драйвер PostgreSQL для Patroni |
| `python3-ydiff` | цветной diff в `patronictl edit-config` |

## 3.4 Сертификаты для Patroni и PostgreSQL

Patroni и PostgreSQL работают от `postgres`, ключ должен быть доступен только ему.

```bash
source /root/cluster.env
install -d -o postgres -g postgres -m 0750 /etc/patroni /etc/patroni/pki
install -o postgres -g postgres -m 0644 /root/pki/ca.crt  /etc/patroni/pki/ca.crt
install -o postgres -g postgres -m 0644 /root/pki/$ME.crt /etc/patroni/pki/server.crt
install -o postgres -g postgres -m 0600 /root/pki/$ME.key /etc/patroni/pki/server.key
ls -l /etc/patroni/pki
```

PostgreSQL откажется стартовать с SSL, если ключ доступен кому-то кроме владельца — отсюда `0600`.

## 3.5 Firewall

```bash
firewall-cmd --permanent --add-port=5432/tcp --add-port=8008/tcp
firewall-cmd --reload
firewall-cmd --list-ports      # 2379 2380 5432 8008 (+10050 zabbix)
```

## 3.6 io_method (PostgreSQL 18)

В PostgreSQL 18 появился асинхронный ввод-вывод, режим задаётся `io_method`:
`io_uring` (быстрее) или `worker` (по умолчанию). Проверим, доступен ли io_uring:

```bash
cat /proc/sys/kernel/io_uring_disabled      # 0 — разрешён, 1/2 — запрещён
ldd /usr/pgsql-18/bin/postgres | grep uring # собран ли postgres с liburing
```

В RHEL 10 io_uring по умолчанию запрещён ядром (это настройка безопасности, её не трогаем),
поэтому в главе 4 ставим `io_method: worker`. Параметр должен быть одинаковым на всех узлах.

## ✅ Проверка главы

```bash
sysctl vm.overcommit_memory vm.overcommit_ratio   # 2, 80
ls -l /dev/watchdog                               # владелец postgres
rpm -q patroni patroni-etcd                       # 4.1.5
ls -l /etc/patroni/pki                            # server.key 0600 postgres
firewall-cmd --list-ports
```
