Разворачиваем и настраиваем БД с большими данными
-----------------------------------------------

## Greenplum

> https://greenplum.org/install-greenplum-oss-on-ubuntu/

### Установка

Обращаем внимание на поддерживаемые версии в описании.
Для неподдерживаемыз версий гринплам не будет ставиться и грузить репозиторий.
```
uname -a
lsb_release -a
```
```
sudo apt update
```
```
sudo apt install software-properties-common
```
```
sudo add-apt-repository ppa:greenplum/db
```
```
sudo apt update
```

Install the latest Greenplum Database release:
```
sudo apt install greenplum-db-6
```
```
export GPHOME="/opt/greenplum-db-6.24.3"
```
```
which gpssh
```
Exact path of the Greenplum software directory:
> /opt/greenplum-db-6.24.3/bin/gpssh

```
cd $GPHOME
```
```
source greenplum_path.sh
```

Делаем все каталоги
```
mkdir /home/otus/gp
mkdir /home/otus/gp/gpdata
mkdir /home/otus/gp/gpdata/master
mkdir /home/otus/gp/gpdata/primary
mkdir /home/otus/gp/gpdata/secondary
```

Copy a Greenplum cluster configuration file template into local directory for editing
```
cd /home/otus/gp
```
```
cp $GPHOME/docs/cli_help/gpconfigs/gpinitsystem_config .
cp $GPHOME/docs/cli_help/gpconfigs/hostfile_gpinitsystem .
```
Имя машины
```
cat /etc/hostname
or
hostname
```
> gp
  
Edit gpinitsystem Configuration File
```
MACHINE_LIST_FILE=/home/otus/gp
declare -a DATA_DIRECTORY=(/home/otus/gp/gpdata/primary /home/otus/gp/gpdata/primary)
declare -a MIRROR_DATA_DIRECTORY=(/home/otus/gp/gpdata/secondary /home/otus/gp/gpdata/secondary)
MASTER_HOSTNAME=gp
MASTER_DIRECTORY=/home/otus/gp/gpdata/master
MACHINE_LIST_FILE=/home/otus/gp/hostfile_gpinitsystem
DATABASE_NAME=test
```
Edit hostlist_singlenode Configuration File
```
gp
```
Прописываем в gpinitsystem имя базы по логину (напр otus).
Чтобы не указывать имя базы в psql.

Далее инициализируем конфигурацию.
Проверяем еще раз параметры конфигурации.
Второй раз инициализацию будет сделать скорее всего нельзя (надо копать).
```
gpssh-exkeys -f hostfile_gpinitsystem
```
```
gpinitsystem -c gpinitsystem_config
```
Проверяем статус сервиса
```
gpstate -e
```
> MASTER_DATA_DIRECTORY not set

Глюк установщика
```
export MASTER_DATA_DIRECTORY="/home/otus/gp/gpdata/master/gpseg-1"
```
```
gpstate -e
```
```
psql postgres
```
Перезапуск с новой сессией
```
export MASTER_DATA_DIRECTORY="/home/otus/gp/gpdata/master/gpseg-1"
export GPHOME="/opt/greenplum-db-6.24.3"
cd $GPHOME
source greenplum_path.sh
gpstate -e
```
```
psql
```
Перманентим скрипт в $HOME/.bashrc в конец файлика
```
nano $HOME/.bashrc
```
```
export MASTER_DATA_DIRECTORY="/home/otus/gp/gpdata/master/gpseg-1"
export GPHOME="/opt/greenplum-db-6.24.3"
cd $GPHOME
source greenplum_path.sh
gpstart -d /home/otus/gp/gpdata/master/gpseg-1 -l /home/otus/gp/logs -q -a
cd $HOME
```

Запуск сервисом
Вар1
```
[Service]
Type=notify
EnvironmentFile=/home/otus/env/gp
ExecStart=/bin/bash -c "cd $GPHOME; source greenplum_path.sh; gpstart -d /home/otus/gp/gpdata/master/gpseg-1 -l /home/otus/gp/logs -q -a"
ExecStop=/bin/bash -c "cd $GPHOME; gpstop -q -a"
User=otus

[Install]
WantedBy=multi-user.target
```
вар2
```
[Service]
Type=notify
EnvironmentFile=/home/otus/env/gp
ExecStart=/bin/bash -c "cd /opt/greenplum-db-6.24.3; source greenplum_path.sh; gpstart -d /home/otus/gp/gpdata/master/gpseg-1 -l /home/otus/gp/logs -q -a"
ExecStop=/bin/bash -c "cd /opt/greenplum-db-6.24.3; gpstop -q -a"
User=otus

[Install]
WantedBy=multi-user.target
```
Адм
```
sudo nano /etc/systemd/system/gp-otus1.service

sudo systemctl disable gp-otus1.service
sudo systemctl enable gp-otus1.service
sudo systemctl daemon-reload
sudo systemctl start gp-otus1.service
sudo systemctl stop gp-otus1.service
sudo systemctl restart gp-otus1.service

sudo systemctl status gp-otus1.service
journalctl -u gp-otus1
```

### Работа с Greenplum

Версия
```
gpssh --version
gpssh version 6.24.3 build commit:25d3498a400ca5230e81abb94861f23389315213 Open Source
```

Список баз
psql -l

Список команд
```
psql
\?
```

Файлики
```
cd $MASTER_DATA_DIRECTORY
ls -la
```

HBA
```
cd $MASTER_DATA_DIRECTORY
cat pg_hba.conf
cat postgresql.conf
gpconfig --help
gpconfig -l | grep mem
```

Check
```
gpconfig -s work_mem
Values on all segments are consistent
GUC          : work_mem
Master  value: 32MB
Segment value: 32MB
```

Set
```
gpconfig -s work_mem -v 128MB
gpstop --help
gpstop -u
```

Segment configuration
```
psql postgres
Describe
\d gp_segment_configuration;
select * from gp_segment_configuration;
select * from pg_stat_activity;
\d pg_class
```

### Postgres

#### Генерация данных
Гоняем pg_bench. Берем таблицу и сливаем ее в сторону через select into.
Получаем ~650МБ, для тестов достаточно.

#### Выгрузка наружу
Выгружаем через COPY.
```
time psql mtest -c "COPY test TO STDOUT WITH CSV DELIMITER ';' HEADER ENCODING 'UTF-8'" > test.csv
```
Получили 23 сек.

#### Тестируем загрузку
```
CREATE TABLE test1 AS
TABLE test
WITH NO DATA;
```
``` 
COPY test1(aid, bid, abalance, filler)
FROM '/mnt/unload/test.csv'
DELIMITER ';'
CSV HEADER;
```
Получили 41 сек.

### Загрузка данных в greenplum

В предыдущем разделе данные сливали на прицепленный диск.
Попробуем его отцепить от машины с postgres и прецепить к машине с greenplum.
```
psql
otus=# create table test1(aid integer, bid integer, abalance integer, filler text) distributed randomly;
CREATE TABLE
\q
```
```
otus@gp1:~$ gpfdist -d /mnt/data/unload/ -p 8081 > gpfdist.log 2>&1 &
[1] 1536
```
```
psql
otus=# CREATE READABLE EXTERNAL TABLE test1_ext (like test1)
otus-# LOCATION ('gpfdist://10.129.0.25:8081//test.csv')
otus-# FORMAT 'CSV' (HEADER)
otus-# LOG ERRORS SEGMENT REJECT LIMIT 50;
NOTICE:  HEADER means that each one of the data files has a header row
CREATE EXTERNAL TABLE
```
```
nano test.yaml
```
```
---
VERSION: 1.0.0.1
DATABASE: otus
USER: otus
HOST: gp1
PORT: 5432
GPLOAD:
   INPUT:
    - SOURCE:
         LOCAL_HOSTNAME:
           - gp1
         PORT: 8083
         FILE:
           - /mnt/data/unload/test.csv
    - COLUMNS:
           - aid: integer
           - bid: integer
           - abalance: integer
           - filler: text
    - FORMAT: text
    - DELIMITER: ';'
    - ERROR_LIMIT: 25
    - LOG_ERRORS: True
   OUTPUT:
    - TABLE: public.test1
    - MODE: INSERT
```
```
gpload -f test.yaml
```
Получаем порядка 70 сек.
Результат можно объяснить тем, что хотя мы и грузим данные с прицепленного диска, но распределение по разным дискам у нас в greenplum не настроено.
И накладные расходы на параллелизацию забивают профит от распараллеливания при чтении с одного и того же диска.
Также видимо играет роль объем данных, у нас он очень маленький.

### Тестируем аналитический запрос

Для эксперимента возьмем простейший, но выгнутый запрос с аналитикой.

select max(aid) from test1 where bid in (select bid from test1 where abalance < 0);

Результаты здесь интересные (стоит принять во внимание также недетерминированное состояние клауда).

В postgres запрос на холодную уходит в 50 сек.
Дальше с переменным дерганием плавает в р-не 10-30 сек.
И наконец с окончательного прогрева стабилизируется на отметке 4-6 сек.

В greenplum запрос сразу стартует с 4.7 сек и далее плаваем в р-не 4-5 сек.
Т.е. во-первых, фаза прогрева кэша фактически отсутствует.
Видимо чтение идет всегда с дисков.
Это надо дополнительно исследовать и почитать про архитектуру и особенности движка.
Наверное, для аналитики это имеет смысл.
Т.к. аналитические данные в кэш не начитаешься, а структура хранения как раз спроектирована для подобных операций.
Но это догадки.
Во-сторых, помним, что мы работаем с очень маленьким набором.
Возможно на серьезных наборах разница начнет взлетать.
В-третьих, повторяясь, хранилище у нас на одном диске.
Если сделать их десяток с грамотным распределением и разнести на прицепленные диски, то возможно реально будет чему удивиться в сравнении.

Но даже просто факт отсутствия прогрева со стабильным результатом уже на первом запросе - это уже существенный результат в пользу greenplum на больших данных.









































