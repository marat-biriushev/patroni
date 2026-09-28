# Глава 8. Проверки: switchover, failover, отказы

Кластер работает. Теперь ломаем его по-разному и смотрим, как он себя ведёт. Это главная часть:
только так становится понятно, зачем были нужны все предыдущие шаги.

Подготовка — два окна на psql-узле. В каждом новом окне сначала:
`source /root/cluster.env; source /etc/profile.d/patroni.sh`.

**Окно 1 — наблюдение за кластером:**
```bash
source /etc/profile.d/patroni.sh
watch -n1 patronictl list
```

**Окно 2 — непрерывная запись через VIP** (видно, сколько длится простой):
```bash
source /root/cluster.env
export PGPASSWORD="$PG_SUPERUSER_PASS"
PSQL="/usr/pgsql-18/bin/psql -h $VIP -p 5000 -U postgres -Atq"
$PSQL -c "create table if not exists ha_test(id serial, ts timestamptz default now(), node inet default inet_server_addr())"
while true; do
  $PSQL -c "insert into ha_test default values returning id, node" 2>&1 | sed "s/^/$(date +%T) /"
  sleep 1
done
```

## 7.1 Плановый switchover

Штатная смена лидера — для обновлений, перезагрузок, обслуживания:

```bash
patronictl switchover $SCOPE --leader int-res-test-psql01 --candidate int-res-test-psql02
```

Смотрим:
- в окне 1 лидер сменился, бывший лидер стал репликой (TL увеличился на 1);
- в окне 2 — несколько секунд ошибок, потом записи идут с новым `node`.

Верните лидера обратно: `patronictl switchover $SCOPE --leader int-res-test-psql02 --candidate int-res-test-psql01`.

## 7.2 Аварийный failover: остановка Patroni на лидере

```bash
# на текущем лидере
systemctl stop patroni
```

Patroni при остановке гасит PostgreSQL и **снимает ключ лидера** — синхронная реплика
становится лидером почти сразу. Верните узел: `systemctl start patroni` — он вернётся как реплика
(при необходимости через `pg_rewind`).

## 7.3 Жёсткий отказ лидера

Имитируем внезапную смерть — ключ лидера никто не снимает:

```bash
# на текущем лидере
pkill -9 -f /usr/bin/patroni; pkill -9 -f /usr/pgsql-18/bin/postgres
```

Теперь новый лидер появится только через `ttl` (до 30 с), когда истечёт ключ в etcd.
Засеките время в окне 2. Потом `systemctl start patroni` на пострадавшем узле.

Если в 7.3 узел **перезагрузился сам** — это сработал watchdog: Patroni не пинал `/dev/watchdog`,
и ядро перезагрузило узел, чтобы старый лидер точно не продолжил писать.

## 7.4 Синхронная репликация

```bash
patronictl list        # кто Sync Standby
```

Остановите Patroni на **синхронной** реплике — через несколько секунд её роль перейдёт ко второй
реплике. Остановите обе реплики — запись на лидере продолжится (`synchronous_mode_strict: false`),
но без гарантии, что данные есть где-то ещё. В `strict: true` запись бы встала.

## 7.5 Отказ etcd

```bash
# на одном psql-узле
systemctl stop etcd
patronictl list                    # всё работает: кворум 2 из 3
```

Остановите etcd на **втором** узле. Кворума больше нет, продлить ключ лидера невозможно:
через `retry_timeout`/`ttl` лидер **понизит себя до read-only**, запись в окне 2 остановится.
Это правильно: без DCS нельзя гарантировать, что лидер один.

Верните etcd на обоих узлах (`systemctl start etcd`) — кластер восстановится сам.

## 7.6 Переезд VIP

```bash
# на ha01 (MASTER)
systemctl stop haproxy
```

Через ~4 с (`interval 2 × fall 2`) приоритет ha01 упадёт до 90, VIP переедет на ha02:

```bash
ip -br a | grep 172.31.56.76       # на ha02
journalctl -u keepalived -n 20     # на обоих: Entering BACKUP / MASTER STATE
```

Запись в окне 2 прервётся на пару секунд и продолжится.
Верните: `systemctl start haproxy` на ha01 — VIP вернётся на ha01 (у него выше приоритет).

Жёстче: выключите ha01 целиком (`poweroff`) — VIP переедет, как только перестанут приходить
VRRP-объявления (~3 с).

## 7.7 Изменение конфигурации кластера

Динамическая конфигурация (DCS) меняется не в файле, а командой:

```bash
patronictl show-config
patronictl edit-config                        # откроется редактор; или без редактора:
patronictl edit-config -p 'work_mem=16MB' --force
patronictl list                               # колонка Pending restart — если параметр требует рестарта
patronictl restart $SCOPE --pending           # перезапустить только те узлы, где нужно
```

Правка `bootstrap.dcs` в `patroni.yml` после инициализации **ни на что не влияет** — частая ошибка.

## Что должно остаться в голове

| Компонент | Что будет, если он упадёт |
|---|---|
| Один etcd | ничего, кворум 2/3 |
| Два etcd | лидер уходит в read-only, запись стоит до восстановления кворума |
| Patroni на лидере (штатно) | быстрый failover, секунды |
| Узел лидера (жёстко) | failover через ≤ ttl (30 с) |
| Синхронная реплика | роль sync переходит ко второй реплике |
| HAProxy на MASTER | VIP переезжает на BACKUP за ~4 с |
| Узел ha MASTER | VIP переезжает за ~3 с |
