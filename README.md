# PostgreSQL 18 + Patroni: the hard way

Ручная сборка отказоустойчивого кластера PostgreSQL 18 под Patroni на RHEL 10 — без Ansible,
каждый шаг руками и с проверкой. Цель — понять, из чего состоит кластер и почему он устроен
именно так. По мотивам [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way).

```
                     приложения
                         │
               VIP 172.31.56.76 (keepalived)
         ┌───────────────┴───────────────┐
   int-res-test-ha01               int-res-test-ha02
   172.31.56.74 MASTER             172.31.56.75 BACKUP
   HAProxy :5000 → primary   :5001 → replicas   :7000 → stats
         └───────────────┬───────────────┘
         ┌───────────────┼───────────────┐
 int-res-test-psql01  int-res-test-psql02  int-res-test-psql03
   172.31.56.71         172.31.56.72         172.31.56.73
   PostgreSQL 18 + Patroni (REST :8008) + etcd (:2379 клиенты, :2380 кластер)
```

## Главы

| # | Глава | Где выполнять |
|---|---|---|
| 0 | [Подготовка и сброс](docs/00-prepare.md) | все узлы |
| 1 | [Сертификаты (CA и TLS)](docs/01-certificates.md) | psql01, затем раздать |
| 2 | [etcd — распределённое хранилище (DCS)](docs/02-etcd.md) | psql01–03 |
| 3 | [Узел PostgreSQL: ядро, watchdog, пакеты Patroni](docs/03-node.md) | psql01–03 |
| 4 | [Patroni](docs/04-patroni.md) | psql01–03 |
| 5 | [HAProxy](docs/05-haproxy.md) | ha01–02 |
| 6 | [keepalived и VIP](docs/06-keepalived.md) | ha01–02 |
| 7 | [Проверки: switchover, failover, отказы](docs/07-tests.md) | везде |

Идти строго по порядку: каждая глава опирается на предыдущую и заканчивается проверкой.
Если проверка не проходит — не идите дальше, сначала разберитесь.

## Исходное состояние

- RHEL 10.2, SELinux включён, firewalld включён, интернета нет.
- На psql-узлах уже стоит PostgreSQL 18 из PGDG (`postgresql18-server`, `-contrib`),
  `initdb` не выполнялся; `/var/lib/pgsql` (150G) и `/var/lib/etcd` (5G) — отдельные тома.
- Пакеты, которых нет в зеркале, скачаны вручную (см. [главу 0](docs/00-prepare.md)):
  6 RPM Patroni и архив `etcd-v3.6.15-linux-amd64.tar.gz`.

## Соглашения

- Все команды — от `root` (`sudo -i`).
- Блок, помеченный **на всех psql**, выполняется на каждом из трёх узлов БД.
- Переменные (IP, пароли) берутся из файла `/root/cluster.env`, который создаётся в главе 0.
  Перед работой в новой сессии: `source /root/cluster.env`.
