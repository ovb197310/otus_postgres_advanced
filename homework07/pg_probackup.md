# PG_PROBACKUP

- install (as root)

```bash
cd /tmp
wget https://repo.postgrespro.ru/pg_probackup/deb/pool/main-jammy/p/pg-probackup-16/pg-probackup-16_2.5.15-1.ac92457c2d1cfe43fced5b1167b5c90ecdc24cbe.jammy_amd64.deb
dpkg -i pg-probackup-16_2.5.15-1.ac92457c2d1cfe43fced5b1167b5c90ecdc24cbe.jammy_amd64.deb
ln -s /usr/bin/pg_probackup-16 /usr/bin/pg_probackup
```

- prepare (as root)

```bash
mkdir -p /var/pg_probackup
chown postgres: /var/pg_probackup
chmod 700 /var/pg_probackup
```

- as postgres

```bash
#init directory
pg_probackup init -B /var/pg_probackup
#add local
pg_probackup add-instance -B /var/pg_probackup -D /data/adsite02 --instance=adsite02
```

result

```
postgres@z14-3520-pgsql:/var/pg_probackup$ tree .
.
├── backups
│   └── adsite02
│       └── pg_probackup.conf
└── wal
    └── adsite02
```

- Configuring the Database Cluster

```sql
CREATE DATABASE backupdb;

#\c backupdb

BEGIN;
CREATE ROLE backup WITH LOGIN;
GRANT USAGE ON SCHEMA pg_catalog TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.current_setting(text) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.set_config(text, text, boolean) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_is_in_recovery() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_start(text, boolean) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_stop(boolean) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_create_restore_point(text) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_switch_wal() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_last_wal_replay_lsn() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_current() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_current_snapshot() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_snapshot_xmax(txid_snapshot) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_control_checkpoint() TO backup;
COMMIT;
```

- change pg_hba.conf 

```conf
host    replication     backup             172.16.104.112/24               trust
host    replication     backup             172.16.104.109/24               trust
host    backupdb        backup             172.16.104.112/24               trust
host    backupdb        backup             172.16.104.109/24               trust
```

- config SSH trusted (skip detail)


- full backup

```bash
pg_probackup backup -B /var/pg_probackup/ -b FULL --instance=adsite02 --stream  \
  --remote-host=z14-3521-pgsql  --remote-user=postgres -U backup -d backupdb
```

```log
INFO: Backup start, pg_probackup version: 2.5.15, instance: adsite02, backup ID: SONGRT, backup mode: FULL, wal mode: STREAM, remote: true, compress-algorithm: none, compress-level: 1
INFO: This PostgreSQL instance was initialized with data block checksums. Data block corruption will be detected
INFO: Database backup start
INFO: wait for pg_backup_start()
INFO: Wait for WAL segment /var/pg_probackup/backups/adsite02/SONGRT/database/pg_wal/000000160000000000000017 to be streamed
INFO: PGDATA size: 37MB
INFO: Current Start LSN: 0/17000028, TLI: 16
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 3s
INFO: wait for pg_stop_backup()
INFO: pg_stop backup() successfully executed
INFO: stop_lsn: 0/170001A0
INFO: Getting the Recovery Time from WAL
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 3s
INFO: Validating backup SONGRT
INFO: Backup SONGRT data files are valid
INFO: Backup SONGRT resident size: 53MB
INFO: Backup SONGRT completed
```

- backup info

```bash
pg_probackup show -B /var/pg_probackup/
```

Instance|  Version|  ID|      Recovery Time|           Mode|  WAL Mode|  TLI|   Time|  Data|   WAL|  Zratio|  Start LSN|   Stop LSN|    Status
---|---|---|---|---|---|---|---|---|---|---|---|---|---
adsite02|  16|       SONH78|  2024-12-17 21:11:35+03|  FULL|  STREAM|    22/0|   12s|  37MB|  32MB|    1.00|  0/19000028|  0/19000168|  OK
adsite02|  16|       SONGRT|  2024-12-17 21:02:22+03|  FULL|  STREAM|    22/0|   14s|  37MB|  16MB|    1.00|  0/17000028|  0/170001A0|  OK

-- add external dirs

```bash
pg_probackup backup -B /var/pg_probackup/ -b FULL --instance=adsite02 --stream  \
  --remote-host=z14-3521-pgsql  --remote-user=postgres -U backup -d backupdb \
  --external-dirs=/etc/postgresql
```

Instance|  Version|  ID|      Recovery Time|           Mode|  WAL Mode|  TLI|   Time|  Data|   WAL|  Zratio|  Start LSN|   Stop LSN|    Status
---|---|---|---|---|---|---|---|---|---|---|---|---|---
adsite02|  16|       SONHC9|  2024-12-17 21:14:36+03|  FULL|  STREAM|    22/0|   15s|  37MB|  32MB|    1.00|  0/1C000028|  0/1C000168|  OK

```bash
cd /var/pg_probackup/backups/adsite02/SONHC9
tree -d .
```

```txt
├── database
│   ├── base
│   │   ├── 1
│   │   ├── 16389
│   │   ├── 16407
│   │   ├── 4
│   │   ├── 5
│   │   └── pgsql_tmp
│   ├── global
│   ├── pg_commit_ts
│   ├── pg_dynshmem
│   ├── pg_logical
│   │   ├── mappings
│   │   └── snapshots
│   ├── pg_multixact
│   │   ├── members
│   │   └── offsets
│   ├── pg_notify
│   ├── pg_replslot
│   ├── pg_serial
│   ├── pg_snapshots
│   ├── pg_stat
│   ├── pg_stat_tmp
│   ├── pg_subtrans
│   ├── pg_tblspc
│   ├── pg_twophase
│   ├── pg_wal
│   └── pg_xact
└── external_directories
    └── externaldir1
        └── 16
            └── adsite02
                └── conf.d
```
