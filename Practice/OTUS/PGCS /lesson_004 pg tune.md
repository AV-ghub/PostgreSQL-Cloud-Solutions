Тюнинг Постгреса
========================

Развертывание
-------------------------

Создаем проект в ЯО: postgres2023-19720207.
Создаем каталог и сервис Compute Cloud.

    Создаем виртуальныю машину.
    Идентификатор: epdkuglh8pcn3hf97gue
    Имя: otus-db-pg-vm-1
    Зона доступности: ru-central1-b
    Сервисный аккаунт: otus

### Postgres

    sudo apt install -y postgresql-15

Pgbench
-------------------------

### Создаем любую базку для тестов, в нашем случае mtest.

Заливаем минимально приличным объемом

    postgres@otus-db-pg-vm-1:~$ pgbench -p 5433 -i -s 50 mtest
    dropping old tables...
    NOTICE:  table "pgbench_accounts" does not exist, skipping
    NOTICE:  table "pgbench_branches" does not exist, skipping
    NOTICE:  table "pgbench_history" does not exist, skipping
    NOTICE:  table "pgbench_tellers" does not exist, skipping
    creating tables...
    generating data (client-side)...
    5000000 of 5000000 tuples (100%) done (elapsed 34.22 s, remaining 0.00 s)
    vacuuming...
    creating primary keys...
    done in 65.87 s (drop tables 0.03 s, create tables 0.50 s, client-side generate 34.63 s, vacuum 11.37 s, primary keys 19.33 s).

Смотрим

    postgres@otus-db-pg-vm-1:~$ psql -p 5433
    postgres=# \c mtest
    mtest=# \dt
              List of relations
     Schema |       Name       | Type  |  Owner
    --------+------------------+-------+----------
     public | pgbench_accounts | table | postgres
     public | pgbench_branches | table | postgres
     public | pgbench_history  | table | postgres
     public | pgbench_tellers  | table | postgres
     public | ttt              | table | postgres
    (5 rows)

    mtest=# \l+
                                                                                   List of databases
    Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges   |  Size   | Tablespace |                Description
    -----------+----------+----------+-------------+-------------+------------+-----------------+-----------------------+---------+------------+--------------------------------------------
     mtest     | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |                       | 755 MB  | pg_default |

Нормально. Поехали дальше.

### Тест

    postgres@otus-db-pg-vm-1:~$ pgbench -p 5433 -c 10 -j 2 -t 100 mtest
    pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
    starting vacuum...end.
    transaction type: <builtin: TPC-B (sort of)>
    scaling factor: 50
    query mode: simple
    number of clients: 10
    number of threads: 2
    maximum number of tries: 1
    number of transactions per client: 100
    number of transactions actually processed: 1000/1000
    number of failed transactions: 0 (0.000%)
    latency average = 10.382 ms
    initial connection time = 14.855 ms
    tps = 963.193487 (without initial connection time)
    -----------------------------------------------

И далее видим странные качания

    latency average = 13.871 ms
    initial connection time = 12.938 ms
    tps = 720.918681 (without initial connection time)
    -----------------------------------------------
    latency average = 31.490 ms
    initial connection time = 13.412 ms
    tps = 317.561836 (without initial connection time)
    -----------------------------------------------
    latency average = 24.942 ms
    initial connection time = 13.221 ms
    tps = 400.925014 (without initial connection time)
    -----------------------------------------------
    latency average = 15.622 ms
    initial connection time = 13.266 ms
    tps = 640.115528 (without initial connection time)
    -----------------------------------------------
    latency average = 13.243 ms
    initial connection time = 13.210 ms
    tps = 755.114770 (without initial connection time)
    -----------------------------------------------
    latency average = 22.170 ms
    initial connection time = 13.088 ms
    tps = 451.059991 (without initial connection time)
    -----------------------------------------------
    latency average = 24.788 ms
    initial connection time = 13.058 ms
    tps = 403.424753 (without initial connection time)
    -----------------------------------------------
    latency average = 21.832 ms
    initial connection time = 12.873 ms
    tps = 458.048904 (without initial connection time)
    -----------------------------------------------
    latency average = 17.058 ms
    initial connection time = 13.759 ms
    tps = 586.225575 (without initial connection time)
    -----------------------------------------------

И это при том что

    starting vacuum...end.

Принимаем среднее tps = 525

Смотрим таблички

    mtest=# select
    table_name,
    pg_size_pretty(pg_relation_size(quote_ident(table_name))),
    pg_relation_size(quote_ident(table_name))
    from information_schema.tables
    where table_schema = 'public'
    order by 3 desc;
    table_name    | pg_size_pretty | pg_relation_size
    ------------------+----------------+------------------
     pgbench_accounts | 645 MB         |        676257792
     pgbench_history  | 208 kB         |           212992
     pgbench_tellers  | 104 kB         |           106496
     pgbench_branches | 16 kB          |            16384
     ttt              | 8192 bytes     |             8192
    (5 rows)

Тюнинг
-----------------------------------------------

### shared_buffers

    mtest=# ALTER SYSTEM SET shared_buffers TO '400MB';
    otus@otus-db-pg-vm-1:~$ sudo pg_ctlcluster 15 main restart

По ощущениям стало хуже

    max tps = 689.913057
    min tps = 172.392660

Грубо 480.

Увеличиваем до 900, получаем вообще финиш.
Вертаем обратно на 128.
И все взлетает.
Максимум достиг 

    tps = 1293.512131

Феномен пока непонятен и требует изучения.
Хотя

    postgres@otus-db-pg-vm-1:~$ free -m
                   total        used        free      shared  buff/cache   available
    Mem:            3922         296         247          88        3379        3248

показывает более чем приличный запас для развития.

Попробуем повысить градус

    postgres@otus-db-pg-vm-1:~$ pgbench -p 5433 -c 10 -j 2 -t 10000 mtest

##### 128МБ

    latency average = 30.945 ms
    initial connection time = 13.324 ms
    tps = 323.154777 (without initial connection time)

##### 900МБ

    postgres@otus-db-pg-vm-1:~$ pgbench -p 5433 -c 10 -j 2 -t 10000 mtest
    latency average = 16.018 ms
    initial connection time = 15.490 ms
    tps = 624.280743 (without initial connection time)

Похоже на правду.
Таки делаем заключение о накладных расходах на какие-то переключения и прогрев.
Механику видимо надо копать на очень глубоком уровне.

### work_mem

    postgres=# show work_mem;
    work_mem
    ----------
    4MB

#### Рекомендации

    Total RAM * 0.25 / max_connections

Считаем и получаем по нашему бенчу пирмерно 10МБ. Ставим.

    latency average = 15.744 ms
    initial connection time = 14.297 ms
    tps = 635.159866 (without initial connection time)

Совсем чуть, но все-таки что-то есть.

### maintenance_work_mem

#### Рекомендации

    Total RAM * 0.05

Получаем 200МБ.

    latency average = 12.737 ms
    initial connection time = 14.399 ms
    tps = 785.108039 (without initial connection time)

Прямо неплохо.
Опять же, по честному надо тестить изолированно. Тут возможно комплексное влияние.
Ну да ладно, смотрим пока в принципе.

### effective_cache_size

#### Рекомендации

    50% of the machine’s total RAM

    postgres=# show effective_cache_size;
    effective_cache_size
    ----------------------
    4GB

Ну тут делать нечего, ок.

В заключение запускаем змия.

### synchronous_commit

    postgres=# ALTER SYSTEM SET synchronous_commit = off;

    latency average = 2.461 ms
    initial connection time = 14.093 ms
    tps = 4063.415945 (without initial connection time)

Оно и понятно. Диски это всегда основа БД.

Идем в полный отрыв.

### fsync

    postgres=# ALTER SYSTEM SET fsync = off;

На удивление отрыв не очень состоялся.

    latency average = 2.595 ms
    initial connection time = 14.082 ms
    tps = 3854.253857 (without initial connection time)

Пробуем еще раз и 

    latency average = 2.527 ms
    initial connection time = 13.996 ms
    tps = 3957.096370 (without initial connection time)

Вертаем обратно

    latency average = 2.466 ms
    initial connection time = 14.062 ms
    tps = 4055.804792 (without initial connection time)

Результат стабильный.

В общем овчинка потерь не стоит.
На том и порешим.

В целом да, synchronous_commit прикольная вещь для недорогого хайпа.
Надо будет еще посмотреть как все перчисленное ведет себя в среде SSD по сравнению с крутящейся болванкой.
По идее там хайпа не будет или будет но вовсе не такой, посмотрим попозжа.
