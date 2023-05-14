Настройка дисков для Постгреса
========================

Предварительные установки
-------------------------

Создаем проект в ЯО: postgres2023-19720207 (https://github.com/AV-ghub/PostgreSQL-Cloud-Solutions/blob/main/lesson2.md).
Создаем каталог и сервис Compute Cloud. 

Создаем виртуальныю машину.

    Идентификатор: epdkuglh8pcn3hf97gue
    Имя: otus-db-pg-vm-1
    Зона доступности: ru-central1-b
    Сервисный аккаунт: otus

## Создание и работа с диском

На вкладке Диски жмем Создать диск.
Выбираем что-нибудь, создаем. Ждем Creating -> Ready.
Идем в виртуалку, жмем +Присоединить диск.

Логинимся в виртуалку.
Смотрим что у нас есть

    otus@otus-db-pg-vm-1:~$ lsblk
    NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
    vda    252:0    0    20G  0 disk
    ├─vda1 252:1    0     1M  0 part
    └─vda2 252:2    0    20G  0 part /var/snap/firefox/common/host-hunspell
                                 /
    vdb    252:16   0    20G  0 disk
    otus@otus-db-pg-vm-1:~$

Отлично, все на месте, монтируем.

Сначала железо (https://sh-tsang.medium.com/partitioning-formatting-and-mounting-a-hard-drive-in-linux-ubuntu-18-04-324b7634d1e0)

    root@otus-db-pg-vm-1:/mnt/data# sudo parted /dev/vdb
    GNU Parted 3.4
    Using /dev/vdb
    Welcome to GNU Parted! Type 'help' to view a list of commands.
    (parted) mklabel gpt
    (parted) mkpart primary 0GB 20GB
    (parted) quit
    Information: You may need to update /etc/fstab.

Итого

    root@otus-db-pg-vm-1:/mnt/data# lsblk
    NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
    ...                                 /
    vdb    252:16   0    20G  0 disk
    └─vdb1 252:17   0  18.6G  0 part

Далее

    otus@otus-db-pg-vm-1:~$ sudo mkfs.ext4 /dev/vdb1
    ...
    Creating filesystem with 4882432 4k blocks and 1220608 inodes
    Filesystem UUID: 2d89a129-e8e1-48c7-b8b1-4d0933a862b4
    ...

    root@otus-db-pg-vm-1:/mnt/data# fdisk -l
    Disk /dev/vdb: 20 GiB, 21474836480 bytes, 41943040 sectors
    ...

    root@otus-db-pg-vm-1:/mnt/data# sudo mount /dev/vdb1 /mnt/data
    
    root@otus-db-pg-vm-1:/mnt/data# cd /mnt/data/
    
    root@otus-db-pg-vm-1:/mnt/data# ll
    total 24
    drwxr-xr-x 3 root root  4096 May 12 16:44 ./
    drwxr-xr-x 3 root root  4096 May 12 16:26 ../
    drwx------ 2 root root 16384 May 12 16:44 lost+found/

Ручной проброс состоялся. Идем дальше.

    nano /etc/fstab
    /dev/vdb1     /mnt/data      ext4        defaults      0       0

Перезагружаемся

    otus@otus-db-pg-vm-1:~$ cd /mnt/data
    otus@otus-db-pg-vm-1:/mnt/data$ ll
    total 28
    drwxr-xr-x 4 root root  4096 May 12 17:08 ./
    drwxr-xr-x 3 root root  4096 May 12 16:26 ../
    drwx------ 2 root root 16384 May 12 17:04 lost+found/

Все на месте.

## Postgres

Ставим

    sudo apt install -y postgresql-15

При попытке запуска

    otus@otus-db-pg-vm-1:~$ psql -U postgres
    psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "postgres"

Решение тут https://stackoverflow.com/questions/69676009/psql-error-connection-to-server-on-socket-var-run-postgresql-s-pgsql-5432

Проверяем

    otus@otus-db-pg-vm-1:~$ sudo -u postgres psql
    could not change directory to "/home/otus": Permission denied
    psql (15.2 (Ubuntu 15.2-1.pgdg22.04+1), server 14.7 (Ubuntu 14.7-0ubuntu0.22.04.1))
    Type "help" for help.

    postgres=# \c
    psql (15.2 (Ubuntu 15.2-1.pgdg22.04+1), server 14.7 (Ubuntu 14.7-0ubuntu0.22.04.1))
    You are now connected to database "postgres" as user "postgres".
    postgres=#

Действительно так.

Делаем

    otus@otus-db-pg-vm-1:~$ sudo nano /etc/postgresql/15/main/pg_ident.conf
    # MAPNAME       SYSTEM-USERNAME         PG-USERNAME
    otus             otus               postgres

Рестартуем кластер. Не помогло.
Тогда делаем просто

    sudo -i -u postgres

И работаем. Разберемся позже, не цель задания.

    postgres=# create database test;
    CREATE DATABASE
    postgres=# \c test
    psql (15.2 (Ubuntu 15.2-1.pgdg22.04+1), server 14.7 (Ubuntu 14.7-0ubuntu0.22.04.1))
    You are now connected to database "test" as user "postgres".
    test=# create table tt(id int, val text);
    CREATE TABLE
    test=# insert into tt values(1, 'qq'), (2, 'ww');
    INSERT 0 2
    test=# select * from tt;
     id | val
    ----+-----
      1 | qq
      2 | ww
    (2 rows)

Параметры установки

    postgres@otus-db-pg-vm-1:~$ /usr/lib/postgresql/15/bin/pg_config --configure
    '--mandir=/usr/share/postgresql/15/man' 
    '--bindir=/usr/lib/postgresql/15/bin' 
    '--with-pgport=5432' 
 
Далее.

Права

    postgres@otus-db-pg-vm-1:~$ exit
    logout
    otus@otus-db-pg-vm-1:~$ sudo chown -R postgres:postgres /mnt/data/

Обезвреживаем

    otus@otus-db-pg-vm-1:~$ sudo pg_ctlcluster 15 main stop
    otus@otus-db-pg-vm-1:~$ sudo pg_ctlcluster 15 main status
    pg_ctl: no server running

Сливаем в нужное место

    otus@otus-db-pg-vm-1:~$ sudo mv /var/lib/postgresql/15 /mnt/data

Констатируем потерю

    otus@otus-db-pg-vm-1:~$ cd  /var/lib/postgresql/15
    -bash: cd: /var/lib/postgresql/15: No such file or directory
    
Смотрим что осталось    
    
    otus@otus-db-pg-vm-1:~$ cd  /var/lib/postgresql/
    otus@otus-db-pg-vm-1:/var/lib/postgresql$ ll
    total 24
    drwxr-xr-x  4 postgres postgres 4096 May 13 18:14 ./
    drwxr-xr-x 45 root     root     4096 May  5 17:47 ../
    drwxr-xr-x  3 postgres postgres 4096 Apr 28 16:27 14/
    -rw-------  1 postgres postgres  257 May 13 18:08 .bash_history
    drwxr-xr-x  4 root     root     4096 May 10 04:49 data/
    -rw-------  1 postgres postgres  180 May 13 18:04 .psql_history

Пытаемся

    otus@otus-db-pg-vm-1:/var/lib/postgresql$ sudo -u postgres pg_ctlcluster 15 main start
    Error: /var/lib/postgresql/15/main is not accessible or does not exist

Оно и понятно, стартовать нечему.
Кластер тут

    otus@otus-db-pg-vm-1:/etc/postgresql/15/main$ cd /mnt/data
    otus@otus-db-pg-vm-1:/mnt/data$ ll
    total 32
    drwxr-xr-x 5 postgres postgres  4096 May 13 18:14 ./
    drwxr-xr-x 3 root     root      4096 May 12 16:26 ../
    drwxr-xr-x 3 postgres postgres  4096 May 13 17:31 15/
    drwx------ 2 postgres postgres 16384 May 12 17:04 lost+found/
    drwxr-xr-x 2 postgres postgres  4096 May 12 17:08 tmp/

Ремонтируем

    otus@otus-db-pg-vm-1:/etc/postgresql/15/main$ sudo nano postgresql.conf
    data_directory = '/mnt/data/15/main'            # use data in another directory

Вот теперь стартуем    
    
    otus@otus-db-pg-vm-1:/etc/postgresql/15/main$ sudo -i -u postgres
    postgres@otus-db-pg-vm-1:~$ pg_ctlcluster 15 main start
    Warning: the cluster will not be running as a systemd service. Consider using systemctl:
      sudo systemctl start postgresql@15-main
    
Залезаем    
    
    postgres@otus-db-pg-vm-1:~$ psql
    psql (15.2 (Ubuntu 15.2-1.pgdg22.04+1), server 14.7 (Ubuntu 14.7-0ubuntu0.22.04.1))
    Type "help" for help.
    postgres=# \c
    psql (15.2 (Ubuntu 15.2-1.pgdg22.04+1), server 14.7 (Ubuntu 14.7-0ubuntu0.22.04.1))
    You are now connected to database "postgres" as user "postgres".
    
И радостно селектим    
    
    postgres=# \c test
    psql (15.2 (Ubuntu 15.2-1.pgdg22.04+1), server 14.7 (Ubuntu 14.7-0ubuntu0.22.04.1))
    You are now connected to database "test" as user "postgres".
    test=# select * from tt;
     id | val
    ----+-----
      1 | qq
      2 | ww
    (2 rows)

    test=#


## ***********************************************


Создаем вторую виртуалку otus-db-pg-vm-2.
Из приатаченного к otus-db-pg-vm-1 hdd-storage создаем имидж.
На базе него создаем диск hdd-storage-2.
Фактически клонируем.

> Хинт
> 
> По ходу на этом этапе мы можем перемещаться между зонами, но это не точно.

Прицепляем hdd-storage-2 к otus-db-pg-vm-2.
Грузимся и видим знакомую картину

    otus@otus-db-pg-vm-2:~$ lsblk
    NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
    vdb    252:16   0   20G  0 disk
    └─vdb1 252:17   0 18.6G  0 part

Повторяем магию 

    otus@otus-db-pg-vm-2:~$ sudo mount /dev/vdb1 /mnt/data
    
И получаем нашу базу в /mnt/data/15/main

Ставим postgres-15.
Далее все как обычно.
Останавливаем кластер.
Переименовываем каталог с данными.
Правим конфиг 

    data_directory = '/mnt/data/15/main' 

Стартуем, заходим и видим кластер в нашем распоряжении.
Переключаемся на нашу базу, селектим и понимаем что мы попали куда хотели.
