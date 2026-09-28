# Глава 6. HAProxy

Всё в этой главе — **на обоих ha-узлах** (ha01 и ha02).

## Зачем

Приложение не должно знать, какой узел сейчас лидер: после failover лидер другой.
HAProxy принимает подключения на постоянных портах и сам направляет их куда нужно:

| Порт | Куда | Как HAProxy узнаёт |
|---|---|---|
| 5000 | только лидер (запись) | `GET https://<узел>:8008/primary` → 200 только у лидера |
| 5001 | реплики (чтение) | `GET https://<узел>:8008/replica` → 200 только у здоровых реплик |
| 7000 | страница статистики | — |

HAProxy работает в режиме `tcp`: он не разбирает протокол PostgreSQL, а просто пересылает
байты. Поэтому PostgreSQL видит подключения с IP HAProxy (мы разрешили их в pg_hba, глава 5).

HAProxy слушает на VIP — адресе, которого на BACKUP-узле в обычном состоянии нет.
Чтобы он всё равно мог запуститься, разрешаем привязку к «чужим» адресам: `ip_nonlocal_bind`.

## 5.1 Установка

```bash
dnf install haproxy          # из RHEL AppStream, HAProxy 3.0
haproxy -v
```

## 5.2 Привязка к VIP

```bash
cat > /etc/sysctl.d/90-keepalived.conf <<'EOF'
net.ipv4.ip_nonlocal_bind = 1
EOF
sysctl --system | grep nonlocal
```

## 5.3 CA для проверок

HAProxy проверяет сертификат REST API Patroni — ему нужен наш CA:

```bash
install -d -o root -g haproxy -m 0750 /etc/haproxy/pki
install -o root -g haproxy -m 0644 /root/pki/ca.crt /etc/haproxy/pki/ca.crt
```

## 5.4 Конфигурация

```bash
source /root/cluster.env
cp -n /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.orig
cat > /etc/haproxy/haproxy.cfg <<EOF
global
    log         /dev/log local0
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats mode 660 level admin

defaults
    mode                    tcp
    log                     global
    option                  tcplog
    option                  dontlognull
    retries                 2
    timeout queue           5s
    timeout connect         4s
    timeout client          30m
    timeout server          30m
    timeout check           5s

listen stats
    mode http
    bind $VIP:7000
    stats enable
    stats uri /
    stats refresh 10s
    stats auth admin:$HAPROXY_STATS_PASS

# Запись: только лидер
listen primary
    bind $VIP:5000
    option httpchk
    http-check send meth GET uri /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server $PSQL1 $PSQL1_IP:5432 maxconn 1000 check port 8008 check-ssl verify required ca-file /etc/haproxy/pki/ca.crt
    server $PSQL2 $PSQL2_IP:5432 maxconn 1000 check port 8008 check-ssl verify required ca-file /etc/haproxy/pki/ca.crt
    server $PSQL3 $PSQL3_IP:5432 maxconn 1000 check port 8008 check-ssl verify required ca-file /etc/haproxy/pki/ca.crt

# Чтение: реплики
listen replicas
    bind $VIP:5001
    balance roundrobin
    option httpchk
    http-check send meth GET uri /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server $PSQL1 $PSQL1_IP:5432 maxconn 1000 check port 8008 check-ssl verify required ca-file /etc/haproxy/pki/ca.crt
    server $PSQL2 $PSQL2_IP:5432 maxconn 1000 check port 8008 check-ssl verify required ca-file /etc/haproxy/pki/ca.crt
    server $PSQL3 $PSQL3_IP:5432 maxconn 1000 check port 8008 check-ssl verify required ca-file /etc/haproxy/pki/ca.crt
EOF
chmod 0640 /etc/haproxy/haproxy.cfg && chgrp haproxy /etc/haproxy/haproxy.cfg
haproxy -c -f /etc/haproxy/haproxy.cfg        # Configuration file is valid
```

Разбор главного:

- `check port 8008` — трафик идёт на 5432, а проверка здоровья — на REST API Patroni (8008).
- `inter 3s fall 3 rise 2` — проверять каждые 3 с; 3 неудачи подряд — сервер DOWN, 2 успеха — UP.
  Значит, после смены лидера HAProxy переключится за ~6–9 с.
- `on-marked-down shutdown-sessions` — когда сервер помечен DOWN, существующие соединения
  к нему рвутся. Иначе приложение осталось бы подключено к бывшему лидеру (теперь реплике)
  и получало ошибки «read-only transaction».
- `timeout client/server 30m` — простаивающее соединение закрывается через 30 минут.
  Пул соединений приложения (HikariCP) должен иметь `maxLifetime` меньше (например 25 мин)
  или `keepaliveTime` ~5 мин.

## 5.5 SELinux

HAProxy работает в домене `haproxy_t`, и SELinux разрешает ему слушать только «веб-порты».
Помечаем наши порты как `http_port_t` и разрешаем подключаться к любым портам (5432, 8008):

```bash
dnf install policycoreutils-python-utils       # semanage, если его нет
for p in 5000 5001 7000; do
  semanage port -a -t http_port_t -p tcp $p 2>/dev/null || semanage port -m -t http_port_t -p tcp $p
done
semanage port -l | grep -w http_port_t
setsebool -P haproxy_connect_any on
getsebool haproxy_connect_any                  # on
```

Если что-то не работает — смотрите отказы SELinux: `ausearch -m avc -ts recent`.

## 5.6 Firewall и запуск

```bash
firewall-cmd --permanent --add-port=5000/tcp --add-port=5001/tcp --add-port=7000/tcp
firewall-cmd --reload

systemctl enable --now haproxy
systemctl status haproxy --no-pager
```

## ✅ Проверка главы

VIP ещё не поднят (это глава 7), поэтому проверяем через сокет статистики:

```bash
echo "show stat" | socat stdio /var/lib/haproxy/stats 2>/dev/null | cut -d, -f1,2,18 | column -s, -t \
  || echo "socat не установлен — смотрите журнал ниже"
journalctl -u haproxy --no-pager -n 30
```

Ожидаем:
- в `primary`: лидер **UP**, два других **DOWN** — так и задумано, это не ошибка;
- в `replicas`: две реплики **UP**, лидер **DOWN**.

Если все серверы DOWN — проверьте с ha-узла доступность REST API:

```bash
curl -s -o /dev/null -w '%{http_code}\n' --cacert /etc/haproxy/pki/ca.crt https://$PSQL1_IP:8008/primary
```
