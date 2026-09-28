# Глава 1. Установка PostgreSQL 18

Всё в этой главе — **на всех psql**.

> На тестовом стенде эта глава уже выполнена. Прочитайте её, чтобы понимать, что стоит на узлах,
> и выполните только блоки «Проверка». Полностью её проходят на новых узлах (например, в проде).

## 1.1 Диски

Данные PostgreSQL и etcd не должны жить на корневом разделе: если WAL или база заполнят `/`,
упадёт весь узел (журналы, sshd, etcd). Поэтому под них — отдельные тома.

На тестовых узлах диск 200G, из которых под ОС размечено ~30G, остальное свободно:

```bash
lsblk
parted -s /dev/sda unit GiB print free      # где начинается Free Space (у нас ~30GiB)
```

Разметка свободного места под LVM:

| Том | Размер | Точка монтирования | Зачем |
|---|---|---|---|
| `pgvg/pgdata` | 150G | `/var/lib/pgsql` | данные и WAL PostgreSQL |
| `pgvg/etcd` | 5G | `/var/lib/etcd` | данные etcd (глава 3) |
| свободно в VG | ~15G | — | запас на рост |

```bash
# Раздел на свободном месте (parted откажется, если начало пересечётся с существующим разделом)
parted -s -a optimal /dev/sda mkpart pgdata 30GiB 100%
parted -s /dev/sda set 5 lvm on
partprobe /dev/sda

# LVM
pvcreate /dev/sda5
vgcreate pgvg /dev/sda5
lvcreate -n pgdata -L 150G pgvg
lvcreate -n etcd   -L 5G   pgvg
mkfs.xfs /dev/pgvg/pgdata
mkfs.xfs /dev/pgvg/etcd

# Монтирование
mkdir -p /var/lib/pgsql /var/lib/etcd
echo '/dev/mapper/pgvg-pgdata /var/lib/pgsql xfs defaults,noatime 0 0' >> /etc/fstab
echo '/dev/mapper/pgvg-etcd   /var/lib/etcd   xfs defaults,noatime 0 0' >> /etc/fstab
systemctl daemon-reload
mount -a
restorecon -Rv /var/lib/pgsql /var/lib/etcd      # вернуть правильные метки SELinux
```

`restorecon` нужен потому, что свежесмонтированная ФС приходит без меток SELinux, а
`/var/lib/pgsql` должен иметь тип `postgresql_db_t`.

`noatime` — не обновлять время последнего чтения файла при каждом чтении: для БД это лишние записи.

**Проверка:**

```bash
df -h /var/lib/pgsql /var/lib/etcd
ls -Zd /var/lib/pgsql            # ...postgresql_db_t...
```

«Used 3.0G» на пустом 150G XFS — это журнал и метаданные файловой системы, так и должно быть.

## 1.2 Репозиторий PGDG из внутреннего зеркала

Зеркало `http://mirror.ipotekabank.uz/repos/` повторяет структуру `download.postgresql.org`.
Для PostgreSQL 18 на RHEL 10 нужен один репозиторий:

```bash
cat > /etc/yum.repos.d/postgres-local.repo <<'EOF'
[postgres18]
name=PostgreSQL 18 Local Repo
baseurl=http://mirror.ipotekabank.uz/repos/postgre/yum/18/redhat/rhel-10-x86_64/
enabled=1
gpgcheck=0
priority=1
EOF

curl -s -o /dev/null -w '%{http_code}\n' \
  http://mirror.ipotekabank.uz/repos/postgre/yum/18/redhat/rhel-10-x86_64/repodata/repomd.xml   # 200
dnf -q repolist
```

### Зачем `priority=1`

В RHEL 10 AppStream есть **свои** пакеты `postgresql18*` от Red Hat. Версии совпадают (18.6-1),
но при сравнении release `el10_2` оказывается «больше», чем `PGDG`, и без приоритета `dnf`
возьмёт сервер из AppStream:

```bash
dnf -q list --showduplicates postgresql18-server
# 18.6-1.el10_2        rhel-10-for-x86_64-appstream-rpms   ← Red Hat
# 18.6-1PGDG.rhel10.2  postgres18                          ← PGDG
```

Сборка Red Hat раскладывает файлы иначе: нет `/usr/pgsql-18/bin`, другие каталоги и юниты.
Смешение пакетов Red Hat и PGDG ломает установку, а Patroni из PGDG рассчитан на раскладку PGDG.

`priority=1` (по умолчанию у всех репозиториев 99): если пакет с одним именем есть в нескольких
репозиториях, `dnf` берёт его **только** из репозитория с наивысшим приоритетом.
`redhat.repo` при этом не трогаем.

### RHEL 9

В RHEL 9 в AppStream есть модуль `postgresql`, который перекрывает пакеты PGDG. Его отключают:

```bash
dnf -qy module disable postgresql      # только RHEL 9; в RHEL 10 модулей нет
```

## 1.3 Пакеты

```bash
dnf install postgresql18 postgresql18-server postgresql18-contrib
```

| Пакет | Что внутри |
|---|---|
| `postgresql18` | клиент `psql`, `pg_dump`, `pg_basebackup` и др. |
| `postgresql18-server` | сервер `postgres`, `initdb`, `pg_ctl`, `pg_rewind`, systemd-юнит |
| `postgresql18-contrib` | расширения (`pg_stat_statements` и др.) |
| `postgresql18-libs` | подтянется как зависимость |

Всё ставится в `/usr/pgsql-18/`, данные по умолчанию — `/var/lib/pgsql/18/data`.
Пакет создаёт системного пользователя `postgres` с домашним каталогом `/var/lib/pgsql`.

**Проверка — пакеты именно из PGDG:**

```bash
rpm -q postgresql18-server postgresql18-contrib     # ...-18.6-1PGDG.rhel10.2...
dnf -q info --installed postgresql18-server | grep -E 'Release|From repo'
/usr/pgsql-18/bin/postgres --version
```

## 1.4 Выключить штатный сервис

```bash
systemctl disable postgresql-18
systemctl mask postgresql-18
systemctl is-enabled postgresql-18     # masked
```

`mask` — ссылка `/etc/systemd/system/postgresql-18.service → /dev/null`. После неё службу нельзя
запустить никак: ни `systemctl start`, ни при загрузке, ни как зависимость.

Зачем: PostgreSQL на этих узлах запускает и останавливает **только Patroni**. Если кто-то по
привычке выполнит `systemctl start postgresql-18`, поднимется второй экземпляр в обход Patroni —
на реплике это путь к двум primary. Снять маску при необходимости: `systemctl unmask postgresql-18`.

## 1.5 Каталоги данных и WAL

```bash
install -d -o postgres -g postgres -m 0700 /var/lib/pgsql/18/data /var/lib/pgsql/18/wal
ls -ld /var/lib/pgsql/18/data /var/lib/pgsql/18/wal
```

- `0700` — PostgreSQL откажется запускаться, если каталог данных доступен кому-то кроме владельца.
- WAL в отдельном каталоге: при `initdb --waldir` `pg_wal` станет ссылкой на него. Позже его можно
  вынести на отдельный диск, не трогая данные.

## 1.6 Без initdb

`initdb` здесь **не выполняем**. Кластер инициализирует Patroni (глава 5): на первом узле он сам
сделает `initdb` с нужными параметрами, создаст служебные роли и станет лидером, а на остальных
узлах сделает копию через `pg_basebackup`. Если сделать `initdb` руками на всех трёх узлах,
получатся три независимые базы, и реплики из них не собрать.

## 1.7 Удобства для пользователя postgres

Бинарники PGDG лежат в `/usr/pgsql-18/bin`, которого нет в `PATH`. Домашний `.bash_profile` из
пакета PGDG подключает файл `~/.pgsql_profile`, если он есть:

```bash
cat > /var/lib/pgsql/.pgsql_profile <<'EOF'
export PATH=/usr/pgsql-18/bin:$PATH
export PGDATA=/var/lib/pgsql/18/data
export PATRONICTL_CONFIG_FILE=/etc/patroni/patroni.yml
EOF
chown postgres:postgres /var/lib/pgsql/.pgsql_profile
sudo -iu postgres which psql          # /usr/pgsql-18/bin/psql
```

## ✅ Проверка главы

```bash
df -h /var/lib/pgsql /var/lib/etcd                     # отдельные тома
grep priority /etc/yum.repos.d/postgres-local.repo     # priority=1
rpm -q postgresql18-server postgresql18-contrib        # ...PGDG.rhel10.2
systemctl is-enabled postgresql-18                     # masked
ls -ld /var/lib/pgsql/18/data /var/lib/pgsql/18/wal    # postgres, drwx------, пустые
ls -A /var/lib/pgsql/18/data | wc -l                   # 0 — initdb не делался
sudo -iu postgres which psql                           # /usr/pgsql-18/bin/psql
```
