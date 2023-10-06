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
https://download.postgresql.org/pub/repos/yum/reporpms/EL-`rpm -E %{rhel}`-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```
Загружаем ([не работает](https://postgrespro.ru/list/thread-id/2529687))
```
sudo yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-`rpm -E %{rhel}`-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-42.0-32.noarch.rpm
```
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
Введите "help", чтобы получить справку.                                                                                                                                                 postgres=# \c                                                                                                                                                                          
Вы подключены к базе данных "postgres" как пользователь "postgres".                                                                                                                     postgres-# \q 
-bash-4.2$


```


















