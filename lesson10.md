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
/home/otus/gp
/home/otus/gp/gpdata
/home/otus/gp/gpdata/master
/home/otus/gp/gpdata/primary
/home/otus/gp/gpdata/secondary
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
Edit gpinitsystem_singlenode Configuration File
> Аналогично gpinitsystem


```
gpssh-exkeys -f hostfile_gpinitsystem
```
```
gpinitsystem -c gpinitsystem_config
```
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
--gpssh-exkeys -f hostfile_gpinitsystem
--gpinitsystem -c gpinitsystem_config
gpstate -e
psql postgres
```













