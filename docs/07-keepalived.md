# Глава 7. keepalived и VIP

Всё в этой главе — **на обоих ha-узлах**, конфигурация отличается только ролью и приоритетом.

## Зачем

Два HAProxy — это два разных адреса. Приложению нужен **один**. keepalived держит общий
виртуальный IP (VIP, `172.31.56.76`) на одном из узлов и переносит его на другой, если первый
упал или на нём умер HAProxy.

Протокол — **VRRP**: узлы раз в секунду шлют друг другу объявления «я жив, мой приоритет N».
VIP держит узел с наибольшим приоритетом. Перестали приходить объявления от MASTER —
BACKUP забирает VIP и рассылает gratuitous ARP, чтобы коммутаторы узнали новый MAC.

- **unicast** вместо multicast: объявления идут напрямую на IP соседа. Multicast в
  корпоративных сетях часто режется.
- **track_script**: keepalived каждые 2 с проверяет, жив ли HAProxy. Если нет — снижает свой
  приоритет на `weight`. MASTER 150 − 60 = 90 < 100 у BACKUP → VIP переезжает, хотя сам узел жив.

## 6.1 Установка

```bash
dnf install keepalived
keepalived --version
```

## 6.2 Конфигурация

```bash
source /root/cluster.env
if [ "$ME" = "$HA1" ]; then STATE=MASTER; PRIO=150; PEER=$HA2_IP; else STATE=BACKUP; PRIO=100; PEER=$HA1_IP; fi
IFACE=$(ip -4 -o addr show | awk -v ip="$ME_IP" '$4 ~ "^"ip"/" {print $2}')
echo "$ME: $STATE prio=$PRIO peer=$PEER iface=$IFACE"

cp -n /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.orig
cat > /etc/keepalived/keepalived.conf <<EOF
global_defs {
    router_id $ME
    enable_script_security
    script_user root
}

vrrp_script chk_haproxy {
    script "/usr/bin/pidof haproxy"   # 0 — процесс есть
    interval 2                         # проверять каждые 2 с
    fall 2                             # 2 неудачи — считаем мёртвым
    rise 2                             # 2 успеха — снова живым
    weight -60                         # на сколько снизить приоритет
}

vrrp_instance VI_PG {
    state $STATE
    interface $IFACE
    virtual_router_id 51               # одинаковый на обоих узлах, уникальный в сегменте сети
    priority $PRIO
    advert_int 1

    unicast_src_ip $ME_IP
    unicast_peer {
        $PEER
    }

    authentication {
        auth_type PASS
        auth_pass $VRRP_PASS
    }

    virtual_ipaddress {
        $VIP/$VIP_PREFIX dev $IFACE
    }

    track_script {
        chk_haproxy
    }
}
EOF
chmod 0600 /etc/keepalived/keepalived.conf
keepalived -t -f /etc/keepalived/keepalived.conf && echo CONFIG OK
```

`virtual_router_id` должен быть уникальным среди всех VRRP-групп в этом сегменте сети.
Если в сети уже есть другие keepalived — уточните у сетевиков, что 51 свободен.

## 6.3 Firewall

VRRP — это не TCP/UDP, а отдельный IP-протокол № 112. Разрешаем его только от соседа:

```bash
source /root/cluster.env
PEER=$([ "$ME" = "$HA1" ] && echo $HA2_IP || echo $HA1_IP)
firewall-cmd --permanent --add-rich-rule="rule family=ipv4 source address=$PEER protocol value=vrrp accept"
firewall-cmd --reload
firewall-cmd --list-rich-rules
```

Если VRRP не проходит, **оба** узла считают себя MASTER и оба поднимают VIP (split-brain) — это
первое, что проверять при странностях с VIP.

## 6.4 Запуск

Сначала ha01, потом ha02:

```bash
systemctl enable --now keepalived
journalctl -u keepalived -f        # ha01: "Entering MASTER STATE"; ha02: "Entering BACKUP STATE"
```

## ✅ Проверка главы

```bash
ip -br a | grep 172.31.56.76       # только на ha01
```

С любого psql-узла — подключение через VIP:

```bash
source /root/cluster.env
PGPASSWORD="$PG_SUPERUSER_PASS" /usr/pgsql-18/bin/psql -h $VIP -p 5000 -U postgres \
  -c "select inet_server_addr() as node, pg_is_in_recovery() as replica"     # лидер, f
PGPASSWORD="$PG_SUPERUSER_PASS" /usr/pgsql-18/bin/psql -h $VIP -p 5001 -U postgres \
  -c "select inet_server_addr() as node, pg_is_in_recovery() as replica"     # реплика, t
```

Повторите запрос на 5001 несколько раз — `node` будет чередоваться между репликами (roundrobin).

Статистика HAProxy в браузере: `http://172.31.56.76:7000/` (admin / `$HAPROXY_STATS_PASS`).
