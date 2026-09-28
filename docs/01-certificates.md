# Глава 1. Сертификаты

## Зачем

В кластере три вида сетевых разговоров, и все они должны быть зашифрованы и проверяемы:

| Кто → кому | Порт | Что защищаем |
|---|---|---|
| etcd ↔ etcd | 2380 (peer) | репликацию данных DCS между узлами etcd |
| Patroni/etcdctl → etcd | 2379 (client) | ключ лидера, конфигурацию кластера, пароль пользователя etcd |
| HAProxy/patronictl → Patroni | 8008 (REST API) | проверки здоровья, команды switchover/restart с Basic Auth |
| клиенты → PostgreSQL | 5432 | SSL в самой базе (тот же сертификат узла) |

Схема: **один свой CA** (центр сертификации) и **по одному сертификату на узел**.
Каждый участник доверяет CA и поэтому доверяет любому сертификату, который CA подписал.

Ключевое слово — **SAN (Subject Alternative Name)**: список имён и IP, для которых сертификат
действителен. Клиент, подключаясь к `https://172.31.56.72:2379`, проверяет, что `172.31.56.72`
есть в SAN сертификата сервера. Нет в SAN — соединение отвергается (`x509: certificate is valid for …, not …`).

## 1.1 CA (на psql01)

```bash
source /root/cluster.env
mkdir -p /root/pki && cd /root/pki && chmod 700 /root/pki

openssl genrsa -out ca.key 4096
openssl req -x509 -new -key ca.key -sha256 -days 3650 \
  -subj "/CN=${SCOPE}-ca" -out ca.crt

openssl x509 -in ca.crt -noout -subject -dates
openssl x509 -in ca.crt -noout -text | grep -A1 "Basic Constraints"   # CA:TRUE
```

`ca.key` — самый ценный файл кластера: кто им владеет, может выпустить сертификат,
которому поверит весь кластер. Он остаётся только на psql01 в `/root/pki`.

## 1.2 Сертификаты узлов (на psql01)

Один сертификат на psql-узел — его используют и etcd, и Patroni, и PostgreSQL.
В SAN: короткое имя, FQDN, `localhost`, IP узла, `127.0.0.1` и VIP (клиенты, пришедшие через
VIP, увидят этот сертификат у PostgreSQL).

```bash
cd /root/pki
for pair in "$PSQL1:$PSQL1_IP" "$PSQL2:$PSQL2_IP" "$PSQL3:$PSQL3_IP"; do
  name=${pair%%:*}; ip=${pair##*:}
  openssl genrsa -out $name.key 2048
  openssl req -new -key $name.key -subj "/CN=$name" -out $name.csr
  cat > $name.ext <<EOF
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = DNS:$name, DNS:$name.ipa.ipotekabank.uz, DNS:localhost, IP:$ip, IP:127.0.0.1, IP:$VIP
EOF
  openssl x509 -req -in $name.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
    -days 825 -sha256 -extfile $name.ext -out $name.crt
done
ls -l
```

`extendedKeyUsage = serverAuth, clientAuth` — сертификат годится и для сервера (etcd принимает
подключения), и для клиента (etcd сам подключается к соседям по 2380).

Проверьте, что всё подписано нашим CA и SAN правильный:

```bash
for n in $PSQL1 $PSQL2 $PSQL3; do
  openssl verify -CAfile ca.crt $n.crt
  openssl x509 -in $n.crt -noout -ext subjectAltName
done
```

## 1.3 Раздать файлы

Каждому psql-узлу нужны `ca.crt`, `<узел>.crt`, `<узел>.key`. HA-узлам — только `ca.crt`.

С psql01 (если root по SSH между узлами открыт):

```bash
cd /root/pki
for n in $PSQL2 $PSQL3; do
  ssh $n mkdir -p /root/pki
  scp ca.crt $n.crt $n.key $n:/root/pki/
done
for n in $HA1 $HA2; do
  ssh $n mkdir -p /root/pki
  scp ca.crt $n:/root/pki/
done
```

Если SSH между узлами закрыт — перенесите эти же файлы через свою машину (MobaXterm/WinSCP)
в `/root/pki/` на каждом узле.

На каждом узле закройте права: `chmod 700 /root/pki; chmod 600 /root/pki/*.key`.

## ✅ Проверка главы

На каждом psql-узле:

```bash
source /root/cluster.env; cd /root/pki
openssl verify -CAfile ca.crt $ME.crt                     # OK
openssl x509 -noout -modulus -in $ME.crt | md5sum
openssl rsa  -noout -modulus -in $ME.key | md5sum         # хэши совпадают: ключ от этого сертификата
```

На ha-узлах: `ls -l /root/pki/ca.crt`.
