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
| 1 | [Установка PostgreSQL 18](docs/01-postgresql.md) | psql01–03 |
| 2 | [Сертификаты (CA и TLS)](docs/02-certificates.md) | psql01, затем раздать |
| 3 | [etcd — распределённое хранилище (DCS)](docs/03-etcd.md) | psql01–03 |
| 4 | [Узел PostgreSQL: ядро, watchdog, пакеты Patroni](docs/04-node.md) | psql01–03 |
| 5 | [Patroni](docs/05-patroni.md) | psql01–03 |
| 6 | [HAProxy](docs/06-haproxy.md) | ha01–02 |
| 7 | [keepalived и VIP](docs/07-keepalived.md) | ha01–02 |
| 8 | [Проверки: switchover, failover, отказы](docs/08-tests.md) | везде |
| 9 | [Эксплуатация: мониторинг, регламент, аварии](docs/09-operations.md) | везде |

Идти строго по порядку: каждая глава опирается на предыдущую и заканчивается проверкой.
Если проверка не проходит — не идите дальше, сначала разберитесь.

## Исходное состояние

- RHEL 10.2, SELinux включён, firewalld включён, интернета нет.
- На тестовом стенде PostgreSQL 18 уже установлен и тома размечены — [глава 1](docs/01-postgresql.md)
  описывает, как это делается; там её достаточно прочитать и выполнить только проверки.
- Пакеты, которых нет в зеркале, скачаны вручную (см. [главу 0](docs/00-prepare.md)):
  6 RPM Patroni и архив `etcd-v3.6.15-linux-amd64.tar.gz`.

## Соглашения

- Все команды — от `root` (`sudo -i`).
- Блок, помеченный **на всех psql**, выполняется на каждом из трёх узлов БД.
- Переменные (IP, пароли) берутся из файла `/root/cluster.env`, который создаётся в главе 0.
  Перед работой в новой сессии: `source /root/cluster.env`.
