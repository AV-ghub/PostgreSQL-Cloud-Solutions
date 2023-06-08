Разворачиваем и настраиваем БД с большими данными
-----------------------------------------------

## Greenplum

> https://greenplum.org/install-greenplum-oss-on-ubuntu/

### Установка

Обращаем внимание на поддерживаемые версии в описании.
Для неподдерживаемыз версий гринплам не будет ставиться и грузить репозиторий.

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
which gpssh
```
Exact path of the Greenplum software directory:
> /opt/greenplum-db-6.24.3/bin/gpssh

```
cd /opt/greenplum-db-6.24.3
```
```
source greenplum_path.sh
```

Copy a Greenplum cluster configuration file template into local directory for editing
```
cd
```
```
cp $GPHOME/docs/cli_help/gpconfigs/gpinitsystem_singlenode .
cp $GPHOME/docs/cli_help/gpconfigs/gpinitsystem_config .
cp $GPHOME/docs/cli_help/gpconfigs/hostlist_singlenode .
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
MACHINE_LIST_FILE=./hostlist_singlenode
declare -a DATA_DIRECTORY=(/home/otus/gpadmin/primary /home/otus/gpadmin/primary)
MASTER_HOSTNAME=gp
MASTER_DIRECTORY=/home/otus/gpadmin/master
```
Edit hostlist_singlenode Configuration File
```
gp
```
Edit gpinitsystem_singlenode Configuration File
> Аналогично gpinitsystem


























