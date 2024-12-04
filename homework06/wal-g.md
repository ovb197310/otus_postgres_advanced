# Использование wal-g

адаптировано для Linux Mint 22

работы проводятся для кластерах:

Ver|Cluster|Port|Status|Owner|Data directory|Log file
---|---|---|---|---|---|---
16| main|5432|online|postgres|/var/lib/postgresql/16/main|/var/log/postgresql/postgresql-16-main.log
16|test|5433|online|postgres|/var/lib/postgresql/16/test|/var/log/postgresql/postgresql-16-test.log

файлы конфигурации в каталогах

 `/etc/postgresql/16/main`
 `/etc/postgresql/16/test`

## Инсталяция

устанавливаем в соответсвии с [документацией](https://github.com/wal-g/wal-g?tab=readme-ov-file)

### скачивание

```bash
mkdir -p /tmp/wal-g
cd /tmp/wal-g
wget https://github.com/wal-g/wal-g/releases/download/v3.0.3/wal-g-pg-ubuntu-20.04-amd64
wget https://github.com/wal-g/wal-g/releases/download/v3.0.3/wal-g-pg-ubuntu-20.04-amd64.sha256
sha256sum -c wal-g-pg-ubuntu-20.04-amd64.sha256
sudo cp wal-g-pg-ubuntu-20.04-amd64 /usr/local/bin/wal-g
```

### добавляем bash completion

```bash
wal-g completion bash >> ~/.bashrc
source ~/.bashrc
```

### создаем storage

```bash
sudo mkdir /var/wal-g-postgres
sudo chown postgres: /var/wal-g-postgres
```

### минимальный конфигурационный файл

создаем файл `/var/lib/postgresql/.walg.json`

```json
{
    "WALG_FILE_PREFIX": "/var/wal-g-postgres",
    "WALG_COMPRESSION_METHOD": "brotli",
    "WALG_DELTA_MAX_STEPS": "5",
    "PGDATA": "/var/lib/postgresql/16/main",
    "PGHOST": "/run/postgresql/.s.PGSQL.5432"
}
```

### тюнинг postgres.conf для ***main***

создаем файл `/etc/postgresql/16/main/conf.d/wal-g.conf` с контентом

```conf
wal_level=replica
archive_mode=on
archive_command='/usr/local/bin/wal-g wal-push \"%p\" >> /var/log/postgresql/archive.log 2>&1'
archive_timeout=60
restore_command='/usr/local/bin/wal-g wal-fetch \"%f\" \"%p\" >> /var/log/postgresql/restore.log 2>&1'
```

### применение конфигураций

```bash
sudo systemctl postgresql@16-main restart
```

проверяем применение параметров

```sql
select name, sourcefile, setting 
    from pg_settings 
    where name in 
        (
        'wal_level',
        'archive_mode',
        'archive_command',
        'archive_timeout', 
        'restore_command'
        );
```

name|sourcefile|setting                                      
-----------------|-------------------------------------------|----------------------------------------------------------------------------------
 archive_command | /etc/postgresql/16/main/conf.d/wal-g.conf | /usr/local/bin/wal-g wal-push "%p" >> /var/log/postgresql/archive.log 2>&1
 archive_mode    | /etc/postgresql/16/main/conf.d/wal-g.conf | on
 archive_timeout | /etc/postgresql/16/main/conf.d/wal-g.conf | 60
 restore_command | /etc/postgresql/16/main/conf.d/wal-g.conf | /usr/local/bin/wal-g wal-fetch "%f" "%p" >> /var/log/postgresql/restore.log 2>&1
 wal_level       | /etc/postgresql/16/main/conf.d/wal-g.conf | replica

и в логах `/var/log/postgresql/archive.log`

```log
INFO: 2024/12/04 21:41:46.229310 Files will be uploaded to storage: default
INFO: 2024/12/04 21:41:46.279196 FILE PATH: 000000010000000000000005.br
```

и в storage `tree /var/wal-g-postgres/`

```
/var/wal-g-postgres/
└── wal_005
    └── 000000010000000000000005.br

```

### "зальем" данные

```sql
DROP DATABASE IF EXISTS otus;
CREATE DATABASE otus;
\c otus
CREATE TABLE test(i int);
insert into test(i) select * from generate_series(1, 100);
```

### full backup

```bash
wal-g backup-push /var/lib/postgresql/16/main/
wal-g backup-list
```

результат

backup_name|modified|wal_file_name|storage_name
---|---|---|---
base_00000001000000000000000A| 2024-12-04T21:51:53+03:00| 00000001000000000000000A| default



### удаляем каталог с данными в кластере **test**, восстанавливаем из полной копии кластера **main**

```bash
systemctl stop postgresql@16-test
rm -rf /var/lib/postgresql/16/test
wal-g backup-fetch /var/lib/postgresql/16/test LATEST
```

результат

```log
INFO: 2024/12/04 22:02:47.018332 Selecting the latest backup...
INFO: 2024/12/04 22:02:47.018445 Backup to fetch will be searched in storages: [default]
INFO: 2024/12/04 22:02:47.018626 LATEST backup is: 'base_00000001000000000000000C_D_00000001000000000000000A'
INFO: 2024/12/04 22:02:47.026113 Delta from base_00000001000000000000000A at LSN 0/A000028 
INFO: 2024/12/04 22:02:47.038978 Finished extraction of part_003.tar.br
INFO: 2024/12/04 22:02:51.324949 Finished extraction of part_001.tar.br
INFO: 2024/12/04 22:02:51.326900 Finished extraction of pg_control.tar.br
INFO: 2024/12/04 22:02:51.326966 
Backup extraction complete.
INFO: 2024/12/04 22:02:51.326998 base_00000001000000000000000A fetched. Upgrading from LSN 0/A000028 to LSN 0/C000028 
INFO: 2024/12/04 22:02:51.356315 Finished extraction of part_003.tar.br
INFO: 2024/12/04 22:02:51.358588 Finished extraction of part_001.tar.br
INFO: 2024/12/04 22:02:51.362071 Finished extraction of pg_control.tar.br
INFO: 2024/12/04 22:02:51.362109 
Backup extraction complete.
```

### проверка

переконфигурируем тест перед запуском, файл `/etc/postgresql/16/test/conf.d/wal-g.conf`

```ini
wal_level=replica
archive_mode=on
archive_command='/usr/local/bin/wal-g wal-push \"%p\" >> /var/log/postgresql/archive-test.log 2>&1'
archive_timeout=60
restore_command='/usr/local/bin/wal-g wal-fetch \"%f\" \"%p\" >> /var/log/postgresql/restore-test.log 2>&1'
```

и обновим файл
```bash
touch /var/lib/postgresql/16/test/recovery.signal
```

```bash
systemctl start postgresql@16-test
```

проверим наличие данных

```bash
psql -p 5433 -c 'select count(*) from test;' otus
```

```
 count 
-------
   100
(1 row)
```