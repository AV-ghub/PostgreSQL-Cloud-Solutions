## CentOS
### Проверка установки
```
which psql
/usr/bin/which: no psql in (/usr/lib64/qt-3.3/bin:/usr/local/bin:/usr/local/sbin:/usr/bin:/usr/sbin:/bin:/sbin:/home/admin/.local/bin:/home/admin/bin)

rpm -qa | grep sql

yum list installed | grep sql
akonadi-mysql.x86_64                   1.9.2-4.el7                     @anaconda
qt-mysql.x86_64                        1:4.8.7-9.el7_9                 @anaconda
sqlite.x86_64                          3.7.17-8.el7_7.1                @anaconda
```

### Установка
> https://www.dmosk.ru/miniinstruktions.php?mini=postgresql-install
>
> https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-centos-7
>
> https://info.gosuslugi.ru/articles/%D0%A1%D0%BF%D0%B5%D1%86%D0%B8%D1%84%D0%B8%D0%BA%D0%B0_%D1%83%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B8_PostgreSQL_%D0%B4%D0%BB%D1%8F_CentOS/

Получение номера версии ОС для получения нужного номера репозитория описано [тут](https://unix.stackexchange.com/questions/612054/how-do-i-determine-which-version-of-the-rhel-im-building-on)

Для установки произвольной (последней) версии Posgres необходимо сначала подключить репозиторий.
Находится он соответственно тут
```
echo https://download.postgresql.org/pub/repos/yum/reporpms/EL-`rpm -E %{rhel}`-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```
Загружаем ([не работает](https://postgrespro.ru/list/thread-id/2529687))
```
sudo yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-`rpm -E %{rhel}`-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-42.0-32.noarch.rpm
```
<details>
<summary>Output</summary>

```
[admin@localhost ~]$ sudo yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-`rpm -E %{rhel}`-x86_64/pgdg-redhat-repo-latest.noarch.rpm
Loaded plugins: fastestmirror, langpacks
pgdg-redhat-repo-latest.noarch.rpm                                                                                                                                                      | 8.6 kB  00:00:00     
Examining /var/tmp/yum-root-BkM7cQ/pgdg-redhat-repo-latest.noarch.rpm: pgdg-redhat-repo-42.0-32.noarch
Marking /var/tmp/yum-root-BkM7cQ/pgdg-redhat-repo-latest.noarch.rpm to be installed
Resolving Dependencies
--> Running transaction check
---> Package pgdg-redhat-repo.noarch 0:42.0-32 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

===============================================================================================================================================================================================================
 Package                                            Arch                                     Version                                   Repository                                                         Size
===============================================================================================================================================================================================================
Installing:
 pgdg-redhat-repo                                   noarch                                   42.0-32                                   /pgdg-redhat-repo-latest.noarch                                    13 k

Transaction Summary
===============================================================================================================================================================================================================
Install  1 Package

Total size: 13 k
Installed size: 13 k
Is this ok [y/d/N]: y
Downloading packages:
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : pgdg-redhat-repo-42.0-32.noarch                                                                                                                                                             1/1 
  Verifying  : pgdg-redhat-repo-42.0-32.noarch                                                                                                                                                             1/1 

Installed:
  pgdg-redhat-repo.noarch 0:42.0-32                                                                                                                                                                            

Complete!

```
</details>  

Ставим
```
# если удалось поставить репозиторий
sudo yum -y install postgresql15 postgresql15-server postgresql15-contrib postgresql15-libs

# если не удалось
sudo yum -y install postgresql postgresql-server postgresql-contrib postgresql-libs

# домашняя директория postgres
less /etc/passwd
postgres:x:26:26:PostgreSQL Server:/var/lib/pgsql:/bin/bash

# где лежит инициализатор
which postgresql-setup
/bin/postgresql-setup

# должен запускаться автоматом без указания пути
echo $PATH
/usr/lib64/qt-3.3/bin:/usr/local/bin:/bin:/usr/bin:/usr/local/sbin:/usr/sbin

# так и есть
postgresql-setup initdb
Initializing database ... OK

# настройка автозапуска сервиса
systemctl enable postgresql --now
Created symlink from /etc/systemd/system/multi-user.target.wants/postgresql.service to /usr/lib/systemd/system/postgresql.service.

# задаем пароль postgres
sudo passwd postgres
Изменяется пароль пользователя postgres.

# заходим под postgres
su - postgres
Пароль: 
Последний вход в систему:Ср окт  4 11:41:14 MSK 2023на pts/1
Последняя неудачная попытка входа в систему:Ср окт  4 12:00:20 MSK 2023на pts/4
Число неудачных попыток со времени последнего входа: 2.

# проверяем домашнюю директорию (то что смотрели выше)
-bash-4.2$ pwd
/var/lib/pgsql

# подключаемся
-bash-4.2$ psql
psql (9.2.24)
Введите "help", чтобы получить справку.

# тестируемся
postgres=# \c                                                                                                                                                                          
Вы подключены к базе данных "postgres" как пользователь "postgres".

postgres=# \dt *
pg_catalog | pg_aggregate | таблица | postgres
pg_catalog | pg_am | таблица | postgres

# выходим
postgres-# \q 
-bash-4.2$

```

## Ubuntu
> [Установка](https://lumpics.ru/how-install-ubuntu-on-virtualbox-virtual-machine/)

### Прописка прокси
```
sudo nano /etc/yum.conf
proxy=http://proxyserver:port

export http_proxy="proxy.stc:3128"
export https_proxy="proxy.stc:3128"
export no_proxy="localhost,127.0.0.1"

echo "keepcache=1" >> /etc/yum.conf && \
echo "http_proxy=http://proxy.ad.speechpro.com:3128" >> /etc/yum.conf && \
echo "https_proxy=http://proxy.ad.speechpro.com:3128" >> /etc/yum.conf && \
echo "no_proxy=.stc,.speechpro.com,.ad.speechpro.com" >> /etc/yum.conf
```
### Дополнительные репозитории
Если требуются и доступны, то
```
echo -e "\
[rpm_fusion_free] \n\
name=rpm_fusion_free \n\
baseurl=https://mirror.yandex.ru/fedora/rpmfusion/free/el/updates/7/x86_64/ \n\
enabled=1 \n\
gpgcheck=0 \n\
" > /etc/yum.repos.d/rpm_fusion_free.repo

echo -e "\
[rpm_fusion_nonfree] \n\
name=rpm_fusion_nonfree \n\
baseurl=https://mirror.yandex.ru/fedora/rpmfusion/nonfree/el/updates/7/x86_64/ \n\
enabled=1 \n\
gpgcheck=0 \n\
" > /etc/yum.repos.d/rpm_fusion_nonfree.repo

yum install createrepo modulemd-tools -y
```

### [Ставим](https://github.com/AV-ghub/PostgreSQL-Cloud-Solutions/blob/main/Practice/OTUS/PGCS/lesson_006%20patroni.md#%D1%81%D1%82%D0%B0%D0%B2%D0%B8%D0%BC)
Все репозитории [лежат тут](https://download.postgresql.org/pub/repos)
В частности для Ubunta процесс выглядит [следующим образом](https://download.postgresql.org/pub/repos/apt/README)
> PostgreSQL for Debian and Ubuntu Apt Repository
> ===============================================
>
> This repository hosts PostgreSQL server and extension module packages, as well
> as some client applications.
>
> To use the repository, do the following:
> Create /etc/apt/sources.list.d/pgdg.list. The distributions are called
> codename-pgdg. In the example, replace "jessie" with the actual distribution
> you are using:
>
>   deb http://apt.postgresql.org/pub/repos/apt/ jessie-pgdg main
>
> (You may determine the codename of your distribution by running lsb_release -c.)
>
> Import the repository key from https://www.postgresql.org/media/keys/ACCC4CF8.asc,
> update the package lists, and start installing packages:
>
>   wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
>   sudo apt-get update
>   sudo apt-get install postgresql-9.5 pgadmin3
>
> More information:
> * https://wiki.postgresql.org/wiki/Apt
> * https://wiki.postgresql.org/wiki/Apt/FAQ

Далее [возвращаемся сюда](https://github.com/AV-ghub/PostgreSQL-Cloud-Solutions/blob/main/PostgreSQL/Admin/001%20Installation.md#%D1%83%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0) и смотрим общую часть.







