Кластер Patroni
-------------------------------

## Развертывание

### Установка и настройка кластера etcd

> Res: https://sysad.su/%D1%83%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0-%D0%B8-%D0%BD%D0%B0%D1%81%D1%82%D1%80%D0%BE%D0%B9%D0%BA%D0%B0-%D0%BA%D0%BB%D0%B0%D1%81%D1%82%D0%B5%D1%80%D0%B0-etcd-ubuntu-18/
> Alternative: https://www.techsupportpk.com/2020/02/how-to-set-up-highly-available-postgresql-cluster-ubuntu-19-20.html

#### Разворачиваем кластер etcd patroni на трёх узлах: 
- etcd01
- etcd02
- etcd03

На всех хостах производим ***обновление системы***
```
otus@etcd01:~$ sudo apt-get update # apt-get upgrade
```

#### Устанавливаем etcd на первом узле
```
otus@etcd01:~$ sudo apt-get install etcd
```

Конфигурируем ***обнаружение*** при помощи ***статической конфигурации***.
Инициализацию кластера начинаем с ведущего etcd01.

Меняем содержимое файла /etc/default/etcd на следующее:
```
ETCD_NAME="etcd01"
ETCD_DATA_DIR="/var/lib/etcd/pg-cluster"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://etcd01.ru-central1.internal:2380"
ETCD_INITIAL_CLUSTER="etcd01=http://etcd01.ru-central1.internal:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="pg-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://etcd01.ru-central1.internal:2379"
```
```
otus@etcd01:~$ cd /etc/default/
otus@etcd01:/etc/default$ ls -la
total 112
drwxr-xr-x  3 root root  4096 May 23 17:23 .
drwxr-xr-x 97 root root  4096 May 23 17:23 ..
...
-rw-r--r--  1 root root 12632 Feb 25  2022 etcd
...
```

Редактируем
```
otus@etcd01:/etc/default$ sudo nano /etc/default/etcd
```

Запускаем
```
otus@etcd01:/etc/default$ sudo systemctl restart etcd
```

Проверяем
```
otus@etcd01:/etc/default$ systemctl status etcd.service
● etcd.service - etcd - highly-available key value store
     Loaded: loaded (/lib/systemd/system/etcd.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2023-05-23 17:54:41 UTC; 6s ago
       Docs: https://etcd.io/docs
             man:etcd
   Main PID: 2375 (etcd)
      Tasks: 7 (limit: 2232)
     Memory: 4.3M
        CPU: 42ms
     CGroup: /system.slice/etcd.service
             └─2375 /usr/bin/etcd
```
```
otus@etcd01:/etc/default$ sudo etcdctl cluster-health
member 685e284b1739bd16 is healthy: got healthy result from http://etcd01.ru-central1.internal:2379
cluster is healthy
```

##### Добавление второго узла

На ***etcd01*** выполняем оповещение кластера о появлении нового узла ***etcd02***:
```
otus@etcd01:/etc/default$ sudo etcdctl member add etcd02 http://etcd02.ru-central1.internal:2380
Added member named etcd02 with ID 3c2e492c60bebded to cluster

ETCD_NAME="etcd02"
ETCD_INITIAL_CLUSTER="etcd02=http://etcd02.ru-central1.internal:2380,etcd01=http://etcd01.ru-central1.internal:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
```

На узле ***etcd02*** содержимое файла конфигурации /etc/default/etcd заменяем на следующее:
```
ETCD_NAME="etcd02"
ETCD_DATA_DIR="/var/lib/etcd/pg-cluster"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://etcd02.ru-central1.internal:2380"
ETCD_INITIAL_CLUSTER="etcd02=http://etcd02.ru-central1.internal:2380,etcd01=http://etcd01.ru-central1.internal:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
ETCD_INITIAL_CLUSTER_TOKEN="pg-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://etcd02.ru-central1.internal:2379"
```

И перезапускаем etcd:
```
otus@etcd02:~$ sudo systemctl restart etcd
```

Проверяем
```
otus@etcd02:~$ sudo etcdctl cluster-health
member 3c2e492c60bebded is healthy: got healthy result from http://etcd02.ru-central1.internal:2379
member 685e284b1739bd16 is healthy: got healthy result from http://etcd01.ru-central1.internal:2379
cluster is healthy
```

##### Аналогичным образом добавляем третий узел

Выполняем оповещение кластера о появлении нового узла ***etcd03***:
```
# sudo etcdctl member add etcd03 http://etcd03.ru-central1.internal:2380
```

и корректируем файл конфигурации на etcd03:
```
otus@etcd03:~$ sudo nano /etc/default/etcd
```
```
ETCD_NAME="etcd03"
ETCD_DATA_DIR="/var/lib/etcd/pg-cluster"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://etcd03.ru-central1.internal:2380"
ETCD_INITIAL_CLUSTER="etcd02=http://etcd02.ru-central1.internal:2380,etcd01=http://etcd01.ru-central1.internal:2380,etcd03=http://etcd03.ru-central1.internal:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
ETCD_INITIAL_CLUSTER_TOKEN="pg-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://etcd03.ru-central1.internal:2379"
```

Перезапускаем etcd:
```
otus@etcd03:~$ sudo systemctl restart etcd
```

Проверяем
```
otus@etcd03:~$ sudo etcdctl cluster-health
member 3c2e492c60bebded is healthy: got healthy result from http://etcd02.ru-central1.internal:2379
member 685e284b1739bd16 is healthy: got healthy result from http://etcd01.ru-central1.internal:2379
member 8bfcfbf23268cd16 is healthy: got healthy result from http://etcd03.ru-central1.internal:2379
cluster is healthy
```

##### После успешного добавления всех узлов проводим окончательную корректировку конфигурации на всех узлах кластера

На каждой машине
```
cat /etc/hostname
```

в
```
sudo nano /etc/default/etcd
```

выставляем
```
ETCD_INITIAL_CLUSTER_STATE="existing"
ETCD_INITIAL_CLUSTER="etcd02=http://etcd02.ru-central1.internal:2380,etcd01=http://etcd01.ru-central1.internal:2380,etcd03=http://etcd03.ru-central1.internal:2380"
```

и делаем
```
sudo systemctl restart etcd
```

Проверяем
```
otus@etcd01:~$ sudo etcdctl cluster-health
member 3c2e492c60bebded is healthy: got healthy result from http://etcd02.ru-central1.internal:2379
member 685e284b1739bd16 is healthy: got healthy result from http://etcd01.ru-central1.internal:2379
member 8bfcfbf23268cd16 is healthy: got healthy result from http://etcd03.ru-central1.internal:2379
cluster is healthy
```

Кластер установлен и готов к работе.

### PostgreSQL

#### Ставим
```
sudo apt update && sudo apt upgrade -y -q && echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | sudo tee -a /etc/apt/sources.list.d/pgdg.list && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-14

# current (11-10-2023) key location
wget --quiet -O - http://apt.postgresql.org/pub/repos/apt/ACCC4CF8.asc | sudo apt-key add -

# если проблема с терминалом, скачиваем и ставим локально
:~$ ls -la ~/Загрузки/ACCC4CF8.asc 
-rw-rw-r-- 1 anisimov anisimov 4812 окт 11 08:56 /home/anisimov/Загрузки/ACCC4CF8.asc
:~$ sudo apt-key add ~/Загрузки/ACCC4CF8.asc 
[sudo] пароль для anisimov: 
Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).
OK
:~$ sudo apt-get update
Сущ:1 http://ru.archive.ubuntu.com/ubuntu jammy InRelease

```

#### Проверяем
```
pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```
```
pg_isready
/var/run/postgresql:5432 - accepting connections
```
```
otus@p1:~$ sudo su postgres
postgres@p1:/home/otus$ cd
postgres@p1:~$ psql

ALTER USER postgres password 'postgres';
```

#### Пользователь для patroni
```
CREATE USER adm password 'admin_321';
```
```
postgres=# SHOW hba_file;
              hba_file
-------------------------------------
 /etc/postgresql/14/main/pg_hba.conf

postgres=# \q
postgres@p1:~$
```

```
postgres@p1:~/14/main$ cd /etc/postgresql/14/main/
postgres@p1:/etc/postgresql/14/main$ nano pg_hba.conf
```
```
# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             0.0.0.0/0               md5
# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            scram-sha-256
host    replication     all             0.0.0.0/0               md5
host    replication     all             ::1/128                 scram-sha-256
```
```
postgres@p1:/etc/postgresql/14/main$ nano postgresql.conf
```
```
listen_addresses = '*'          # what IP address(es) to listen on;
```
```
otus@p1:~$ sudo systemctl stop postgresql@14-main
otus@p1:~$ sudo systemctl start postgresql@14-main
```

#### Проверяем сетевой вход
```
postgres@p1:~$ psql -h 10.129.0.5
Password for user postgres:
postgres=#
```
```
postgres@p2:~$ psql -h 10.129.0.5 -U adm postgres
Password for user adm:
postgres=>
```

#### Ставим остальные ноды.

Проверяем сетевой вход с других нод.
```
postgres@p1:~$ psql -h 10.129.0.35
Password for user postgres:
postgres=#
```
```
postgres@p2:~$ psql -h 10.129.0.35 -U adm postgres
Password for user adm:
postgres=>
```
```
otus@p1:~$ sudo systemctl stop postgresql@14-main
```
```
otus@p2:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```
```
sudo systemctl disable postgresql
sudo ln -s /usr/lib/postgresql/14/bin/* /usr/sbin/
```

#### Проверяем линки
```
ls -la /usr/sbin/ | grep /14/bin
```
```
sudo su postgres 
cd
rm -rf /var/lib/postgresql/14/main/* 
exit
```

### Patroni

#### Ставим питон
```
sudo apt-get install -y python3 python3-pip git mc
sudo pip3 install psycopg2-binary 
```

#### Патрони
```
sudo pip3 install patroni[etcd]
```

Делаем симлинк
```
sudo ln -s /usr/local/bin/patroni /bin/patroni
```

Включаем старт сервиса
```
sudo nano /etc/systemd/system/patroni.service
sudo nano /etc/patroni.yml
```

##### Node_1
```
scope: patroni_new
name: pp1

restapi:
  listen: 10.129.0.5:8008
  connect_address: 10.129.0.5:8008

etcd:
  hosts: etcd01.ru-central1.internal:2379,etcd02.ru-central1.internal:2379,etcd03.ru-central1.internal:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:

  initdb:  # Note: It needs to be a list (some options need values, others are switches)
  - encoding: UTF8
  - data-checksums

  pg_hba:  # Add following lines to pg_hba.conf after running 'initdb'
  - host replication replicator 0.0.0.0/0 md5
  - host all all 0.0.0.0/0 md5

  users:
    adm:
      password: admin_321
      options:
        - createrole
        - createdb

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.129.0.5:5432
  data_dir: /var/lib/postgresql/14/main
  bin_dir: /usr/lib/postgresql/14/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: rep-pass_321
    superuser:
      username: postgres
      password: zalando_321
    rewind:  # Has no effect on postgres 10 and lower
      username: rewind_user
      password: rewind_password_321
  parameters:
    unix_socket_directories: '.'

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```

##### Node_2
```
scope: patroni_new
name: pp2

restapi:
  listen: 10.129.0.35:8008
  connect_address: 10.129.0.35:8008

etcd:
  hosts: etcd01.ru-central1.internal:2379,etcd02.ru-central1.internal:2379,etcd03.ru-central1.internal:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:

  initdb:  # Note: It needs to be a list (some options need values, others are switches)
  - encoding: UTF8
  - data-checksums

  pg_hba:  # Add following lines to pg_hba.conf after running 'initdb'
  - host replication replicator 0.0.0.0/0 md5
  - host all all 0.0.0.0/0 md5

  users:
    adm:
      password: admin_321
      options:
        - createrole
        - createdb

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.129.0.35:5432
  data_dir: /var/lib/postgresql/14/main
  bin_dir: /usr/lib/postgresql/14/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: rep-pass_321
    superuser:
      username: postgres
      password: zalando_321
    rewind:  # Has no effect on postgres 10 and lower
      username: rewind_user
      password: rewind_password_321
  parameters:
    unix_socket_directories: '.'

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```

Стартуем
```
sudo -u postgres patroni /etc/patroni.yml
sudo systemctl is-enabled patroni 
sudo systemctl enable patroni 
sudo systemctl start patroni 
-- sudo systemctl stop patroni 
sudo patronictl -c /etc/patroni.yml list 
```

##### Adm
```
sudo patronictl -c /etc/patroni.yml reload patroni
-- если поломался старый кластер
patronictl -c /etc/patroni.yml remove 7088634863084761990
-- история
patronictl -c /etc/patroni.yml history
patronictl -c /etc/patroni.yml reinit patroni p2
```

#### Тестируем внешнее подключение
```
postgres@p2:~$ psql -h 10.129.0.35 -U postgres postgres -W
Password: zalando_321
postgres=# \q
```

#### Создаем тестовую базку
```
sudo su postgres
psql -h localhost -c "CREATE DATABASE otus;"
```

### Pgbouncer

#### Установка
```
sudo apt install -y pgbouncer
```
```
sudo nano /etc/pgbouncer/pgbouncer.ini
```
```
[databases]
otus = host=127.0.0.1 port=5432 dbname=otus 
[pgbouncer]
logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid
listen_addr = *
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
admin_users = admindb
```
```
sudo nano /etc/pgbouncer/userlist.txt
```
```
"admindb" "d9cfab6a2f1a0eb0c037e605cd578025"
```
```
sudo systemctl stop pgbouncer
sudo su postgres
-- rm ~/.pgpass
echo "localhost:5432:postgres:postgres:zalando_321">>~/.pgpass
chmod 600 ~/.pgpass
```
```
psql -h localhost
create user admindb with password 'root123';

postgres=# select usename,passwd from pg_shadow;
   usename   |                                                                passwd
-------------+---------------------------------------------------------------------------------------------------------------------------------------
 postgres    | SCRAM-SHA-256$4096:iAFYJbfi89ymN/uPocPD4Q==$/Msrt+JXCXQwZZYZO8BgFQAdBemhBvsRSkDbiO7ThO4=:DAiQm3QacqD5jQDcItRlngP6oOpsiGzN6clvH2QXnWY=
 replicator  | SCRAM-SHA-256$4096:QkqBghgPuTWVhVZcXVN1UQ==$Edy8QATMnCEwyBk695fg68RjueieF+QTDL1p8dmgp+Y=:+McKHLjR9OueMOBBrgPsPbQiHnLOvzPFQvntO9rZx8E=
 rewind_user | SCRAM-SHA-256$4096:Xrq+Vwfa/FrWK7d2mH37dQ==$exuFfci96EwpeLzhQ5GcSNfwmEngzE9S9pHEsg3zeCs=:lgXZX5WVe/e7FP3MOytv4iypsT5ghJnED6YVDQ4/ujs=
 adm         | SCRAM-SHA-256$4096:h4i9BVz9weHzQQBQ6TTWJA==$6HtXn+QkbVnzU1IzCS8rACZlKblGancWQbDsFrs+sFQ=:Y81JpsQHLIFhTSYWMMww2UMWmSWUVKzzij1MzS2y41s=
 admindb     | SCRAM-SHA-256$4096:ieufxe4a7/RFbC64TB85bg==$mad5rclf8OyEwvuutEu7clexsdkzD3cBWCMc2Se5az4=:R1YM0nSIV2e21mshYPXLL5nDYdjx4rTGufhaHFn+GZw=
```
```
sudo nano /etc/pgbouncer/pgbouncer.ini
```
```
admin_users = admindb,postgres
-- admin_users - кто имеет доступ к админке
```
```
sudo nano /etc/pgbouncer/userlist.txt
```
```
"postgres"  "SCRAM-SHA-256$4096:iAFYJbfi89ymN/uPocPD4Q==$/Msrt+JXCXQwZZYZO8BgFQAdBemhBvsRSkDbiO7ThO4=:DAiQm3QacqD5jQDcItRlngP6oOpsiGzN6clvH2QXnWY="
```

##### Adm
-- утилита шифрования пароля в мд5 есть например в поставке pgpool
-- sudo apt install pgpool2
-- pg_md5 root123

```
sudo systemctl status pgbouncer
sudo systemctl start pgbouncer
-- sudo systemctl restart pgbouncer 
-- sudo systemctl stop pgbouncer
```

##### Админка Pgbouncer
```
sudo su postgres 

postgres@p2:~$ psql -p 6432 pgbouncer -h localhost
Password for user postgres: zalando_321
pgbouncer=#

show clients;
```
-- https://eax.me/pgbouncer/

#### Тестируем соединение
```
postgres@p2:~$ psql -p 6432 -d otus -h localhost
Password for user postgres: zalando_321
otus=# \l+
                                                                    List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   |  Size   | Tablespace |                Description
-----------+----------+----------+-------------+-------------+-----------------------+---------+------------+--------------------------------------------
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |                       | 8553 kB | pg_default |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |                       | 8553 kB | pg_default | default administrative connection database
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +| 8401 kB | pg_default | unmodifiable empty database
           |          |          |             |             | postgres=CTc/postgres |         |            |
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +| 8401 kB | pg_default | default template for new databases
           |          |          |             |             | postgres=CTc/postgres |         |            |

otus=#
```

Понимаем, что законнектиться мы можем только туда, что прописано в
```
[databases]
otus = host=127.0.0.1 port=5432 dbname=otus 
```

При попытке пролезть куда-то еще получим
```
otus=# \c postgres
connection to server at "localhost" (::1), port 6432 failed: FATAL:  no such database: postgres
Previous connection kept
```

#### Проверяем сетевую доступность
```
sudo apt install net-tools
netstat -pltn
netstat -tln|grep 5432
netstat -tln|grep 6432
```

На самом деле она слегка врет.
При неправильной прописке пишет что Foreign Address
```
otus@p1:~$ netstat -pltn
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:6432            0.0.0.0:*               LISTEN      -
```
а на самом деле обращаем внимание, что в 
```
sudo nano /etc/pgbouncer/pgbouncer.ini
```

по дефолту где-то ниже может быть запись
```
listen_addr = localhost
```

и тогда netstat покажет *

а pg_bouncer на самом деле будет слушать только 127.0.0.1
И все попытки пробиться извне будут заканичиваться полным провалом.

#### Итого лезем снаружи
```
postgres@p2:~$ psql -p 6432 -d otus -h 10.129.0.5
Password for user postgres: zalando_321
otus=# \l+
```

#### В заключение пробуем пробиться PgAdmin
И сталкиваемся с проблемой, описанной тут https://stackoverflow.com/questions/17176011/pgadmin-connecting-to-postgresql-via-pgbouncer

На самом деле ничто не мешает нам прописать в секции
```
[databases]
otus = host=127.0.0.1 port=5432 dbname=otus 
postgres = host=127.0.0.1 port=5432 dbname=postgres
```

Проверено, работает.

Прочее
```
-- рестарт pg_bouncer, если был запущен как демон
pgbouncer -R -d /etc/pgbouncer/pgbouncer.ini 
-- sudo systemctl restart pgbouncer 
```

### HAProxy

#### Сам прокси
```
sudo apt install -y --no-install-recommends software-properties-common 
sudo add-apt-repository -y ppa:vbernat/haproxy-2.5
sudo apt install -y haproxy=2.5.\*
```

#### Клиент тулс
```
sudo apt update 
sudo apt upgrade -y 
sudo apt install -y postgresql-client-common 
sudo apt install postgresql-client -y
```

#### Тестим баунсер на основной ноде
```
psql -p 6432 -d otus -h 10.129.0.5 -U postgres
zalando_321
```

#### Тестируем ноды патрони по HTTP
```
otus@haproxy:~$ curl -v 10.129.0.5:8008/master
< HTTP/1.0 200 OK

otus@haproxy:~$ curl -v 10.129.0.5:8008/replica
< HTTP/1.0 503 Service Unavailable

otus@haproxy:~$ curl -v 10.129.0.35:8008/replica
< HTTP/1.0 200 OK

otus@haproxy:~$ curl -v 10.129.0.35:8008/master
< HTTP/1.0 503 Service Unavailable
```

#### Делаем конфигурацию
```
sudo nano /etc/haproxy/haproxy.cfg
```

#### Добавляем
```
listen postgres_write
    bind *:5432
    mode            tcp
    option httpchk
    http-check connect
    http-check send meth GET uri /master
    http-check expect status 200
    default-server inter 10s fall 3 rise 3 on-marked-down shutdown-sessions
    server p1 10.129.0.5:6432 check port 8008
    server p2 10.129.0.35:6432 check port 8008

listen postgres_read
    bind *:5432
  # bind *:5433
    mode            tcp
    http-check connect
    http-check send meth GET uri /replica
    http-check expect status 200
    default-server inter 10s fall 3 rise 3 on-marked-down shutdown-sessions
    server p1 10.129.0.5:6432 check port 8008
    server p2 10.129.0.35:6432 check port 8008
```
```
sudo systemctl restart haproxy.service
sudo systemctl status haproxy.service
```

#### Используем локальную тулзу для захода через прописанный в конфиге 5432 туда куда нужно (на < HTTP/1.0 200 OK)
```
psql -h localhost -d otus -U postgres -p 5432
zalando_321
```

Мы там где нужно
```
otus=# select pg_read_file('/etc/hostname') as hostname;
 hostname
----------
 p1      +
```

### KeepaliveD

Настроим keepalived на хостах с HAproxy
-- https://dasunhegoda.com/how-to-setup-haproxy-with-keepalived/833/
-- https://keepalived.readthedocs.io/en/latest/software_design.html
```
sudo apt install -y keepalived
```

-- Load balancing in HAProxy also requires the ability to bind to an IP address that are nonlocal, 
-- meaning that it is not assigned to a device on the local system. Below configuration is added so that 
-- floating/shared IP can be assigned to one of the load balancers. Below line get it done.
```
sudo nano /etc/sysctl.conf
net.ipv4.ip_nonlocal_bind=1
```
```
otus@haproxy:~$ sudo sysctl -p
net.ipv4.ip_nonlocal_bind = 1
```

Конфиги в файлах keepalived.conf
Посмотрим на какой интерфейс нужно добавить ip
```
otus@haproxy:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether d0:0d:10:3c:11:b0 brd ff:ff:ff:ff:ff:ff
    altname enp138s0
    altname ens8
    inet 10.129.0.9/24 metric 100 brd 10.129.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::d20d:10ff:fe3c:11b0/64 scope link
       valid_lft forever preferred_lft forever
```
```
sudo nano /etc/keepalived/keepalived.conf
```
```
global_defs {
# Keepalived process identifier
lvs_id haproxy_DH
}
# Script used to check if HAProxy is running
vrrp_script check_haproxy {
script "killall -0 haproxy"
interval 2
weight 2
}
# Virtual interface
# The priority specifies the order in which the assigned interface to take over in a failover
vrrp_instance VI_01 {
state MASTER
interface ens4
virtual_router_id 51
priority 101
# The virtual ip address shared between the two loadbalancers
virtual_ipaddress {
10.129.0.9
}
track_script {
check_haproxy
}
}
```
```
sudo service keepalived start
```
Посмотрим успешность добавления IP

ip a
