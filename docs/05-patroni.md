# Глава 5. Patroni

## Как это работает

Patroni — демон на каждом узле БД. Он сам запускает и останавливает PostgreSQL и каждые
`loop_wait` (10 с) проходит один цикл:

1. Смотрит в etcd: есть ли ключ лидера и чей он.
2. **Если лидер — я**: продлеваю ключ лидера (TTL 30 с). Не смог продлить до истечения
   `retry_timeout` — сам себя понижаю до реплики (лучше без лидера, чем с двумя).
3. **Если лидер — другой**: работаю репликой, стримлю WAL с лидера.
4. **Если лидера нет**: пытаюсь захватить ключ. Побеждает самая «свежая» реплика
   (с минимальным отставанием), и только одна — etcd не даст записать ключ дважды.

Первый запуск (**bootstrap**): кто первым захватит ключ `initialize`, тот делает `initdb`,
создаёт служебные роли и становится лидером. Остальные делают `pg_basebackup` с него.

Конфигурация бывает двух видов:

- **локальная** — файл `/etc/patroni/patroni.yml` на каждом узле (адреса, пути, пароли);
- **динамическая (DCS)** — общая для кластера, хранится в etcd (`ttl`, `synchronous_mode`,
  параметры PostgreSQL, которые должны совпадать на всех узлах). Секция `bootstrap.dcs` в файле
  используется **только один раз** — при инициализации. Потом менять её — через `patronictl edit-config`.

## 4.1 patroni.yml (на всех psql)

```bash
source /root/cluster.env
cat > /etc/patroni/patroni.yml <<EOF
scope: $SCOPE
namespace: $NAMESPACE
name: $ME

log:
  level: INFO

# ---- REST API: по нему HAProxy узнаёт, кто primary, а patronictl отдаёт команды ----
restapi:
  listen: $ME_IP:8008
  connect_address: $ME_IP:8008
  certfile: /etc/patroni/pki/server.crt
  keyfile: /etc/patroni/pki/server.key
  cafile: /etc/patroni/pki/ca.crt
  authentication:                 # Basic Auth для небезопасных методов (POST/PATCH/DELETE)
    username: patroni
    password: '$PATRONI_API_PASS'

ctl:
  cacert: /etc/patroni/pki/ca.crt

# ---- DCS ----
etcd3:
  hosts: $PSQL1_IP:2379,$PSQL2_IP:2379,$PSQL3_IP:2379
  protocol: https
  cacert: /etc/patroni/pki/ca.crt
  username: patroni
  password: '$ETCD_PATRONI_PASS'

# ---- Используется только при первой инициализации кластера ----
bootstrap:
  dcs:
    ttl: 30                        # сколько живёт ключ лидера без продления
    loop_wait: 10                  # период цикла Patroni
    retry_timeout: 10              # сколько лидер терпит недоступность etcd/PostgreSQL
    maximum_lag_on_failover: 1048576   # реплика с отставанием > 1 МБ не станет лидером
    synchronous_mode: true         # коммит ждёт подтверждения от синхронной реплики
    synchronous_mode_strict: false # если синхронных реплик нет — не блокировать запись
    postgresql:
      use_pg_rewind: true          # бывший лидер догоняет нового через pg_rewind, а не полную копию
      use_slots: true              # слоты репликации: лидер не удалит WAL, нужный реплике
      parameters:
        wal_level: replica
        hot_standby: "on"
        io_method: worker          # io_uring в RHEL 10 запрещён ядром (глава 4.6)
        max_connections: 200
        shared_buffers: 1900MB     # ~25% RAM
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"        # нужно для pg_rewind
        wal_keep_size: 1GB
  initdb:
    - encoding: UTF8
    - locale: C.UTF-8
    - data-checksums
    - waldir: /var/lib/pgsql/18/wal

# ---- Локальные настройки PostgreSQL на этом узле ----
postgresql:
  listen: $ME_IP,127.0.0.1:5432
  connect_address: $ME_IP:5432
  data_dir: /var/lib/pgsql/18/data
  bin_dir: /usr/pgsql-18/bin
  pgpass: /var/lib/pgsql/.pgpass_patroni
  authentication:                  # эти роли Patroni создаст при bootstrap
    superuser:
      username: postgres
      password: '$PG_SUPERUSER_PASS'
    replication:
      username: replicator
      password: '$PG_REPL_PASS'
    rewind:
      username: rewind_user
      password: '$PG_REWIND_PASS'
  parameters:
    unix_socket_directories: /run/postgresql
    password_encryption: scram-sha-256
    ssl: "on"
    ssl_cert_file: /etc/patroni/pki/server.crt
    ssl_key_file: /etc/patroni/pki/server.key
    ssl_ca_file: /etc/patroni/pki/ca.crt
  pg_hba:
    - local all all peer
    - host all all 127.0.0.1/32 scram-sha-256
    # Patroni подключается к СВОЕМУ PostgreSQL по протоколу репликации (узнать timeline
    # и LSN перед pg_rewind). Без этой строки бывший лидер после аварии не встанет в строй.
    - host replication replicator 127.0.0.1/32 scram-sha-256
    # репликация и pg_rewind между узлами БД
    - host replication replicator $PSQL1_IP/32 scram-sha-256
    - host replication replicator $PSQL2_IP/32 scram-sha-256
    - host replication replicator $PSQL3_IP/32 scram-sha-256
    - host all rewind_user $PSQL1_IP/32 scram-sha-256
    - host all rewind_user $PSQL2_IP/32 scram-sha-256
    - host all rewind_user $PSQL3_IP/32 scram-sha-256
    # клиенты через HAProxy: PostgreSQL видит адрес HAProxy, а не приложения
    - host all all $HA1_IP/32 scram-sha-256
    - host all all $HA2_IP/32 scram-sha-256
  basebackup:
    - checkpoint: fast
    - waldir: /var/lib/pgsql/18/wal

watchdog:
  mode: automatic                  # использовать, если /dev/watchdog доступен
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false                # true — узел никогда не станет лидером
  noloadbalance: false             # true — исключить из балансировки чтения
  nosync: false                    # true — не делать синхронной репликой
  clonefrom: false
EOF
chown postgres:postgres /etc/patroni/patroni.yml
chmod 0600 /etc/patroni/patroni.yml          # внутри пароли

# проверка синтаксиса и значений
sudo -u postgres patroni --validate-config /etc/patroni/patroni.yml && echo CONFIG OK
```

> Пароли в файле взяты в одинарные кавычки YAML. Если в пароле есть `'` — замените его.

`--validate-config` может ругаться на то, что порт уже занят или каталог данных пуст — до первого
запуска это нормально. Важно, чтобы не было ошибок синтаксиса и неизвестных параметров.

## 4.2 systemd-юнит (на всех psql)

```bash
cat > /etc/systemd/system/patroni.service <<'EOF'
[Unit]
Description=Patroni PostgreSQL HA
After=network-online.target etcd.service
Wants=network-online.target

[Service]
Type=simple
User=postgres
Group=postgres
# Меньше арен malloc в glibc — меньше потребление памяти (наследуют и процессы postgres)
Environment=MALLOC_ARENA_MAX=1
ExecStart=/usr/bin/patroni /etc/patroni/patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID
# Останавливаем только Patroni: он сам корректно остановит PostgreSQL
KillMode=process
TimeoutSec=30
# Не перезапускать автоматически: после падения Patroni решение принимает человек
Restart=no

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload

echo 'export PATRONICTL_CONFIG_FILE=/etc/patroni/patroni.yml' > /etc/profile.d/patroni.sh
```

## 4.3 Первый узел — лидер (только psql01)

```bash
systemctl enable --now patroni
journalctl -u patroni -f
```

В логе увидите: `trying to bootstrap a new cluster` → вывод `initdb` →
`initialized a new cluster` → `no action. I am (int-res-test-psql01), the leader with the lock`.

```bash
source /etc/profile.d/patroni.sh
patronictl list
```

```
+ Cluster: pg18-ipoteka-cluster ----+--------+---------+----+-----------+
| Member              | Host        | Role   | State   | TL | Lag in MB |
+---------------------+-------------+--------+---------+----+-----------+
| int-res-test-psql01 | 172.31.56.71| Leader | running |  1 |           |
+---------------------+-------------+--------+---------+----+-----------+
```

Загляните, что Patroni записал в etcd:

```bash
source /root/cluster.env; source /etc/profile.d/etcdctl.sh
etcdctl --user "patroni:$ETCD_PATRONI_PASS" get /db/ --prefix --keys-only
etcdctl --user "patroni:$ETCD_PATRONI_PASS" get /db/$SCOPE/leader
```

Ключи: `config` (динамическая конфигурация), `initialize` (ID кластера), `leader`
(кто лидер), `members/<узел>` (состояние каждого узла), `status`/`sync`.

## 4.4 Реплики (psql02, затем psql03)

```bash
systemctl enable --now patroni
journalctl -u patroni -f
```

В логе: `bootstrap from leader` → `pg_basebackup` → `replica has been created` →
`no action. I am (…), a secondary, and following a leader (int-res-test-psql01)`.

Пока psql02 копирует данные, psql03 не запускайте — по одному проще разобраться, если что-то пойдёт не так.

## ✅ Проверка главы

```bash
patronictl list
```

Ожидаем: один `Leader`, один `Sync Standby`, один `Replica`, все `streaming`/`running`,
одинаковый `TL`, `Lag in MB` = 0.

REST API — то, что будет спрашивать HAProxy:

```bash
source /root/cluster.env
for ip in $PSQL1_IP $PSQL2_IP $PSQL3_IP; do
  printf "%s primary=%s replica=%s\n" $ip \
    $(curl -s -o /dev/null -w '%{http_code}' --cacert /etc/patroni/pki/ca.crt https://$ip:8008/primary) \
    $(curl -s -o /dev/null -w '%{http_code}' --cacert /etc/patroni/pki/ca.crt https://$ip:8008/replica)
done
# лидер: primary=200 replica=503; реплики: primary=503 replica=200
```

Репликация глазами PostgreSQL (на лидере):

```bash
sudo -iu postgres /usr/pgsql-18/bin/psql -c "select application_name, client_addr, state, sync_state from pg_stat_replication"
sudo -iu postgres /usr/pgsql-18/bin/psql -c "show synchronous_standby_names"
```

Проверка SSL и watchdog:

```bash
PGPASSWORD="$PG_SUPERUSER_PASS" /usr/pgsql-18/bin/psql "host=127.0.0.1 user=postgres sslmode=require" \
  -c "select ssl, version from pg_stat_ssl where pid = pg_backend_pid()"   # t, TLSv1.3
journalctl -u patroni | grep -i watchdog      # "Software Watchdog activated" на лидере
```
