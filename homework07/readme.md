# Настройка кластера ETCD/Patrony/PostgreSQL/HAProxy/keepalived

используем 3 ВМ для кластера ETCD, 2 ВМ для кластера PostgreSQL,

role|ip|FQDN|HostName
---|---|---|---
etcd node|172.16.104.70|z14-3517-etcd.vesta.ru|z14-3517-etcd
etcd node|172.16.104.71|z14-3518-etcd.vesta.ru|z14-3518-etcd
etcd node|172.16.104.81|z14-3519-etcd.vesta.ru|z14-3519-etcd
PostgreSQL node|172.16.104.109|z14-3520-pgsql.vesta.ru|z14-3520-pgsql
PostgreSQL node|172.16.104.112|z14-3521-pgsql.vesta.ru|z14-3521-pgsql
VIP (keepalived)|172.16.104.100|adsite02.alfastrah.ru

## вносим ноды в /etc/hosts для отвязки от DNS (для исключения DNS запросов)

```conf
172.16.104.70 z14-3517-etcd.vesta.ru z14-3517-etcd
172.16.104.71 z14-3518-etcd.vesta.ru z14-3518-etcd
172.16.104.81 z14-3519-etcd.vesta.ru z14-3519-etcd
172.16.104.109 z14-3520-pgsql.vesta.ru z14-3520-pgsql
172.16.104.112 z14-3521-pgsql.vesta.ru z14-3521-pgsql
172.16.104.100 adsite02.alfastrah.ru   adsite02
```

## ставим и настраиваем etcd

```bash
apt install -y etcd
```

### создаем каталоги

```bash
mkdir -pv /etc/etcd && chown etcd: /etc/etcd && chmod 755 /etc/etcd
mkdir -pv /var/lib/etcd/adsite02 && chown etcd: /var/lib/etcd/adsite02 && chmod 700 /var/lib/etcd/adsite02
```

### ложим конфиг

пример конфигурации для __z14-3517-etcd__

```conf
ETCD_NAME="z14-3517-etcd"
ETCD_LISTEN_CLIENT_URLS="http://172.16.104.70:2379,http://127.0.0.1:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://172.16.104.70:2379"
ETCD_LISTEN_PEER_URLS="http://172.16.104.70:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://172.16.104.70:2380"
ETCD_INITIAL_CLUSTER_TOKEN="adsite02"
ETCD_INITIAL_CLUSTER="z14-3517-etcd=http://z14-3517-etcd:2380,z14-3518-etcd=http://z14-3518-etcd:2380,z14-3519-etcd=http://z14-3519-etcd:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
#ETCD_INITIAL_CLUSTER_STATE="existing"
ETCD_DATA_DIR="/var/lib/etcd/adsite02"
#ETCD_ENABLE_V2="true"
ETCD_ELECTION_TIMEOUT="5000"
ETCD_HEARTBEAT_INTERVAL="1000"
```

```bash
scp -i \.ssh\230430\BalitskiiOV-230430.ssh z14-3517-etcd/etcd.conf balitskiiov@z14-3517-etcd:/tmp

scp -i \.ssh\230430\BalitskiiOV-230430.ssh z14-3518-etcd/etcd.conf balitskiiov@z14-3518-etcd:/tmp

scp -i \.ssh\230430\BalitskiiOV-230430.ssh z14-3519-etcd/etcd.conf balitskiiov@z14-3519-etcd:/tmp

scp -i \.ssh\230430\BalitskiiOV-230430.ssh etcd.service balitskiiov@z14-3517-etcd:/tmp

scp -i \.ssh\230430\BalitskiiOV-230430.ssh etcd.service balitskiiov@z14-3518-etcd:/tmp

scp -i \.ssh\230430\BalitskiiOV-230430.ssh etcd.service balitskiiov@z14-3519-etcd:/tmp
```

### обновляем на серверах

```bash
sudo -i
mv -v /tmp/etcd.conf /etc/etcd && chmod 644 /etc/etcd/etcd.conf && chown etcd: /etc/etcd/etcd.conf
mv -v /tmp/etcd.service /lib/systemd/system/etcd.service
systemctl daemon-reload
```

### Запускаем etcd

```bash
systemctl daemon-reload
systemctl restart etcd
```

### переводим ETCD в режим `existing`

```bash

systemctl restart etcd
```

### результат

```bash
etcdctl member list
```

```text
4846706eeb5e2685: name=z14-3517-etcd peerURLs=http://z14-3517-etcd:2380 clientURLs=http://172.16.104.70:2379 isLeader=true
521b31849a8cac18: name=z14-3518-etcd peerURLs=http://z14-3518-etcd:2380 clientURLs=http://172.16.104.71:2379 isLeader=false
834dd20ce4f5948e: name=z14-3519-etcd peerURLs=http://z14-3519-etcd:2380 clientURLs=http://172.16.104.81:2379 isLeader=false
```

## ставим и настраиваем PostgreSQL 16

```bash
apt install postgresql-16
pg_dropcluster --stop 16 main
```

## сгенерим локаль

```bash
locale-gen ru_RU.UTF-8
```

## создадим кластер

```bash
pg_createcluster -d /data/adsite02 -p 5432 16 adsite02 --encoding=UTF-8 -- --data-checksums --locale=ru_RU.utf8
```

## настраиваем репликаицю

### на мастере

- создадим пользователя `replication`

```sql
create user replication with replication password 'replication123';
```

- разрешаем подключаться с реплики + на будующее под HAProxy

файл `/etc/postgresql/16/adsite02/pg_hba.conf`

```text
host    replication     replication             172.16.104.109/24               scram-sha-256
host    replication     replication             172.16.104.112/24               scram-sha-256
host    all     postgres             172.16.104.112/24               trust
host    all     postgres             172.16.104.109/24               trust
host    all     all             172.16.104.112/24               scram-sha-256
host    all     all             172.16.104.109/24               scram-sha-256
```

- меняем параметры в `/etc/postgresql/16/adsite02/conf.d/00-replication.conf`

```conf
listen_addresses='72.16.104.109'
wal_level=logical
wal_log_hints=on
```

- перезапускаем кластер

```bash
pg_ctlclustre 16 adsite02 restart
```

### на реплике

- останавливаем и удаляем каталог данных

```bash
pg_ctlclustre 16 adsite02 stop
rm -rf /data/adsite02
```

- разрешаем подключаться с реплики + на будующее под HAProxy

файл `/etc/postgresql/16/adsite02/pg_hba.conf`

```text
host    replication     replication             172.16.104.109/24               scram-sha-256
host    replication     replication             172.16.104.112/24               scram-sha-256
host    all     postgres             172.16.104.112/24               trust
host    all     postgres             172.16.104.109/24               trust
host    all     all             172.16.104.112/24               scram-sha-256
host    all     all             172.16.104.109/24               scram-sha-256
```

- переносим даные с мастера и запускаем

```bash
pg_basebackup -h z14-3520-pgsql -U replication -X stream -C -S replica_1 -v -R -W -D /data/adsite02
chown -R postgres: /data/adsite02
pg_ctlclustre 16 adsite02 start
```

## ставим Patroni

```bash
apt install patroni
```

- генерируем базовый конфиг

```bash
pg_createconfig_patroni 16 adsite02
```

- корректируем и получаем

<details>
<summary>/etc/patrony/config.yml</summary>

```yaml
# for another node of cluster change 172.16.104.109 to 172.16.104.112
#name: z14-3521-pgsql

scope: "adsite02"
namespace: "/postgresql-common/"
name: z14-3520-pgsql

etcd:
  hosts: z14-3517-etcd:2379,z14-3518-etcd:2379,z14-3519-etcd:2379

restapi:
  listen: 172.16.104.109:8008
  connect_address: 172.16.104.109:8008
#  certfile: /etc/ssl/certs/ssl-cert-snakeoil.pem
#  keyfile: /etc/ssl/private/ssl-cert-snakeoil.key
  authentication:
    username: username
    password: password

# ctl:
#   insecure: false # Allow connections to SSL sites without certs
#   certfile: /etc/ssl/certs/ssl-cert-snakeoil.pem
#   cacert: /etc/ssl/certs/ssl-cacert-snakeoil.pem

bootstrap:
  # Custom bootstrap method
  # The options --scope= and --datadir= are passed to the custom script by
  # patroni and passed on to pg_createcluster by pg_createcluster_patroni
  method: pg_createcluster
  pg_createcluster:
    command: /usr/share/patroni/pg_createcluster_patroni

  # This section will be written into /<namespace>/<scope>/config after
  # initializing a new cluster and all other cluster members will use it as a
  # `global configuration`
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    check_timeline: true
#    master_start_timeout: 300
#    synchronous_mode: false
#    standby_cluster:
#      host: 127.0.0.1
#      port: 1111
#      primary_slot_name: patroni
    postgresql:
      use_pg_rewind: true
      remove_data_directory_on_rewind_failure: true
      remove_data_directory_on_diverged_timelines: true
      use_slots: true
      # The following parameters are given as command line options
      # overriding the settings in postgresql.conf.
      parameters:
#        wal_level: hot_standby
#        hot_standby: "on"
#        wal_keep_segments: 8
#        max_wal_senders: 10
#        max_replication_slots: 10
#        max_worker_processes = 8
#        wal_log_hints: "on"
#        track_commit_timestamp = "off"
#      recovery_conf:
#        restore_command: cp ../wal_archive/%f %p
      # Set pg_hba.conf to the following values after bootstrapping or cloning.
      # If you want to allow regular connections from the local network, or
      # want to use pg_rewind, you need to uncomment the fourth entry.
      pg_hba:
      - local   all             all                                     peer
      - host    all             all             127.0.0.1/32            md5
      - host    all             all             ::1/128                 md5
      - local   replication     all                                     peer
      - host    replication     all             127.0.0.1/32            md5
      - host    replication     all             ::1/128                 md5
      - host    replication     replication             172.16.104.109/24               md5
      - host    replication     replication             172.16.104.112/24               md5
      - host    all     postgres             172.16.104.112/24               trust
      - host    all     all             172.16.104.112/24               md5
#  # Some possibly desired options for 'initdb'. Note: It needs to be a list
#  # (some options need values, others are # switches)
      initdb:
      - encoding: UTF8
      - data-checksums
      - locale: ru_RU.utf8

#  # Additional script to be launched after initial cluster creation (will be
#  # passed the connection URL as parameter)
#  post_init: /usr/local/bin/setup_cluster.sh

#  # Additional users to be created after initializing the cluster
#  users:
#    foo:
#      password: bar
#      options:
#        - createrole
#        - createdb

postgresql:
  # Custom clone method
  # The options --scope= and --datadir= are passed to the custom script by
  # patroni and passed on to pg_createcluster by pg_clonecluster_patroni
  create_replica_method:
    - pg_clonecluster
  pg_clonecluster:
    command: /usr/share/patroni/pg_clonecluster_patroni

  # Listen to all interfaces by default, this makes vip-manager work
  # out-of-the-box without having to set net.ipv4.ip_nonlocal_bind or similar.
  # If you prefer to only listen on some interfaces, edit the below:
#  listen: "172.16.104.109,127.0.0.1:5432"
  listen: "*:5432"
  connect_address: 172.16.104.109:5432
  use_unix_socket: true
  # Default Debian/Ubuntu directory layout
  data_dir: /data/adsite02
  bin_dir: /usr/lib/postgresql/16/bin
  config_dir: /etc/postgresql/16/adsite02
  pgpass: /data/adsite02/16-adsite02.pgpass
  authentication:
    replication:
      username: "replication"
      password: "replication123"
    # A superuser role is required in order for Patroni to manage the local
    # Postgres instance.  If the option `use_unix_socket' is set to `true',
    # then specifying an empty password results in no md5 password for the
    # superuser being set and sockets being used for authentication. The
    # `password:' line is nevertheless required.  Note that pg_rewind will not
    # work if no md5 password is set unless a rewind user is configured, see
    # below.
    superuser:
      username: "postgres"
      password:
    # A rewind role can be specified in order for Patroni to use on PostgreSQL
    # 11 or later for pg_rewind, i.e. rewinding a former primary after failover
    # without having to re-clone it. Patroni will assign this user the
    # necessary permissions (that only exist from PostgreSQL)
#    rewind:
#      username: "rewind"
#      password: "rewind-pass"

  parameters:
    unix_socket_directories: '/var/run/postgresql/'
    # Emulate default Debian/Ubuntu logging
    logging_collector: 'on'
    log_directory: '/var/log/postgresql'
    log_filename: 'postgresql-16-adsite02.log'
```

</details>

- запрещаем автоматический запуск PostgreSQL

правим файл `/etc/postgresql/16/adsite02/start.conf`, должно быть manual

```conf
# Automatic startup configuration
#   auto: automatically start the cluster
#   manual: manual startup with pg_ctlcluster/postgresql@.service only
#   disabled: refuse to start cluster
# See pg_createcluster(1) for details. When running from systemd,
# invoke 'systemctl daemon-reload' after editing this file.

manual
```

```bash
systemctl daemon-reload
```

- запускаем Patroni

```bash
systemctl start patroni
```

- проверяем статус (после нескольких рестартов нод)

```bash
patronictl -c /etc/patroni/config.yml list
```

 Member         | Host           | Role    | State     | TL | Lag in MB
----------------|----------------|---------|-----------|----|-----------
 z14-3520-pgsql | 172.16.104.109 | Leader  | running   | 12 |
 z14-3521-pgsql | 172.16.104.112 | Replica | streaming | 12 |         0

- делаем switchover

```bash
patronictl -c /etc/patroni/config.yml switchover
```

Current cluster topology

Cluster: adsite02 (7447406115886761416)

 Member         | Host           | Role    | State     | TL | Lag in MB
----------------|----------------|---------|-----------|----|-----------
 z14-3520-pgsql | 172.16.104.109 | Leader  | running   | 12 |
 z14-3521-pgsql | 172.16.104.112 | Replica | streaming | 12 |         0

Primary [z14-3520-pgsql]:

Candidate ['z14-3521-pgsql'] []:

When should the switchover take place (e.g. 2024-12-12T18:14 )  [now]:

Are you sure you want to switchover cluster adsite02, demoting current leader z14-3520-pgsql? [y/N]: y

2024-12-12 17:14:50.49323 Successfully switched over to "z14-3521-pgsql"

Cluster: adsite02 (7447406115886761416)

 Member         | Host           | Role    | State   | TL | Lag in MB
----------------|----------------|---------|---------|----|-----------
 z14-3520-pgsql | 172.16.104.109 | Replica | stopped |    |   unknown
 z14-3521-pgsql | 172.16.104.112 | Leader  | running | 12 |

- перепроверяем статус кластера

```bash
patronictl -c /etc/patroni/config.yml switchover
```

 Cluster: adsite02 (7447406115886761416)

 Member         | Host           | Role    | State     | TL | Lag in MB
----------------|----------------|---------|-----------|----|-----------
 z14-3520-pgsql | 172.16.104.109 | Replica | streaming | 13 |         0
 z14-3521-pgsql | 172.16.104.112 | Leader  | running   | 13 |

## Установка и настройка keepalived

```bash
apt install keepalived
```

конфигурация `/etc/keepalived/keepalived.conf`

```conf
global_defs {
   router_id adsite02
   enable_script_security
}


vrrp_script myhealth {
    script "/bin/nc -z -w 2 127.0.0.1 5432"
    interval 10
    user nobody
}

vrrp_instance VI_1 {
    state MASTER
    interface ens160  # сетевой интерфейс
    virtual_router_id 51
    priority 101  # приоритет выше, чем у резервного сервера
    advert_int 1
#    nopreempt
    authentication {
        auth_type PASS
        auth_pass 1q2w3E4R  # пароль
    }

    virtual_ipaddress {
        172.16.104.100 dev ens160 label ens160:vIP
    }

    track_script {
        myhealth
    }
}
```

## Установка HAProxy

на серверах БД ставим haproxy

```bash
apt install haproxy
```

производим настройку по портам

role|ip|FQDN|HostName|port Leader|port Replica
---|---|---|---|---|---
PostgreSQL node|172.16.104.109|z14-3520-pgsql.vesta.ru|z14-3520-pgsql|6432|7432
PostgreSQL node|172.16.104.112|z14-3521-pgsql.vesta.ru|z14-3521-pgsql|6432|7432

конфигурация `/etc/haproxy/haproxy.cfg`

```conf
global
  maxconn 3000
  log 127.0.0.1 local2

defaults
  log global
  mode tcp
  retries 2
  timeout client 30m
  timeout connect 4s
  timeout server 30m
  timeout check 5s

listen stats
  mode http
  bind *:7000
  stats enable
  stats uri /

listen adsite02_6432
  bind *:6432
  option httpchk GET /primary
  http-check expect status 200
  default-server inter 15s fall 3 rise 2 on-marked-down shutdown-sessions
  server z14-3520-pgsql:5432 z14-3520-pgsql:5432 maxconn 1000 check port 8008
  server z14-3521-pgsql:5432 z14-3521-pgsql:5432 maxconn 1000 check port 8008

listen adsite02_7432
  bind *:7432
  option httpchk GET /replica
  http-check expect status 200
  default-server inter 15s fall 3 rise 2 on-marked-down shutdown-sessions
  server z14-3520-pgsql:5432 z14-3520-pgsql:5432 maxconn 1000 check port 8008
  server z14-3521-pgsql:5432 z14-3521-pgsql:5432 maxconn 1000 check port 8008
```

## Проверка

- подключаемся к adsite02.alfastrah.ru

```bash
psql -h adsite02.alfastrah.ru -p 6432 -U postgres -W postgres
```

- создаем БД **otus_test** и таблицу с данными

  `create database otus_test;`

  `\c otus_test`

```
  create table persons(id serial, first_name text, second_name text);
  insert into persons(first_name, second_name) values('ivan', 'ivanov');
  insert into persons(first_name, second_name) values('petr', 'petrov');
  select * from persons;
```

- выключаем сервер z14-3520-pgsql

```bash
halt
```

- подключаемся к adsite02.alfastrah.ru

```bash
psql -h adsite02.alfastrah.ru -p 6432 -U postgres -W otus_test
```

```sql
select * from persons;
```

Результат: __Данные на месте__

- включаем сервер z14-3520-pgsql и делаем switchover

```bash
patronictl -c /etc/patroni/config.yml switchover
```

Member         | Host           | Role    | State     | TL | Lag in MB
----------------|----------------|---------|-----------|----|-----------
 z14-3520-pgsql | 172.16.104.109 | Leader | running | 32 |         
 z14-3521-pgsql | 172.16.104.112 | Replica  | streaming   | 32 |0
