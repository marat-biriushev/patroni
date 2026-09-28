# Глава 0. Подготовка и сброс

## 0.1 Убрать то, что оставил Ansible

Если на psql-узлах уже запускалась Ansible-роль etcd, начинаем с чистого листа.
**На всех psql:**

```bash
systemctl disable --now etcd 2>/dev/null
rm -rf /var/lib/etcd/* /etc/etcd /etc/profile.d/etcdctl.sh \
       /usr/bin/etcd /usr/bin/etcdctl /usr/bin/etcdutl \
       /etc/systemd/system/etcd.service /var/tmp/patroni-rpms
systemctl daemon-reload
```

Порты в firewalld, если их открыл Ansible, можно оставить — они всё равно понадобятся.

## 0.2 Файл с переменными кластера

Один и тот же файл на **всех пяти узлах**. Он избавляет от опечаток в IP и паролях:
дальше во всех командах используются переменные из него.

```bash
cat > /root/cluster.env <<'EOF'
# ---- Топология ----
export SCOPE=pg18-ipoteka-cluster        # имя кластера Patroni
export NAMESPACE=/db/                    # префикс ключей Patroni в etcd
export PSQL1=int-res-test-psql01 PSQL1_IP=172.31.56.71
export PSQL2=int-res-test-psql02 PSQL2_IP=172.31.56.72
export PSQL3=int-res-test-psql03 PSQL3_IP=172.31.56.73
export HA1=int-res-test-ha01 HA1_IP=172.31.56.74
export HA2=int-res-test-ha02 HA2_IP=172.31.56.75
export VIP=172.31.56.76 VIP_PREFIX=24

# ---- Этот узел ----
export ME=$(hostname -s)
case "$ME" in
  "$PSQL1") export ME_IP=$PSQL1_IP ;;
  "$PSQL2") export ME_IP=$PSQL2_IP ;;
  "$PSQL3") export ME_IP=$PSQL3_IP ;;
  "$HA1")   export ME_IP=$HA1_IP ;;
  "$HA2")   export ME_IP=$HA2_IP ;;
  *) echo "Неизвестный узел $ME — проверьте cluster.env" ;;
esac

# ---- Пароли (придумайте свои) ----
export ETCD_ROOT_PASS='ChangeMe-etcd-root'
export ETCD_PATRONI_PASS='ChangeMe-etcd-patroni'
export PG_SUPERUSER_PASS='ChangeMe-postgres'
export PG_REPL_PASS='ChangeMe-replicator'
export PG_REWIND_PASS='ChangeMe-rewind'
export PATRONI_API_PASS='ChangeMe-restapi'
export HAPROXY_STATS_PASS='ChangeMe-stats'
export VRRP_PASS='Vrrp1234'              # не длиннее 8 символов
EOF
chmod 600 /root/cluster.env
source /root/cluster.env
echo "$ME = $ME_IP"
```

> Пароли в открытом виде в файле — допустимо для учебного стенда. В проде их держат в vault/secret-хранилище.

## 0.3 Проверить исходное состояние

**На всех psql:**

```bash
cat /etc/redhat-release                       # RHEL 10.2
getenforce                                    # Enforcing — SELinux не выключаем
systemctl is-active firewalld                 # active
df -h /var/lib/pgsql /var/lib/etcd            # отдельные тома
rpm -q postgresql18-server postgresql18-contrib   # ...PGDG.rhel10.2
systemctl is-enabled postgresql-18            # masked — сервер будет запускать Patroni
ls -ld /var/lib/pgsql/18/data /var/lib/pgsql/18/wal   # пустые, postgres, drwx------
free -g; nproc
```

Почему `postgresql-18.service` замаскирован: PostgreSQL на этих узлах будет запускать **только**
Patroni. Если кто-то по привычке сделает `systemctl start postgresql-18`, на узле окажется
второй postmaster в обход Patroni — прямой путь к двум primary.

Если каталогов `data`/`wal` нет:

```bash
install -d -o postgres -g postgres -m 0700 /var/lib/pgsql/18/data /var/lib/pgsql/18/wal
```

## 0.4 Разложить офлайн-файлы

Скачанные вручную файлы нужны на **каждом psql-узле** в `/root/offline/`:

```
/root/offline/
├── etcd-v3.6.15-linux-amd64.tar.gz
└── rpms/
    ├── patroni-4.1.5-1PGDG.rhel10.2.noarch.rpm
    ├── patroni-etcd-4.1.5-1PGDG.rhel10.2.noarch.rpm
    ├── py-consul-1.6.0-45PGDG.rhel10.2.noarch.rpm
    ├── python3.12-etcd-0.4.5-50PGDG.rhel10.2.noarch.rpm
    ├── python3-psycopg2-2.9.13-42PGDG.rhel10.2.x86_64.rpm
    └── python3-ydiff-1.4.2-48PGDG.rhel10.2.noarch.rpm
```

Скопируйте через SFTP (MobaXterm/WinSCP) и проверьте:

```bash
ls -l /root/offline /root/offline/rpms
```

## ✅ Проверка главы

- `source /root/cluster.env` печатает правильные `ME`/`ME_IP` на всех пяти узлах.
- На psql-узлах: PostgreSQL 18 установлен, сервис замаскирован, каталоги пустые, офлайн-файлы на месте.
