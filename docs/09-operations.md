# Глава 9. Эксплуатация

Кластер собран и проверен. Эта глава — что делать с ним каждый день, при плановых работах
и при авариях. Все команды — от `root` на psql-узле, если не сказано иное.

Перед работой в новой сессии:

```bash
source /root/cluster.env
source /etc/profile.d/etcdctl.sh
source /etc/profile.d/patroni.sh
R="--user root:$ETCD_ROOT_PASS"          # логин etcd для etcdctl
```

## 9.1 Ежедневная проверка (1 минута)

```bash
patronictl list                           # 1 Leader, 1 Sync Standby, 1 Replica; все streaming, лаг 0, одинаковый TL
etcdctl $R endpoint health                # 3 x healthy
df -h /var/lib/pgsql /var/lib/etcd        # < 80%
```

На ha-узлах:

```bash
ip -br a | grep 172.31.56.76              # VIP ровно на одном узле
systemctl is-active haproxy keepalived    # active active
```

Статистика HAProxy: `http://172.31.56.76:7000/` (admin / `$HAPROXY_STATS_PASS`): в `primary` один
сервер UP, в `replicas` — две реплики UP.

## 9.2 Мониторинг (Zabbix)

### Что отслеживать

| Метрика | Норма | Тревога |
|---|---|---|
| Число лидеров | ровно 1 | 0 (нет записи) или > 1 (split-brain) |
| Состояние узла Patroni | `running` / `streaming` | `stopped`, `starting` дольше 5 минут |
| Лаг реплики | 0 | > 16 МБ дольше 5 минут |
| Кворум etcd | 3 healthy | < 3 — предупреждение, < 2 — авария |
| Место `/var/lib/pgsql`, `/var/lib/etcd` | < 80% | ≥ 80% |
| Срок сертификатов | > 30 дней | ≤ 30 дней |
| VIP | на одном ha-узле | нет ни на одном / на обоих |
| HAProxy, keepalived | active | не active |

### Скрипт проверок для агента Zabbix (на psql-узлах)

Агент работает от пользователя `zabbix` и не может читать `/etc/patroni` и `/etc/etcd`.
Поэтому ему нужна своя копия CA (это открытый сертификат, не секрет), а данные берутся по сети
через REST API Patroni и `/health` etcd — они не требуют пароля.

```bash
install -d -m 0755 /etc/zabbix/pgha
install -m 0644 /root/pki/ca.crt /etc/zabbix/pgha/ca.crt

cat > /usr/local/bin/pgha-check <<'EOF'
#!/bin/bash
# Проверки HA-кластера для Zabbix. Использование: pgha-check <метрика>
CA=/etc/zabbix/pgha/ca.crt
IP=$(hostname -I | awk '{print $1}')
API="https://$IP:8008"
json() { python3 -c "import sys,json; d=json.load(sys.stdin); $1" 2>/dev/null; }

case "$1" in
  role)        curl -s --cacert $CA $API/patroni | json 'print(d.get("role",""))' ;;          # primary / replica
  state)       curl -s --cacert $CA $API/patroni | json 'print(d.get("state",""))' ;;         # running / streaming
  leaders)     curl -s --cacert $CA $API/cluster | json 'print(sum(m["role"]=="leader" for m in d["members"]))' ;;
  members)     curl -s --cacert $CA $API/cluster | json 'print(len(d["members"]))' ;;
  max_lag)     curl -s --cacert $CA $API/cluster | json 'print(max([m["lag"] for m in d["members"] if isinstance(m.get("lag"),int)] or [0]))' ;;
  etcd_health) curl -s --cacert $CA https://$IP:2379/health | json 'print(1 if d.get("health")=="true" else 0)' ;;
  cert_days)   end=$(echo | openssl s_client -connect $IP:8008 2>/dev/null | openssl x509 -noout -enddate | cut -d= -f2)
               echo $(( ( $(date -d "$end" +%s) - $(date +%s) ) / 86400 )) ;;
  *) echo "usage: $0 role|state|leaders|members|max_lag|etcd_health|cert_days"; exit 1 ;;
esac
EOF
chmod 0755 /usr/local/bin/pgha-check
restorecon -v /usr/local/bin/pgha-check

for m in role state leaders members max_lag etcd_health cert_days; do printf "%-12s %s\n" $m "$(pgha-check $m)"; done
```

Подключение к агенту (путь каталога зависит от версии агента: `zabbix_agentd.d` или `zabbix_agent2.d`):

```bash
cat > /etc/zabbix/zabbix_agentd.d/pgha.conf <<'EOF'
UserParameter=pgha[*],/usr/local/bin/pgha-check $1
EOF
systemctl restart zabbix-agent          # или zabbix-agent2
```

Элементы данных в Zabbix: `pgha[role]`, `pgha[leaders]`, `pgha[max_lag]`, `pgha[etcd_health]`,
`pgha[cert_days]` и т.д. Триггеры — по таблице выше, например
`last(/host/pgha[leaders])<>1`, `last(/host/pgha[max_lag])>16777216`, `last(/host/pgha[cert_days])<30`.

> Если SELinux блокирует `curl` из агента (видно в `ausearch -m avc -ts recent`), включите
> `setsebool -P zabbix_can_network on`.

Метрики самой БД (подключения, блокировки, размер, слоты репликации) удобнее снимать штатным
шаблоном «PostgreSQL by Zabbix agent 2» с отдельной ролью:

```sql
create role zbx_monitor login password '...' in role pg_monitor;   -- на лидере, через VIP:5000
```

Подключение агента — `127.0.0.1:5432` (pg_hba `host all all 127.0.0.1/32 scram-sha-256` уже есть).

## 9.3 Резервные копии

HA защищает от падения узла, но **не** от `DROP TABLE`, ошибочной миграции или логической
порчи данных: такие изменения мгновенно реплицируются на все узлы. Нужны:

- pgBackRest: полный бэкап еженедельно, дифференциальный ежедневно, непрерывный архив WAL
  (восстановление на любой момент времени);
- регулярная проверка восстановления на отдельной машине — бэкап без проверенного
  восстановления считается отсутствующим.

Пакет `pgbackrest` лежит в PGDG `common` (`pgbackrest-2.59.1-1PGDG.rhel10.2.x86_64.rpm`).
Настройка — отдельная глава (в планах).

## 9.4 Регламентные операции

### Плановая смена лидера

```bash
patronictl switchover $SCOPE --candidate <sync-standby> --force
```

Кандидат — только `Sync Standby`. Отложенная: `--scheduled "2026-10-01T02:00"`.

### Изменение параметров PostgreSQL

Параметры, общие для кластера (`max_connections`, `shared_buffers`, `work_mem`…), меняются
**только** через DCS, а не в `patroni.yml` (`bootstrap.dcs` после инициализации не читается):

```bash
patronictl show-config
patronictl edit-config -p 'work_mem=16MB' --force        # или интерактивно: patronictl edit-config
patronictl list                                           # колонка Pending restart
patronictl restart $SCOPE --pending --force               # если параметр требует рестарта
```

`patronictl restart` перезапускает узлы по одному; для лидера — короткий простой записи.
Чтобы его избежать: сначала перезапустить реплики, затем switchover, затем бывшего лидера.

### Изменение pg_hba (новая сеть приложений)

pg_hba задан в локальном `patroni.yml` — правка на **всех трёх** узлах:

```bash
# добавить строку в секцию postgresql.pg_hba, например:
#   - host all all 10.20.0.0/16 scram-sha-256
vi /etc/patroni/patroni.yml
systemctl reload patroni                 # Patroni перепишет pg_hba.conf и сделает reload PostgreSQL
sudo -iu postgres psql -c "select * from pg_hba_file_rules where error is not null"   # пусто
```

Через HAProxy отдельные строки не нужны: PostgreSQL видит адреса ha-узлов, они уже разрешены.

### Перезагрузка / обслуживание узла

- **Реплика** — можно просто перезагрузить: Patroni стартует сам и догонит лидера.
- **Лидер** — сначала `switchover`, потом работы.
- **Несколько узлов подряд** — строго по одному, дожидаясь `streaming` в `patronictl list`.
- **Работы на etcd или сети между узлами** — поставьте автоматику на паузу, чтобы Patroni не
  понизил лидера из-за временной недоступности DCS:
  ```bash
  patronictl pause $SCOPE
  # ... работы ...
  patronictl resume $SCOPE
  ```

### Обновление PostgreSQL (минорное, 18.x → 18.y)

Бинарники обновляются на всех узлах, но работающий PostgreSQL продолжает использовать старые до рестарта:

```bash
# на каждой реплике по очереди:
dnf update postgresql18\*
patronictl restart $SCOPE <реплика> --force
# затем:
patronictl switchover $SCOPE --candidate <sync-standby> --force
# на бывшем лидере (теперь реплике):
dnf update postgresql18\*
patronictl restart $SCOPE <бывший-лидер> --force
```

Мажорное обновление (18 → 19) — отдельная процедура (`pg_upgrade` при остановленном кластере
или логическая репликация), не делать этим способом.

### Обновление Patroni

По одному узлу: реплики, затем switchover, затем бывший лидер.

```bash
dnf install /path/to/patroni-4.x.y*.rpm /path/to/patroni-etcd-4.x.y*.rpm
systemctl restart patroni                # на реплике — безопасно
```

### Пересоздание сломанной реплики

Если реплика не догоняет лидера или не может выполнить `pg_rewind`:

```bash
patronictl reinit $SCOPE <узел> --force  # удалит данные узла и скопирует заново с лидера
```

### Продление сертификатов

Тем же CA (глава 2.2) на psql01, раздать файлы (2.3), затем:

```bash
# на каждом psql, по одному, дожидаясь etcdctl endpoint health:
install -o root -g etcd -m 0644 /root/pki/$ME.crt /etc/etcd/pki/server.crt
install -o root -g etcd -m 0640 /root/pki/$ME.key /etc/etcd/pki/server.key
systemctl restart etcd
etcdctl $R endpoint health
# Patroni и PostgreSQL — без рестарта:
install -o postgres -g postgres -m 0644 /root/pki/$ME.crt /etc/patroni/pki/server.crt
install -o postgres -g postgres -m 0600 /root/pki/$ME.key /etc/patroni/pki/server.key
systemctl reload patroni
sudo -iu postgres psql -c "select pg_reload_conf()"
```

### Замена узла etcd

```bash
etcdctl $R member list -w table
etcdctl $R member remove <ID старого узла>
etcdctl $R member add <имя> --peer-urls=https://<IP>:2380
```

На новом узле — конфиг из главы 3, но `initial-cluster-state: existing` и пустой `/var/lib/etcd`.
Во время замены в кластере 2 узла из 3: отказ ещё одного = потеря кворума.

## 9.5 Для приложений

| Назначение | Адрес |
|---|---|
| Запись (и чтение, требующее актуальности) | `172.31.56.76:5000` |
| Чтение с реплик | `172.31.56.76:5001` |

Роли и базы создаются на лидере через VIP:

```sql
create role app_user login password '...';
create database app_db owner app_user;
```

Настройки пула соединений (HikariCP):

- `maxLifetime` меньше 30 минут (например 25 мин): HAProxy закрывает соединения, простаивающие 30 минут;
- приложение должно повторять транзакцию при обрыве соединения: при смене лидера HAProxy
  рвёт все соединения к старому лидеру (5–35 с простоя в зависимости от сценария, глава 8).

## 9.6 Разбор аварий

### Первые команды

```bash
patronictl list                                   # кто лидер, кто отстал, какой TL
patronictl history                                # история смен лидера
journalctl -u patroni --since "-30 min"           # что решал Patroni
ls -lt /var/lib/pgsql/18/data/log/ | head         # логи самого PostgreSQL
etcdctl $R endpoint status -w table               # состояние etcd
```

### Типовые ситуации

| Симптом | Причина | Действие |
|---|---|---|
| Нет лидера, запись `read-only` у всех | нет кворума etcd | поднять etcd; Patroni восстановится сам (глава 8.4) |
| Бывший лидер после аварии `running`, но не `streaming`, пустой TL | не удался `pg_rewind` | `journalctl -u patroni`; если не решается — `patronictl reinit` |
| Реплика отстаёт и лаг растёт | сеть, диск, долгие запросы на реплике | `pg_stat_replication` на лидере; при необходимости `reinit` |
| Узел перезагрузился сам | сработал watchdog: Patroni завис или потерял DCS | `last -x reboot`; журнал прошлой загрузки (нужен persistent journal) |
| VIP на обоих ha-узлах | VRRP между ними не проходит | firewalld (`--list-rich-rules`), сеть; журнал keepalived |
| HAProxy: все серверы DOWN | нет лидера или проблемы с REST API / TLS | `curl --cacert ... https://<узел>:8008/primary` с ha-узла |
| Место под WAL заканчивается | неактивный слот репликации или не работает архив | `select * from pg_replication_slots`; вернуть реплику или удалить мёртвый слот |

## 9.7 Что попросить у команды ОС

- **Постоянный журнал** (`Storage=persistent` в `/etc/systemd/journald.conf`): после срабатывания
  watchdog без него не видно, что было до перезагрузки.
- **Мониторинг места** на `/var/lib/pgsql` и `/var/lib/etcd`.
- **Ротацию логов PostgreSQL** трогать не нужно: `postgresql-%a.log` перезаписываются по дням недели.
- **NTP** на всех узлах: сертификаты и таймауты зависят от времени.
