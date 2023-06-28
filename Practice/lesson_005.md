Бэкапы Постгреса
========================

Развертывание
-------------------------

Создаем проект в ЯО: postgres2023-19720207.
Создаем каталог и сервис Compute Cloud.

Создаем виртуальныю машину

Идентификатор: epdkuglh8pcn3hf97gue
Имя: otus-db-pg-vm-1
Зона доступности: ru-central1-b
Сервисный аккаунт: otus

### Postgres
```
sudo apt install -y postgresql-15
```

### pg_probackup

Ставим

```
sudo sh -c 'echo "deb [arch=amd64] https://repo.postgrespro.ru/pg_probackup/deb/ $(lsb_release -cs) main-$(lsb_release -cs)" > /etc/apt/sources.list.d/pg_probackup.list' && sudo wget -O - https://repo.postgrespro.ru/pg_probackup/keys/GPG-KEY-PG_PROBACKUP | sudo apt-key add - && sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt install pg-probackup-15 pg-probackup-15-dbg
```

```
postgres@otus-db-pg-vm-1:~$ pg_probackup-15 version
pg_probackup-15 2.5.12 (PostgreSQL 15.0)
```

Создаем каталог на отдельном диске

```
postgres@otus-db-pg-vm-1:~$ sudo mkdir /mnt/data/backups
postgres@otus-db-pg-vm-1:~$ sudo chmod 777 /mnt/data/backups/
```

И устанавливаем переменную окружения **BACKUP_PATH**
```
postgres@otus-db-pg-vm-1:~$ cat ~/.bashrc
cat: /var/lib/postgresql/.bashrc: No such file or directory
```
Смотрим https://askubuntu.com/questions/127056/where-is-bashrc

Действуем
```
postgres@otus-db-pg-vm-1:~$ echo "BACKUP_PATH=/mnt/data/backups/">>~/.bashrc
postgres@otus-db-pg-vm-1:~$ ls -la
-rw-rw-r--  1 postgres postgres   31 May 19 07:20 .bashrc

postgres@otus-db-pg-vm-1:~$ echo "export BACKUP_PATH">>~/.bashrc
```
Проверяем
```
postgres@otus-db-pg-vm-1:~$ cat ~/.bashrc
BACKUP_PATH=/mnt/data/backups/
export BACKUP_PATH

postgres@otus-db-pg-vm-1:~$ echo $BACKUP_PATH
/mnt/data/backups/
```
Создадим роль ***backup*** в PostgreSQL для выполнения бекапов и дадим ему соответствующие права

https://habr.com/ru/companies/barsgroup/articles/515592/
> Для создания резервных копий необходимо подключение к СУБД, поэтому создаём в кластере PostgreSQL базу, 
> к которой будет происходить подключение для управления процессом резервного копирования 
> (с точки зрения безопасности это лучше, чем подключаться к продуктовой БД), 
> а также изменяем параметры базы данных:

```
postgres@otus-db-pg-vm-1:~$ psql -p 5433
postgres=# create database adm;
CREATE DATABASE
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
ALTER ROLE backup WITH REPLICATION;
```

Инициализируем наш бакап (**$BACKUP_PATH**)
```
postgres@otus-db-pg-vm-1:~$ pg_probackup-15 init
INFO: Backup catalog '/mnt/data/backups' successfully initialized
```

Итог
```
postgres@otus-db-pg-vm-1:~$ cd $BACKUP_PATH
postgres@otus-db-pg-vm-1:/mnt/data/backups$ ls -la
drwx------ 2 postgres postgres 4096 May 19 07:38 backups
drwx------ 2 postgres postgres 4096 May 19 07:38 wal
```

Инициализируем инстанс **main**
```
postgres@otus-db-pg-vm-1:~$ psql -p 5433 -c 'show data_directory;'
  data_directory
-------------------
 /mnt/data/15/main

postgres@otus-db-pg-vm-1:~$ pg_probackup-15 add-instance --instance 'main' -D /mnt/data/15/main
INFO: Instance 'main' successfully initialized
```

Создадим новую базу данных
```
postgres@otus-db-pg-vm-1:~$ psql -p 5433 -c "CREATE DATABASE otus;"
CREATE DATABASE
```

Создадим таблицу в этой базе данных и заполним ее тестовыми данными
```
postgres@otus-db-pg-vm-1:~$ psql -p 5433 otus -c "CREATE TABLE test(i int);"
psql -p 5433 otus -c "INSERT INTO test VALUES (10), (20), (30);"
psql -p 5433 otus -c "SELECT * FROM test;"
CREATE TABLE
INSERT 0 3
 i
----
 10
 20
 30
(3 rows)
```

### Подготовим кластер к резервному копированию

Включим ***checksums***
```
postgres@otus-db-pg-vm-1:~$ pg_ctlcluster 15 main stop
postgres@otus-db-pg-vm-1:~$ /usr/lib/postgresql/15/bin/pg_checksums -D /mnt/data/15/main --enable
Checksum operation completed
Files scanned:   1853
Blocks scanned:  103591
Files written:  1532
Blocks written: 103439
pg_checksums: syncing data directory
pg_checksums: updating control file
Checksums enabled in cluster

postgres@otus-db-pg-vm-1:~$ pg_ctlcluster 15 main start
postgres@otus-db-pg-vm-1:~$ psql -p 5433 otus -c "show data_checksums;"
 data_checksums
----------------
 on

postgres@otus-db-pg-vm-1:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
15  main    5433 online postgres /mnt/data/15/main           /var/log/postgresql/postgresql-15-main.log
```

Пытаемся бакапиться 
```
postgres@otus-db-pg-vm-1:~$ pg_probackup-15 backup --instance 'main' -b FULL --stream --temp-slot
INFO: Backup start, pg_probackup version: 2.5.12, instance: main, backup ID: RUWCL2, backup mode: FULL, wal mode: STREAM, remote: false, compress-algorithm: none, compress-level: 1
ERROR: pg_probackup-15 was built with PostgreSQL 15, but connection is made with 14
WARNING: Backup RUWCL2 is running, setting its status to ERROR
```
```
postgres@otus-db-pg-vm-1:~$ pg_probackup-15 show
BACKUP INSTANCE 'main'
=====================================================================================================================
 Instance  Version  ID      Recovery Time  Mode  WAL Mode  TLI  Time  Data  WAL  Zratio  Start LSN  Stop LSN  Status
=====================================================================================================================
 main      ----     RUWCL2  ----           FULL  STREAM    0/0    1s     0    0    1.00  0/0        0/0       ERROR
```

Читаем https://postgrespro.github.io/pg_probackup/#pbk-connection-opts
и правим
```
postgres@otus-db-pg-vm-1:~$ pg_probackup-15 backup --instance 'main' -b FULL --stream --temp-slot -p 5433
INFO: Backup start, pg_probackup version: 2.5.12, instance: main, backup ID: RUWD0G, backup mode: FULL, wal mode: STREAM, remote: false, compress-algorithm: none, compress-level: 1
INFO: This PostgreSQL instance was initialized with data block checksums. Data block corruption will be detected
WARNING: Current PostgreSQL role is superuser. It is not recommended to run pg_probackup under superuser.
INFO: Database backup start
INFO: wait for pg_backup_start()
INFO: Wait for WAL segment /mnt/data/backups/backups/main/RUWD0G/database/pg_wal/000000010000000100000055 to be streamed
INFO: PGDATA size: 810MB
INFO: Current Start LSN: 1/55000028, TLI: 1
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 4s
INFO: wait for pg_stop_backup()
INFO: pg_stop backup() successfully executed
INFO: stop_lsn: 1/550001A0
INFO: Getting the Recovery Time from WAL
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 4s
INFO: Validating backup RUWD0G
INFO: Backup RUWD0G data files are valid
INFO: Backup RUWD0G resident size: 826MB
INFO: Backup RUWD0G completed
```

Обращаем внимание на ругательство и принимаем к сведению.

В остальном
```
postgres@otus-db-pg-vm-1:~$ pg_probackup-15 show
BACKUP INSTANCE 'main'
===================================================================================================================================
 Instance  Version  ID      Recovery Time           Mode  WAL Mode  TLI  Time   Data   WAL  Zratio  Start LSN   Stop LSN    Status
===================================================================================================================================
 main      15       RUWD0G  2023-05-19 08:31:33+00  FULL  STREAM    1/0   14s  810MB  16MB    1.00  1/55000028  1/550001A0  OK
 main      ----     RUWCL2  ----                    FULL  STREAM    0/0    1s      0     0    1.00  0/0         0/0         ERROR
```

Победа

Внесем дополнительные данные
```
psql -p 5433 otus -c "insert into test values (4);"
```
```
postgres@otus-db-pg-vm-1:~$ pg_probackup-15 backup --instance 'main' -b DELTA --stream --temp-slot -U backup -p 5433
INFO: Backup start, pg_probackup version: 2.5.12, instance: main, backup ID: RUWE7V, backup mode: DELTA, wal mode: STREAM, remote: false, compress-algorithm: none, compress-level: 1
ERROR: could not connect to database postgres: connection to server on socket "/var/run/postgresql/.s.PGSQL.5433" failed: FATAL:  Peer authentication failed for user "backup"
WARNING: Backup RUWE7V is running, setting its status to ERROR
```

Зададим пароль backup
```
psql -c "ALTER USER backup PASSWORD 'otus123';" -p 5433
```

https://stackoverflow.com/questions/18664074/getting-error-peer-authentication-failed-for-user-postgres-when-trying-to-ge

```
psql -p 5433 otus -c "SHOW hba_file;"
nano /etc/postgresql/15/main/pg_hba.conf

local   all             postgres                           peer
local   all             all                                md5
```

Тестируем
```
postgres@otus-db-pg-vm-1:~$ psql -p5433 -U backup -W adm
Password:
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
Type "help" for help.

adm=> \q

postgres@otus-db-pg-vm-1:~$ pg_probackup-15 backup --instance 'main' -b DELTA --stream --temp-slot -U backup -p 5433
INFO: Backup start, pg_probackup version: 2.5.12, instance: main, backup ID: RUWJHY, backup mode: DELTA, wal mode: STREAM, remote: false, compress-algorithm: none, compress-level: 1
Password for user backup:
ERROR: query failed: ERROR:  permission denied for function pg_backup_start
```

Прописываем
```
echo "localhost:5433:replication:backup:otus123">>~/.pgpass
WARNING: password file "/var/lib/postgresql/.pgpass" has group or world access; permissions should be u=rw (0600) or less
```
```
chmod 600 ~/.pgpass
```
```
postgres@otus-db-pg-vm-1:~$ pg_probackup-15 backup --instance 'main' -b DELTA --stream --temp-slot -h localhost -U backup --pgdatabase=adm -p 5433
INFO: Backup start, pg_probackup version: 2.5.12, instance: main, backup ID: RUWOCQ, backup mode: DELTA, wal mode: STREAM, remote: false, compress-algorithm: none, compress-level: 1
Password for user backup:
```

Последняя попытка
```
nano ~/.pgpass

localhost:5433:replication:backup:otus123
localhost:5433:adm:backup:otus123
localhost:5433:otus:backup:otus123
```
```
pg_probackup-15 backup --instance 'main' -b DELTA --stream --temp-slot -h localhost -U backup --pgdatabase=adm -p 5433
INFO: Backup start, pg_probackup version: 2.5.12, instance: main, backup ID: RUWPE8, backup mode: DELTA, wal mode: STREAM, remote: false, compress-algorithm: none, compress-level: 1
INFO: This PostgreSQL instance was initialized with data block checksums. Data block corruption will be detected
INFO: Database backup start
```

Победа.

Итог битвы
```
postgres@otus-db-pg-vm-1:~$ pg_probackup-15 show

BACKUP INSTANCE 'main'
=====================================================================================================================================
 Instance  Version  ID      Recovery Time           Mode   WAL Mode  TLI  Time    Data   WAL  Zratio  Start LSN   Stop LSN    Status
=====================================================================================================================================
 main      15       RUWPE8  2023-05-19 12:58:59+00  DELTA  STREAM    1/1    6s  1341kB  32MB    1.00  1/5F000028  1/5F000168  OK
 main      15       RUWOQP  2023-05-19 12:45:00+00  FULL   STREAM    1/0   20s   810MB  32MB    1.00  1/5D000028  1/5D000168  OK
 main      15       RUWOCQ  2023-05-19 12:36:34+00  DELTA  STREAM    1/1   19s  1701kB  16MB    1.00  1/5B000028  1/5B0001A0  OK
 main      15       RUWO3F  ----                    DELTA  STREAM    0/1    6s       0     0    1.00  0/0         0/0         ERROR
 main      15       RUWO2R  ----                    DELTA  STREAM    0/1    5s       0     0    1.00  0/0         0/0         ERROR
 main      15       RUWNZ4  ----                    DELTA  STREAM    0/1    7s       0     0    1.00  0/0         0/0         ERROR
 main      15       RUWJPB  ----                    DELTA  STREAM    1/1    4s       0     0    1.00  1/59000028  0/0         ERROR
 main      15       RUWJLB  ----                    DELTA  STREAM    1/1    6s       0     0    1.00  1/57000060  0/0         ERROR
 main      15       RUWJHY  ----                    DELTA  STREAM    0/1    5s       0     0    1.00  0/0         0/0         ERROR
 main      ----     RUWES2  ----                    DELTA  STREAM    0/1    7s       0     0    1.00  0/0         0/0         ERROR
 main      ----     RUWEQD  ----                    DELTA  STREAM    0/1     0       0     0    1.00  0/0         0/0         ERROR
 main      ----     RUWE7V  ----                    DELTA  STREAM    0/1     0       0     0    1.00  0/0         0/0         ERROR
 main      15       RUWD0G  2023-05-19 08:31:33+00  FULL   STREAM    1/0   14s   810MB  16MB    1.00  1/55000028  1/550001A0  OK
 main      ----     RUWCL2  ----                    FULL   STREAM    0/0    1s       0     0    1.00  0/0         0/0         ERROR
```

Новый кластер
----------------------------
```
pg_createcluster 15 main2 -D /mnt/data/15/main2

Ver Cluster Port Status Owner    Data directory     Log file
15  main2   5434 down   postgres /mnt/data/15/main2 /var/log/postgresql/postgresql-15-main2.log

pg_ctlcluster 15 main2 start
postgres@otus-db-pg-vm-1:~$ psql -p5434
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
```

```
pg_ctlcluster 15 main2 stop
```
```
rm -rf /mnt/data/15/main2
```
```
pg_probackup-15 restore --instance 'main' -i 'RUWPE8' -D /mnt/data/15/main2
INFO: Restore of backup RUWPE8 completed.

pg_ctlcluster 15 main2 start

postgres@otus-db-pg-vm-1:~$ psql -p5434
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
Type "help" for help.

postgres=# \l+
                                                                                   List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges   |  Size   | Tablespace |                Description
-----------+----------+----------+-------------+-------------+------------+-----------------+-----------------------+---------+------------+--------------------------------------------
 adm       | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |                       | 7369 kB | pg_default |
 mtest     | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |                       | 773 MB  | pg_default |
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |                       | 7377 kB | pg_default |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |                       | 7453 kB | pg_default | default administrative connection database
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +| 7297 kB | pg_default | unmodifiable empty database
           |          |          |             |             |            |                 | postgres=CTc/postgres |         |            |
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +| 7525 kB | pg_default | default template for new databases
           |          |          |             |             |            |                 | postgres=CTc/postgres |         |            |
(6 rows)
```

*************************************************************************************

Бакап реплики
-------------------------------

Смотрим параметры
```
postgres@otus-db-pg-vm-1:/mnt/data/15/main2$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main2   5434 online postgres /mnt/data/15/main2          /var/log/postgresql/postgresql-15-main2.log
```

Инициализируем реплику
```
sudo su postgres
pg_probackup-15 add-instance --instance 'main2' -D /mnt/data/15/main2

postgres@otus-db-pg-vm-1:/mnt/data/15/main2$ pg_probackup-15 add-instance --instance 'main2' -D /mnt/data/15/main2
INFO: Instance 'main2' successfully initialized
```

Бакапимся
```
postgres@otus-db-pg-vm-1:/mnt/data/15/main2$ pg_probackup-15 backup --instance 'main2' -b FULL --stream --temp-slot -p 5434
INFO: Backup start, pg_probackup version: 2.5.12, instance: main2, backup ID: RV1Z0P, backup mode: FULL, wal mode: STREAM, remote: false, compress-algorithm: none, compress-level: 1
...
INFO: Database backup start
INFO: wait for pg_backup_start()
INFO: Wait for WAL segment /mnt/data/backups/backups/main2/RV1Z0P/database/pg_wal/000000010000000100000061 to be streamed
INFO: PGDATA size: 810MB
INFO: Current Start LSN: 1/61000060, TLI: 1
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 1m:58s
INFO: wait for pg_stop_backup()
INFO: pg_stop backup() successfully executed
INFO: stop_lsn: 1/610001D8
...
INFO: Backup RV1Z0P completed
```

Смотрим результат
```
postgres@otus-db-pg-vm-1:/home/otus$ pg_probackup-15 show
BACKUP INSTANCE 'main'
=====================================================================================================================================
 Instance  Version  ID      Recovery Time           Mode   WAL Mode  TLI  Time    Data   WAL  Zratio  Start LSN   Stop LSN    Status
=====================================================================================================================================
 main      15       RUWPE8  2023-05-19 12:58:59+00  DELTA  STREAM    1/1    6s  1341kB  32MB    1.00  1/5F000028  1/5F000168  OK
...

BACKUP INSTANCE 'main2'
====================================================================================================================================
 Instance  Version  ID      Recovery Time           Mode  WAL Mode  TLI   Time   Data   WAL  Zratio  Start LSN   Stop LSN    Status
====================================================================================================================================
 main2     15       RV1Z0P  2023-05-22 09:16:51+00  FULL  STREAM    1/0  2m:6s  810MB  32MB    1.00  1/61000060  1/610001D8  OK
```
```
echo "localhost:5434:replication:backup:otus123">>~/.pgpass
echo "localhost:5434:adm:backup:otus123">>~/.pgpass
echo "localhost:5434:otus:backup:otus123">>~/.pgpass
```
```
pg_probackup-15 backup --instance 'main2' -b DELTA --stream --temp-slot -h localhost -U backup --pgdatabase=adm -p 5434

postgres@otus-db-pg-vm-1:~$ pg_probackup-15 show

BACKUP INSTANCE 'main'
=====================================================================================================================================
 Instance  Version  ID      Recovery Time           Mode   WAL Mode  TLI  Time    Data   WAL  Zratio  Start LSN   Stop LSN    Status
=====================================================================================================================================
 ...
 
BACKUP INSTANCE 'main2'
======================================================================================================================================
 Instance  Version  ID      Recovery Time           Mode   WAL Mode  TLI   Time    Data   WAL  Zratio  Start LSN   Stop LSN    Status
======================================================================================================================================
 main2     15       RV1ZLB  2023-05-22 09:27:13+00  DELTA  STREAM    1/1    11s  1341kB  16MB    1.00  1/63000060  1/630001A0  OK
 main2     15       RV1Z0P  2023-05-22 09:16:51+00  FULL   STREAM    1/0  2m:6s   810MB  32MB    1.00  1/61000060  1/610001D8  OK
```

### Под нагрузкой

Пускаем нагрузку
```
pgbench -p 5434 -c 10 -j 2 -t 10000 mtest
```

И делаем то же самое
```
pg_probackup-15 backup --instance 'main2' -b DELTA --stream --temp-slot -h localhost -U backup --pgdatabase=adm -p 5434
```

Итого
```
postgres@otus-db-pg-vm-1:~$ pg_probackup-15 show

...

BACKUP INSTANCE 'main2'
=======================================================================================================================================
 Instance  Version  ID      Recovery Time           Mode   WAL Mode  TLI    Time    Data   WAL  Zratio  Start LSN   Stop LSN    Status
=======================================================================================================================================
 main2     15       RV1ZRC  2023-05-22 09:33:00+00  DELTA  STREAM    1/1  2m:21s   638MB  16MB    1.00  1/A64B75F0  1/A64B77A0  OK
 main2     15       RV1ZLB  2023-05-22 09:27:13+00  DELTA  STREAM    1/1     11s  1341kB  16MB    1.00  1/63000060  1/630001A0  OK
 main2     15       RV1Z0P  2023-05-22 09:16:51+00  FULL   STREAM    1/0   2m:6s   810MB  32MB    1.00  1/61000060  1/610001D8  OK
```

Видим разницу
```
================
   Time    Data 
================
 2m:21s   638MB 
    11s  1341kB 
  2m:6s   810MB 
```
