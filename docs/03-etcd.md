# Глава 3. etcd — хранилище конфигурации кластера (DCS)

## Зачем

Patroni на каждом узле сам по себе не знает, кто сейчас лидер. Эту информацию хранит **DCS**
(Distributed Configuration Store) — у нас etcd. Главное в нём — **ключ лидера с TTL**:

- лидер Patroni каждые `loop_wait` секунд продлевает ключ `/db/<scope>/leader`;
- если лидер умер и не продлил ключ, через `ttl` секунд ключ исчезает, и реплики устраивают выборы;
- записать ключ может только один — это и есть защита от двух primary.

etcd сам по себе кластер из 3 узлов с протоколом **Raft**: запись проходит, только если с ней
согласно большинство (**кворум**, 2 из 3). Поэтому узлов нечётное число: 3 переживают отказ одного,
5 — двух. Два узла etcd хуже одного: отказ любого из двух ломает кворум.

У нас etcd живёт на тех же трёх psql-узлах, данные — на отдельном томе `/var/lib/etcd`.

## 2.1 Установка (на всех psql)

```bash
cd /root/offline
tar xzf etcd-v3.6.15-linux-amd64.tar.gz
install -m 0755 etcd-v3.6.15-linux-amd64/{etcd,etcdctl,etcdutl} /usr/local/bin/
restorecon -v /usr/local/bin/etcd*          # правильная метка SELinux (bin_t)
etcd --version
```

`install`, а не `mv`: файл создаётся заново и получает метку SELinux каталога назначения.
После `mv` из `/root` осталась бы метка `admin_home_t`, и systemd не смог бы запустить бинарник.

Системный пользователь и каталоги:

```bash
id etcd 2>/dev/null || useradd --system --no-create-home --shell /sbin/nologin etcd
install -d -o etcd -g etcd -m 0700 /var/lib/etcd
install -d -o root -g etcd -m 0750 /etc/etcd /etc/etcd/pki

source /root/cluster.env
install -o root -g etcd -m 0644 /root/pki/ca.crt     /etc/etcd/pki/ca.crt
install -o root -g etcd -m 0644 /root/pki/$ME.crt    /etc/etcd/pki/server.crt
install -o root -g etcd -m 0640 /root/pki/$ME.key    /etc/etcd/pki/server.key
ls -l /etc/etcd/pki
```

## 2.2 Конфигурация (на всех psql)

```bash
source /root/cluster.env
cat > /etc/etcd/etcd.yml <<EOF
name: $ME
data-dir: /var/lib/etcd

# Порт 2380 — общение узлов etcd между собой (Raft)
listen-peer-urls: https://$ME_IP:2380
initial-advertise-peer-urls: https://$ME_IP:2380

# Порт 2379 — клиенты (Patroni, etcdctl)
listen-client-urls: https://$ME_IP:2379,https://127.0.0.1:2379
advertise-client-urls: https://$ME_IP:2379

# Состав кластера при первом запуске
initial-cluster: $PSQL1=https://$PSQL1_IP:2380,$PSQL2=https://$PSQL2_IP:2380,$PSQL3=https://$PSQL3_IP:2380
initial-cluster-token: $SCOPE-etcd
initial-cluster-state: new

auto-compaction-mode: periodic
auto-compaction-retention: "1"
logger: zap
log-level: info
log-outputs: [stderr]

# TLS для клиентов: шифрование + проверка сервера клиентом.
# trusted-ca-file здесь НЕ указываем: с ним etcd потребует от клиентов сертификат,
# а наши клиенты входят по логину/паролю.
client-transport-security:
  cert-file: /etc/etcd/pki/server.crt
  key-file: /etc/etcd/pki/server.key
  client-cert-auth: false

# TLS между узлами: взаимная проверка сертификатов (mTLS)
peer-transport-security:
  cert-file: /etc/etcd/pki/server.crt
  key-file: /etc/etcd/pki/server.key
  trusted-ca-file: /etc/etcd/pki/ca.crt
  client-cert-auth: true
EOF
chown root:etcd /etc/etcd/etcd.yml && chmod 0640 /etc/etcd/etcd.yml
cat /etc/etcd/etcd.yml
```

`initial-cluster-state: new` действует только при самом первом старте с пустым `data-dir`.
Потом состав кластера хранится в самих данных etcd, и эта строка игнорируется.

## 2.3 systemd-юнит (на всех psql)

```bash
cat > /etc/systemd/system/etcd.service <<'EOF'
[Unit]
Description=etcd key-value store
Documentation=https://etcd.io/docs/
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=etcd
Group=etcd
ExecStart=/usr/local/bin/etcd --config-file /etc/etcd/etcd.yml
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
```

`Type=notify`: etcd сообщает systemd «я готов» только когда вошёл в кластер с кворумом.
Поэтому первый узел, запущенный в одиночку, будет «стартовать», пока не поднимутся соседи.

## 2.4 Firewall (на всех psql)

```bash
firewall-cmd --permanent --add-port=2379/tcp --add-port=2380/tcp
firewall-cmd --reload
firewall-cmd --list-ports
```

Без этого узлы голосуют каждый сам за себя (`received 1 MsgPreVoteResp votes`) и кластер не собирается.

## 2.5 Запуск

Запустите на **всех трёх** узлах, не дожидаясь каждого (`--no-block` — не ждать готовности):

```bash
systemctl enable --now --no-block etcd
```

Смотрите, как узлы находят друг друга и выбирают лидера:

```bash
journalctl -u etcd -f        # ждём "elected leader" / "became leader at term"
```

## 2.6 Удобное окружение для etcdctl (на всех psql)

```bash
source /root/cluster.env
cat > /etc/profile.d/etcdctl.sh <<EOF
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=https://$PSQL1_IP:2379,https://$PSQL2_IP:2379,https://$PSQL3_IP:2379
export ETCDCTL_CACERT=/etc/etcd/pki/ca.crt
EOF
source /etc/profile.d/etcdctl.sh

etcdctl endpoint health
etcdctl endpoint status -w table
etcdctl member list -w table
```

Ожидаем: три `is healthy`, ровно один `IS LEADER = true`, одинаковый `RAFT TERM`.

## 2.7 Аутентификация

Пока любой, кто дотянется до 2379, может читать и писать всё. Включаем RBAC:
пользователь `root` (администратор) и пользователь `patroni`, которому разрешён только префикс `/db/`.

Выполнить **один раз** на любом узле:

```bash
source /root/cluster.env; source /etc/profile.d/etcdctl.sh

# root: пароль передаём через stdin (--interactive=false), чтобы не светить его в истории
echo "$ETCD_ROOT_PASS" | etcdctl user add root --interactive=false
etcdctl role add root 2>/dev/null; etcdctl user grant-role root root

# роль и пользователь для Patroni — только свой префикс
etcdctl role add patroni
etcdctl role grant-permission patroni --prefix=true readwrite "$NAMESPACE"
echo "$ETCD_PATRONI_PASS" | etcdctl user add patroni --interactive=false
etcdctl user grant-role patroni patroni

etcdctl auth enable
```

С этого момента без логина etcd ничего не отдаёт — все команды ниже идут с `--user`.

Логин передаём как `--user имя:пароль` — если указать только имя, etcdctl будет ждать ввод
пароля с клавиатуры.

## ✅ Проверка главы

```bash
source /root/cluster.env; source /etc/profile.d/etcdctl.sh

etcdctl --user "root:$ETCD_ROOT_PASS" endpoint health   # 3 x healthy
etcdctl --user "root:$ETCD_ROOT_PASS" auth status         # Authentication Status: true
etcdctl --user "root:$ETCD_ROOT_PASS" user list           # patroni, root
etcdctl --user "root:$ETCD_ROOT_PASS" role get patroni    # readwrite [/db/, /db0)

# patroni может писать в свой префикс...
etcdctl --user "patroni:$ETCD_PATRONI_PASS" put /db/test ok
etcdctl --user "patroni:$ETCD_PATRONI_PASS" get /db/test
etcdctl --user "patroni:$ETCD_PATRONI_PASS" del /db/test
# ...и не может за его пределами
etcdctl --user "patroni:$ETCD_PATRONI_PASS" put /other x  # permission denied

# без логина — отказ
etcdctl get /db/ --prefix                                 # user name is empty
```

Эксперимент: остановите etcd на одном узле (`systemctl stop etcd`) — кластер продолжает работать
(2 из 3). Остановите второй — `endpoint health` у оставшегося покажет ошибку: кворума нет.
Верните оба (`systemctl start etcd`).
