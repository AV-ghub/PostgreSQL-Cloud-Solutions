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
Загружаем
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

Ставим следующую вещь !!!!
> https://github.com/fdupoux/fsarchiver/issues/62#issuecomment-396885830

```
sudo yum install epel-release
```

Ставим PostgreSQL
```
# если удалось поставить репозиторий
sudo yum -y install postgresql15 postgresql15-server postgresql15-contrib postgresql15-libs
```

<details>
 <summary>Output</summary>

```
admin@localhost ~]$ sudo yum -y install postgresql15 postgresql15-server postgresql15-contrib postgresql15-libs
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * base: mirror.docker.ru
 * epel: epel.mirror.serveriai.lt
 * extras: mirror.docker.ru
 * updates: mirror.corbina.net
Package postgresql15-15.4-1PGDG.rhel7.x86_64 already installed and latest version
Package postgresql15-libs-15.4-1PGDG.rhel7.x86_64 already installed and latest version
Resolving Dependencies
--> Running transaction check
---> Package postgresql15-contrib.x86_64 0:15.4-1PGDG.rhel7 will be installed
--> Processing Dependency: libpython3.6m.so.1.0()(64bit) for package: postgresql15-contrib-15.4-1PGDG.rhel7.x86_64
---> Package postgresql15-server.x86_64 0:15.4-1PGDG.rhel7 will be installed
--> Running transaction check
---> Package python3-libs.x86_64 0:3.6.8-19.el7_9 will be installed
--> Processing Dependency: python(abi) = 3.6 for package: python3-libs-3.6.8-19.el7_9.x86_64
--> Running transaction check
---> Package python3.x86_64 0:3.6.8-19.el7_9 will be installed
--> Processing Dependency: python3-setuptools for package: python3-3.6.8-19.el7_9.x86_64
--> Processing Dependency: python3-pip for package: python3-3.6.8-19.el7_9.x86_64
--> Running transaction check
---> Package python3-pip.noarch 0:9.0.3-8.el7 will be installed
---> Package python3-setuptools.noarch 0:39.2.0-10.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

===============================================================================================================================================================================================================
 Package                                                  Arch                                       Version                                                 Repository                                   Size
===============================================================================================================================================================================================================
Installing:
 postgresql15-contrib                                     x86_64                                     15.4-1PGDG.rhel7                                        pgdg15                                      710 k
 postgresql15-server                                      x86_64                                     15.4-1PGDG.rhel7                                        pgdg15                                      5.7 M
Installing for dependencies:
 python3                                                  x86_64                                     3.6.8-19.el7_9                                          updates                                      70 k
 python3-libs                                             x86_64                                     3.6.8-19.el7_9                                          updates                                     6.9 M
 python3-pip                                              noarch                                     9.0.3-8.el7                                             base                                        1.6 M
 python3-setuptools                                       noarch                                     39.2.0-10.el7                                           base                                        629 k

Transaction Summary
===============================================================================================================================================================================================================
Install  2 Packages (+4 Dependent packages)

Total download size: 16 M
Installed size: 73 M
Downloading packages:
(1/6): python3-3.6.8-19.el7_9.x86_64.rpm                                                                                                                                                |  70 kB  00:00:00     
(2/6): python3-pip-9.0.3-8.el7.noarch.rpm                                                                                                                                               | 1.6 MB  00:00:00     
(3/6): postgresql15-contrib-15.4-1PGDG.rhel7.x86_64.rpm                                                                                                                                 | 710 kB  00:00:01     
(4/6): python3-libs-3.6.8-19.el7_9.x86_64.rpm                                                                                                                                           | 6.9 MB  00:00:01     
(5/6): postgresql15-server-15.4-1PGDG.rhel7.x86_64.rpm                                                                                                                                  | 5.7 MB  00:00:01     
(6/6): python3-setuptools-39.2.0-10.el7.noarch.rpm                                                                                                                                      | 629 kB  00:00:02     
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                                                                          6.2 MB/s |  16 MB  00:00:02     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : python3-setuptools-39.2.0-10.el7.noarch                                                                                                                                                     1/6 
  Installing : python3-pip-9.0.3-8.el7.noarch                                                                                                                                                              2/6 
  Installing : python3-libs-3.6.8-19.el7_9.x86_64                                                                                                                                                          3/6 
  Installing : python3-3.6.8-19.el7_9.x86_64                                                                                                                                                               4/6 
  Installing : postgresql15-server-15.4-1PGDG.rhel7.x86_64                                                                                                                                                 5/6 
  Installing : postgresql15-contrib-15.4-1PGDG.rhel7.x86_64                                                                                                                                                6/6 
  Verifying  : python3-3.6.8-19.el7_9.x86_64                                                                                                                                                               1/6 
  Verifying  : postgresql15-server-15.4-1PGDG.rhel7.x86_64                                                                                                                                                 2/6 
  Verifying  : postgresql15-contrib-15.4-1PGDG.rhel7.x86_64                                                                                                                                                3/6 
  Verifying  : python3-setuptools-39.2.0-10.el7.noarch                                                                                                                                                     4/6 
  Verifying  : python3-pip-9.0.3-8.el7.noarch                                                                                                                                                              5/6 
  Verifying  : python3-libs-3.6.8-19.el7_9.x86_64                                                                                                                                                          6/6 

Installed:
  postgresql15-contrib.x86_64 0:15.4-1PGDG.rhel7                                                         postgresql15-server.x86_64 0:15.4-1PGDG.rhel7                                                        

Dependency Installed:
  python3.x86_64 0:3.6.8-19.el7_9                python3-libs.x86_64 0:3.6.8-19.el7_9                python3-pip.noarch 0:9.0.3-8.el7                python3-setuptools.noarch 0:39.2.0-10.el7               

Complete!

```
</details>

```
# если не удалось
sudo yum -y install postgresql postgresql-server postgresql-contrib postgresql-libs

# домашняя директория postgres
less /etc/passwd
postgres:x:26:26:PostgreSQL Server:/var/lib/pgsql:/bin/bash

# где лежит инициализатор
which postgresql-setup
/bin/postgresql-setup

[admin@localhost bin]$ which postgresql-15-setup
/usr/bin/postgresql-15-setup

# должен запускаться автоматом без указания пути
echo $PATH
/usr/lib64/qt-3.3/bin:/usr/local/bin:/bin:/usr/bin:/usr/local/sbin:/usr/sbin

# так и есть
postgresql-setup initdb
Initializing database ... OK

[admin@localhost bin]$ postgresql-15-setup initdb
Initializing database ... mkdir: cannot create directory ‘/var/lib/pgsql’: Permission denied
failed, see /var/lib/pgsql/15/initdb.log

[admin@localhost bin]$ sudo su postgres
bash-4.2$ postgresql-15-setup initdb
Initializing database ... bash-4.2$

# нельзя это делать из-под root или postgres !!!!!!!!!
# правильно
[admin@localhost bin]$ sudo postgresql-15-setup initdb

# настройка автозапуска сервиса
systemctl enable postgresql --now
Created symlink from /etc/systemd/system/multi-user.target.wants/postgresql.service to /usr/lib/systemd/system/postgresql.service.

# нельзя это делать из-под root или postgres !!!!!!!!!
bash-4.2$ systemctl enable postgresql-15 --now
Created symlink from /etc/systemd/system/multi-user.target.wants/postgresql-15.service to /usr/lib/systemd/system/postgresql-15.service.
Job for postgresql-15.service failed because the control process exited with error code. See "systemctl status postgresql-15.service" and "journalctl -xe" for details.
bash-4.2$

# правильно
[admin@localhost bin]$ sudo systemctl enable postgresql-15 --now

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
```
# смотрим конфигурацию
:~$ pg_config
```
<details>
 <summary>Output</summary>
 
```
BINDIR = /usr/lib/postgresql/14/bin
DOCDIR = /usr/share/doc/postgresql-doc-14
HTMLDIR = /usr/share/doc/postgresql-doc-14
INCLUDEDIR = /usr/include/postgresql
PKGINCLUDEDIR = /usr/include/postgresql
INCLUDEDIR-SERVER = /usr/include/postgresql/14/server
LIBDIR = /usr/lib/x86_64-linux-gnu
PKGLIBDIR = /usr/lib/postgresql/14/lib
LOCALEDIR = /usr/share/locale
MANDIR = /usr/share/postgresql/14/man
SHAREDIR = /usr/share/postgresql/14
SYSCONFDIR = /etc/postgresql-common
PGXS = /usr/lib/postgresql/14/lib/pgxs/src/makefiles/pgxs.mk
CONFIGURE =  '--build=x86_64-linux-gnu' '--prefix=/usr' '--includedir=${prefix}/include' '--mandir=${prefix}/share/man' '--infodir=${prefix}/share/info' '--sysconfdir=/etc' '--localstatedir=/var' '--disable-option-checking' '--disable-silent-rules' '--libdir=${prefix}/lib/x86_64-linux-gnu' '--runstatedir=/run' '--disable-maintainer-mode' '--disable-dependency-tracking' '--with-tcl' '--with-perl' '--with-python' '--with-pam' '--with-openssl' '--with-libxml' '--with-libxslt' '--mandir=/usr/share/postgresql/14/man' '--docdir=/usr/share/doc/postgresql-doc-14' '--sysconfdir=/etc/postgresql-common' '--datarootdir=/usr/share/' '--datadir=/usr/share/postgresql/14' '--bindir=/usr/lib/postgresql/14/bin' '--libdir=/usr/lib/x86_64-linux-gnu/' '--libexecdir=/usr/lib/postgresql/' '--includedir=/usr/include/postgresql/' '--with-extra-version= (Ubuntu 14.9-1.pgdg22.04+1)' '--enable-nls' '--enable-thread-safety' '--enable-debug' '--enable-dtrace' '--disable-rpath' '--with-uuid=e2fs' '--with-gnu-ld' '--with-gssapi' '--with-ldap' '--with-pgport=5432' '--with-system-tzdata=/usr/share/zoneinfo' 'AWK=mawk' 'MKDIR_P=/bin/mkdir -p' 'PROVE=/usr/bin/prove' 'PYTHON=/usr/bin/python3' 'TAR=/bin/tar' 'XSLTPROC=xsltproc --nonet' 'CFLAGS=-g -O2 -flto=auto -ffat-lto-objects -flto=auto -ffat-lto-objects -fstack-protector-strong -Wformat -Werror=format-security -fno-omit-frame-pointer' 'LDFLAGS=-Wl,-Bsymbolic-functions -flto=auto -ffat-lto-objects -flto=auto -Wl,-z,relro -Wl,-z,now' '--enable-tap-tests' '--with-icu' '--with-llvm' 'LLVM_CONFIG=/usr/bin/llvm-config-14' 'CLANG=/usr/bin/clang-14' '--with-lz4' '--with-systemd' '--with-selinux' 'build_alias=x86_64-linux-gnu' 'CPPFLAGS=-Wdate-time -D_FORTIFY_SOURCE=2' 'CXXFLAGS=-g -O2 -flto=auto -ffat-lto-objects -flto=auto -ffat-lto-objects -fstack-protector-strong -Wformat -Werror=format-security'
CC = gcc
CPPFLAGS = -Wdate-time -D_FORTIFY_SOURCE=2 -D_GNU_SOURCE -I/usr/include/libxml2
CFLAGS = -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Wendif-labels -Wmissing-format-attribute -Wimplicit-fallthrough=3 -Wcast-function-type -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-format-truncation -Wno-stringop-truncation -g -g -O2 -flto=auto -ffat-lto-objects -flto=auto -ffat-lto-objects -fstack-protector-strong -Wformat -Werror=format-security -fno-omit-frame-pointer
CFLAGS_SL = -fPIC
LDFLAGS = -Wl,-Bsymbolic-functions -flto=auto -ffat-lto-objects -flto=auto -Wl,-z,relro -Wl,-z,now -L/usr/lib/llvm-14/lib -Wl,--as-needed
LDFLAGS_EX = 
LDFLAGS_SL = 
LIBS = -lpgcommon -lpgport -lselinux -llz4 -lxslt -lxml2 -lpam -lssl -lcrypto -lgssapi_krb5 -lz -lreadline -lm 
VERSION = PostgreSQL 14.9 (Ubuntu 14.9-1.pgdg22.04+1)
```
</details>

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







