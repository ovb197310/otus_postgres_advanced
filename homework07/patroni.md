# Patroni + etcd + haproxy + bgbouncer + YC + NLB

## etc basic

- Наспройка

```
ETCD_NAME="etcd-Name-1"
ETCD_LISTEN_CLIENT_URLS="http://192.168.1.14:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://hostname1:2379"
ETCD_LISTEN_PEER_URLS="http://192.168.1.14:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://hostname1:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd_Name_Claster"
ETCD_INITIAL_CLUSTER="etcd-Name-1=http://hostname1:2380, etcd-Name-2=http://hostname2:2380, etcd-Name-3 =
http://hostname3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd
```