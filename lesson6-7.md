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

##### Устанавливаем etcd на первом узле
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
