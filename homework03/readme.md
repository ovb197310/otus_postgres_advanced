# Использование Managed Service for PostgreSQL

## Создаем кластер PostgreSQL с использованием Managed Service for PostgreSQL в Yandex.Cloud
- Создаем кластер в WEB-консоль

![создание01](create01.jpg)

![создание01](create02.jpg)

![создание01](create03.jpg)

## Ограничим доступ (входящий трафик) через Security Group

![SG](sg.jpg)

## проверим статус кластера в консоли yc

![yc cluster](yc-cluster.jpg "Кластера postgres") 

- хосты в кластере

![yc hosts 01](yc-cluster-hosts01.png "Кластера postgres - хосты") 

- проверка разрешения FQDN кластера в IP

![yc hosts ns 01](yc-ns01.png) 

![yc hosts ns 02](yc-ns02.png) 


## Подключение к БД

- скачиваем сертификат

```
mkdir -p ~/.postgresql && \
wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" \
    --output-document ~/.postgresql/root.crt && \
chmod 0600 ~/.postgresql/root.crt
```

- подключаемся

```
psql "host=rc1d-7cjifbvs60b0wtv1.mdb.yandexcloud.net \
    port=6432 \
    sslmode=verify-full \
    dbname=db1 \
    user=user1 \
    target_session_attrs=read-write"
```

![yc hosts con 01](connect01.png) 

- подключаемся по имени FQDN кластера к мастеру

```
psql "host=c-c9qf7sovjgc58qvvh7pi.rw.mdb.yandexcloud.net \
    port=6432 \
    sslmode=verify-full \
    dbname=db1 \
    user=user1 \
    target_session_attrs=read-write"
```

![yc hosts con 02](connetc02.png) 


## Расширение кластера

- добавляем хост в кластер (без публичного доступа), ждем завершения 9 минут

![yc hosts 02](create_second.png)

![yc hosts 02](create_second_log.png)

- даем публичный доступ ко второму хосту, проверяем FQDN мастера и наименее отстающей реплики

![yc hosts 02 pub](add_public.png)

- топология

![topology](topology.png)

## Смена ролей в кластере

- переключаем мастера

![switch](switch.png)

![switch log](switch_log.png)

- проверяем FQDN мастера и наименее отстающей реплики

![switch fqdn](switch_fqdn.png)

## Проверяем кластер на доступ

- подключаемся по имени FQDN кластера к мастеру

```
psql "host=c-c9qf7sovjgc58qvvh7pi.rw.mdb.yandexcloud.net \
    port=6432 \
    sslmode=verify-full \
    dbname=db1 \
    user=user1 \
    target_session_attrs=read-write"
```

![create_insert_master.png](create_insert_master.png)

- подключаемся по имени FQDN кластера к реплике - можно только в режиме **read-only**

```
psql "host=c-c9qf7sovjgc58qvvh7pi.ro.mdb.yandexcloud.net \
    port=6432 \
    sslmode=verify-full \
    dbname=db1 \
    user=user1 \
    target_session_attrs=read-only"
```

![readonly](readonly-replica.png)



## Удаляем кластер

работа завершена
