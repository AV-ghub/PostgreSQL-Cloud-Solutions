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







