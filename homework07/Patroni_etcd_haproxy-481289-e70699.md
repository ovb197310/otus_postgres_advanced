'yc' в составе Яндекс.Облако CLI для управления облачными ресурсами в Яндекс.Облако
<https://cloud.yandex.com/en/docs/cli/quickstart>

Подключаемся к Яндекс.Облако и выполняем конфигурацию окружения с помощью команды:

```bash
yc init
```

Проверяем установленную версию 'yc' (рекомендуется последняя доступная версия):

```bash
yc version
```

Список географических регионов и зон доступности для размещения VM:

```bash
yc compute zone list
```

```bash
yc config set compute-default-zone ru-central1-a && yc config get compute-default-zone
```

Далее будем использовать географический регион __ru-central1__ и зону доступности __ru-central1-a__.

- Список доступных типов дисков:

```bash
yc compute disk-type list
```

Далее будем использовать тип диска ‘network-hdd’.

-- Создаем сетевую инфраструктуру:

```bash
yc vpc network create \
    --name otus-net \
    --description "otus-net" \
```

```bash
yc vpc network list
```

```bash
yc vpc subnet create \
    --name otus-subnet \
    --range 10.0.0.0/24 \
    --network-name otus-net \
    --description "otus-subnet"
```

```bash
yc vpc subnet list
```

```bash
yc dns zone create --name otus-dns \
--zone staging. \
--private-visibility network-ids=enp4vcfcnhmq2eliqbps
```

```bash
yc dns zone list
```

- Сгенерируем SSH ключи:

```bash
ssh-keygen -t rsa -b 2048
```

```bash
ssh-add ~/.ssh/yc_key
```

- Развернем 3 ВМ для ETCD:

```bash
for i in {1..3}; do yc compute instance create --name etcd$i --hostname etcd$i --cores 2 \
  --memory 2 --create-boot-disk size=10G,type=network-hdd,image-folder-id=standard-images,\
  image-family=ubuntu-2004-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 \
  --ssh-key ~/.ssh/yc_key.pub & done;
```

- Развернем 3 ВМ для PostgreSQL:

```bash
for i in {1..3}; do yc compute instance create --name pgsql$i --hostname pgsql$i --cores 2 \
  --memory 4 --create-boot-disk size=10G,type=network-hdd,image-folder-id=standard-images, \
  image-family=ubuntu-2004-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 \
  --ssh-key ~/.ssh/yc_key.pub & done;
```

- Развернем ВМ для HAProxy:

```bash
yc compute instance create \
    --name haproxy \
    --hostname haproxy \
    --cores 2 \
    --memory 2 \
    --create-boot-disk size=10G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2004-lts \
    --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 \
    --ssh-key ~/.ssh/yc_key.pub
```

```bash
yc compute instance list
```

| id |name|zone|state|ext IP|int IP|
|--- |--- |--- |---  |---|---|
| fhm2dol6j8o8p7m1j95p | pgsql3  | ru-central1-a | RUNNING | 51.250.78.154  | 10.0.0.13   |
| fhm729go5cl1v83ovgk0 | etcd1   | ru-central1-a | RUNNING | 51.250.76.163  | 10.0.0.26   |
| fhm9t1ov1o43hhbaf52n | haproxy | ru-central1-a | RUNNING | 158.160.44.242 | 10.0.0.9    |
| fhmf6gktljvtg38ucv6r | etcd3   | ru-central1-a | RUNNING | 89.169.129.0   | 10.0.0.3    |
| fhmkjevfufv0v4ulkuos | etcd2   | ru-central1-a | RUNNING | 89.169.136.249 | 10.0.0.16   |
| fhmml052qfal4e1r5bi6 | pgsql1  | ru-central1-a | RUNNING | 89.169.135.75  | 10.0.0.27   |
| fhmq27vemn7t1jhnlpv5 | pgsql2  | ru-central1-a | RUNNING | 89.169.131.171 | 10.0.0.7

- Прописываем хосты в каждой ВМ:

```bash
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name pgsql$i | grep -E ' \+address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key \
yc-user@$vm_ip_address << EOF
sudo bash -c 'cat >> /etc/hosts <<EOL
10.0.0.26 etcd1
10.0.0.16 etcd2
10.0.0.3 etcd3
10.0.0.27 pgsql1
10.0.0.7 pgsql2
10.0.0.13 pgsql3
10.0.0.9 haproxy
EOL
'
EOF
done;
```

```bash
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name etcd$i | grep -E ' \
  +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key \
  yc-user@$vm_ip_address <<EOF
sudo bash -c 'cat >> /etc/hosts <<EOL
10.0.0.26 etcd1
10.0.0.16 etcd2
10.0.0.3 etcd3
10.0.0.27 pgsql1
10.0.0.7 pgsql2
10.0.0.13 pgsql3
10.0.0.9 haproxy
EOL
'
EOF
done;
```

```bash
vm_ip_address=$(yc compute instance show --name haproxy | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key yc-user@$vm_ip_address
```

```bash
sudo nano /etc/hosts
```

```text
10.0.0.26 etcd1
10.0.0.16 etcd2
10.0.0.3 etcd3
10.0.0.27 pgsql1
10.0.0.7 pgsql2
10.0.0.13 pgsql3
10.0.0.9 haproxy
```

- установка ETCD на 3 ВМ:

```bash
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name etcd$i | grep -E ' \+address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key \
yc-user@$vm_ip_address 'sudo apt update && sudo apt upgrade -y && sudo apt install -y etcd' & \
done;
```

```bash
vm_ip_address=$(yc compute instance show --name etcd1 | grep -E ' +address' | tail -n 1 | awk ' \
{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key yc-user@$vm_ip_address
```

```bash
etcd --version
```

```bash
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name etcd$i | grep -E ' \
+address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key \
yc-user@$vm_ip_address 'sudo systemctl stop etcd && systemctl is-enabled etcd && systemctl \
status etcd' & done;
```

```bash
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name etcd$i | grep -E ' \
+address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key \
yc-user@$vm_ip_address <<EOF
sudo bash -c 'cat >> /etc/default/etcd <<EOL
ETCD_NAME="etcd$i"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://etcd$i:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://etcd$i:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd_claster"
ETCD_INITIAL_CLUSTER="etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ENABLE_V2="true"
EOL
'
EOF
done;
```

```bash
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name etcd$i | grep -E ' \
+address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key \
yc-user@$vm_ip_address 'ETCDCRL_API=2 && echo $ETCDCRL_API' & done;
```

```bash
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name etcd$i | grep -E ' \
+address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key \
yc-user@$vm_ip_address 'sudo systemctl start etcd' & done;
```

```bash
vm_ip_address=$(yc compute instance show --name etcd1 | grep -E ' +address' | tail -n 1 | \
awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key yc-user@$vm_ip_address
```

```bash
etcdctl member list
```

```bash
etcdctl cluster-health
```

```bash
curl -L http://localhost:2379/metrics| grep fsync 
```

- Установка PostgreSQL на 3 ВМ:

```bash
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name pgsql$i | \
grep -E ' +address' | tail -n 1 | awk '{print $2}') && \
ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key yc-user@$vm_ip_address 'sudo apt update && sudo apt upgrade -y -q && echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | sudo tee -a /etc/apt/sources.list.d/pgdg.list && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-14' & done;
```

- Убедимся, что кластера Постгреса стартовали

```bash
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name pgsql$i | \
grep -E ' +address' | tail -n 1 | awk '{print $2}') && \
ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key yc-user@$vm_ip_address 'hostname; pg_lsclusters' & done;
```

- Установка Patroni на 3 ВМ(pgsql):

```bash
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name pgsql$i | \
grep -E ' +address' | tail -n 1 | awk '{print $2}') && \
ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key yc-user@$vm_ip_address 'sudo apt install -y python3 python3-pip git mc && sudo pip3 install psycopg2-binary && sudo systemctl stop postgresql@14-main && sudo -u postgres pg_dropcluster 14 main && sudo pip3 install patroni[etcd] && sudo ln -s /usr/local/bin/patroni /bin/patroni' & done;
```

```bash
vm_ip_address=$(yc compute instance show --name pgsql1 | grep -E ' +address' | tail -n 1 | \
awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key yc-user@$vm_ip_address
```

```bash
/usr/local/bin/patroni --version
```

```bash
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name pgsql$i | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key yc-user@$vm_ip_address 'cat > temp.cfg << EOF 
[Unit]
Description=Runners to orchestrate a high-availability PostgreSQL
After=syslog.target network.target
[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni.yml
KillMode=process
TimeoutSec=30
Restart=no
[Install]
WantedBy=multi-user.target
EOF
cat temp.cfg | sudo tee -a /etc/systemd/system/patroni.service
' & done;
```

- На всех нодах необходимо создать папку для хранения файлов и назначить ей нужные права, в нашем случае это /mnt/patroni.

```bash
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name pgsql$i | \
grep -E ' +address' | tail -n 1 | awk '{print $2}') && \
ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key yc-user@$vm_ip_address 'sudo systemctl daemon-reload && sudo apt install etcd-client && sudo mkdir /mnt/patroni && sudo chown postgres:postgres /mnt/patroni && sudo chmod 700 /mnt/patroni' & done;
```

```bash
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name pgsql$i | \
grep -E ' \+address' | tail -n 1 | awk '{print $2}') && \
ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key \ yc-user@$vm_ip_address 'cat > temp2.cfg << EOF 
scope: patroni
name: $(hostname)
restapi:
  listen: $(hostname -I | tr -d " "):8008
  connect_address: $(hostname -I | tr -d " "):8008
etcd:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
  initdb: 
  - encoding: UTF8
  - data-checksums
  pg_hba: 
  - host replication replicator 10.0.0.0/24 md5
  - host all all 10.0.0.0/24 md5
  users:
    admin:
      password: admin_321
      options:
        - createrole
        - createdb
postgresql:
  listen: 127.0.0.1, $(hostname -I | tr -d " "):5432
  connect_address: $(hostname -I | tr -d " "):5432
  data_dir: /var/lib/postgresql/14/main
  bin_dir: /usr/lib/postgresql/14/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: rep-pass_321
    superuser:
      username: postgres
      password: postgres
    rewind:  
      username: rewind_user
      password: rewind_password_321
  parameters:
    unix_socket_directories: '.'
tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
EOF
cat temp2.cfg | sudo tee -a /etc/patroni.yml
' & done;
```

```yaml
tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

- для настройки синхронной репликации
  
```yaml
bootstrap:
  dcs:
    synchronous_mode: "on"
    synchronous_node_count: 1
```

вот
synchronous_mode установлен в true, это означает, что Patroni будет пытаться обеспечить синхронную репликацию для определенных реплик.
В этом режиме лидер (primary) будет ждать подтверждения от одной или нескольких реплик (standby), прежде чем завершить транзакцию. Это гарантирует, что данные, записанные на лидере, будут доступны на синхронных репликах до того, как транзакция будет подтверждена клиенту.

```bash
vm_ip_address=$(yc compute instance show --name pgsql1 | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key yc-user@$vm_ip_address
```

```bash
sudo nano /etc/patroni.yml
```

>name — имя узла, на котором настраивается данный конфиг.
scope — имя кластера. Его мы будем использовать при обращении к ресурсу, а также под этим именем будет зарегистрирован сервис в etcd.
restapi-connect_address — адрес на настраиваемом сервере, на который будут приходить подключения к patroni.
restapi-auth — логин и пароль для аутентификации на интерфейсе API.
pg_hba — блок конфигурации pg_hba для разрешения подключения к СУБД и ее базам. Необходимо обратить внимание на подсеть для
строки host replication replicator. Она должна соответствовать той, которая используется в вашей инфраструктуре.
postgresql-pgpass — путь до файла, который создаст патрони. В нем будет храниться пароль для подключения к postgresql.
postgresql-connect_address — адрес и порт, которые будут использоваться для подключения к СУДБ.
postgresql - data_dir — путь до файлов с данными базы.
postgresql - bin_dir — путь до бинарников postgresql.
pg_rewind, replication, superuser — логины и пароли, которые будут созданы для базы.
user 


>Some of the PostgreSQL parameters must hold the same values on the primary and the replicas. For those, values set either in the local patroni configuration files or via the environment variables take no effect. To alter or set their values one must change the shared configuration in the DCS. Below is the actual list of such parameters together with the default values:
>>max_connections: 100
max_locks_per_transaction: 64
max_worker_processes: 8
max_prepared_transactions: 0
wal_level: hot_standby
track_commit_timestamp: off

```bash
vm_ip_address=$(yc compute instance show --name pgsql1 | \
  grep -E ' +address' | tail -n 1 | awk '{print $2}') && \
  ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key yc-user@$vm_ip_address

sudo systemctl enable patroni && sudo systemctl start patroni 

sudo patronictl -c /etc/patroni.yml list 
```

```bash
vm_ip_address=$(yc compute instance show --name pgsql2 | \
  grep -E ' +address' | tail -n 1 | awk '{print $2}') && \
  ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key yc-user@$vm_ip_address

sudo systemctl enable patroni && sudo systemctl start patroni 

sudo patronictl -c /etc/patroni.yml list 
```

```bash
vm_ip_address=$(yc compute instance show --name pgsql3 | \
  grep -E ' +address' | tail -n 1 | awk '{print $2}') && \
  ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key yc-user@$vm_ip_address

sudo systemctl enable patroni && sudo systemctl start patroni 

sudo patronictl -c /etc/patroni.yml list 
```

```bash
patronictl -c /etc/patroni.yml edit-config
```

- Рестарт одной ноды:

```bash
patronictl -c /etc/patroni.yml restart patroni pgsql2

patronictl -c /etc/patroni.yml pause patroni

patronictl -c /etc/patroni.yml resume patroni
```

- Выключение одной ноды:

```bash
sudo systemctl stop patroni 
```

- Рестарт всего кластера:

```bash
patronictl -c /etc/patroni.yml restart patroni
```

- Рестарт reload кластера:

```bash
patronictl -c /etc/patroni.yml reload patroni
```

- Плановое переключение:

```bash
patronictl -c /etc/patroni.yml switchover patroni
```

- Реинициализации ноды:

```bash
patronictl -c /etc/patroni.yml reinit patroni pgsql2
```

```bash
psql -h 127.0.0.1 -p 5432 -U postgres -W
```

- Устанавливаем HAproxy на ноду haproxy:

```bash
vm_ip_address=$(yc compute instance show --name haproxy | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/yc_key yc-user@$vm_ip_address

sudo apt-get update && sudo apt-get install -y haproxy
```

- Правим конфиг:

```bash
sudo nano /etc/haproxy/haproxy.cfg
```

```text
global
        log /dev/log local0
        log /dev/log local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
        stats timeout 30s
        user haproxy
        group haproxy
        daemon

defaults
        log global
        option tcplog
        option dontlognull
        timeout connect 5000
        timeout client 50000
        timeout server 50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http

        frontend postgres_frontend
        bind *:5432
        mode tcp
        default_backend postgres_backend

        backend postgres_backend
        mode tcp
        balance roundrobin
        option tcp-check
        default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
        server pgsql1 10.0.0.27:5432 check
        server pgsql2 10.0.0.7:5432 check
        server pgsql3 10.0.0.13:5432 check

        listen stats
        bind *:8404
        mode http
        stats enable
        stats uri /stats
        stats refresh 10s
        stats auth admin:admin_pass
```

```bash
sudo systemctl restart haproxy
sudo systemctl status haproxy
```

- Устанавливаем pgbouncer:

```bash
sudo apt-get update && sudo apt-get install -y pgbouncer
```

- Правим конфиг:

```bash
sudo nano /etc/pgbouncer/pgbouncer.ini
```

- Добавляем конфигурацию:

```conf
[databases]

= host=127.0.0.1 port=5432

[pgbouncer]
listen_addr = *
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
admin_users = postgres
pool_mode = session
max_client_conn = 100
default_pool_size = 20
```

```bash
sudo nano /etc/pgbouncer/userlist.txt
```

```text
"postgres" "md5e8a48653851e28c69d0506508fb27fc5"
```

```bash
sudo systemctl restart pgbouncer
sudo systemctl status pgbouncer.service

sudo apt update && sudo apt install postgresql-client

yc compute instance list
psql -h 158.160.44.242 -p 6432 -U postgres -W
```

rep-pass1

```bash
echo -n "admin_otus" | md5sum

"postgres" "md53c58d1f85e6e3fdd70026d16cb169d1d"

58fed5f758290943caa82f967e882211

- Удалим наш кластер Patroni:

```bash
for i in {1..3}; do yc compute instance delete pgsql$i; done && for i in {1..3}; do yc compute instance delete etcd$i; done && yc compute instance delete haproxy && yc vpc subnet delete otus-subnet && yc vpc network delete otus-net && yc dns zone delete otus-dns && unset vm_ip_address
```
