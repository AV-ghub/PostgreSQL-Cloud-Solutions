### [Src](https://github.com/AV-ghub/PostgreSQL-Cloud-Solutions/blob/main/PostgreSQL/Admin/001%20Installation.md)

### first of all install repo (from admin)
```
sudo yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-`rpm -E %{rhel}`-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

### install binaries (from admin)
```
sudo yum -y install postgresql13 postgresql13-server postgresql13-contrib postgresql13-libs
sudo yum -y install postgresql14 postgresql14-server postgresql14-contrib postgresql14-libs
sudo yum -y install postgresql15 postgresql15-server postgresql15-contrib postgresql15-libs
sudo yum -y install postgresql16 postgresql16-server postgresql16-contrib postgresql16-libs
sudo yum -y install postgresql17 postgresql17-server postgresql17-contrib postgresql17-libs
```

> NOTE
> if we install 'em in strong sequence by version increasing
> we'll get the last psql (ver 17) in our bin catalog
> so we'll be able to start psql for every instance (13-17) by only appropriate port mentioning

### make initial db catalog (from admin, not root or postgres!)
```
sudo postgresql-13-setup initdb
sudo postgresql-14-setup initdb
sudo postgresql-15-setup initdb
sudo postgresql-16-setup initdb
sudo postgresql-17-setup initdb
```

### setup host conection and different ports for ability to simultaniously starting
```
sudo nano postgresql.conf (from admin)
```
> don't forget to uncomment rows!!
```
listen_addresses = '*'         # what IP address(es) to listen on;
port = 543(3,4,5,6,7)
```
```
sudo nano pg_hba.conf (from admin)
```
```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             all             0.0.0.0/0               scram-sha-256
# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     trust
host    replication     all             127.0.0.1/32            scram-sha-256
host    replication     all             ::1/128                 scram-sha-256
```

### for remote connection
```
sudo passwd postgres
```
> and also obligatory in psql later, when it will be possible to start it
```
postgres=# alter user postgres with password 'postgres';
```

### enable only which is demanded (from admin)
```
sudo systemctl enable postgresql-13 --now
sudo systemctl enable postgresql-14 --now
sudo systemctl enable postgresql-15 --now
sudo systemctl enable postgresql-16 --now
sudo systemctl enable postgresql-17 --now
```

### disable 
```
sudo systemctl disable postgresql-13 --now
sudo systemctl disable postgresql-14 --now
sudo systemctl disable postgresql-15 --now
sudo systemctl disable postgresql-16 --now
sudo systemctl disable postgresql-17 --now
```

### restart
```
sudo systemctl restart postgresql-13 --now
sudo systemctl restart postgresql-14 --now
sudo systemctl restart postgresql-15 --now
sudo systemctl restart postgresql-16 --now
sudo systemctl restart postgresql-17 --now
```

### start only which is demanded (from admin)
```
systemctl start postgresql-13 --now
/usr/pgsql-13/bin/pg_ctl -D /var/lib/pgsql/13/data/ -l файл_журнала start

systemctl start postgresql-14 --now
/usr/pgsql-14/bin/pg_ctl -D /var/lib/pgsql/14/data/ -l файл_журнала start

systemctl start postgresql-15 --now
/usr/pgsql-15/bin/pg_ctl -D /var/lib/pgsql/15/data/ -l файл_журнала start

systemctl start postgresql-16 --now
/usr/pgsql-16/bin/pg_ctl -D /var/lib/pgsql/16/data/ -l файл_журнала start

systemctl start postgresql-17 --now
/usr/pgsql-17/bin/pg_ctl -D /var/lib/pgsql/17/data/ -l файл_журнала start
```

### check (from admin)
```
systemctl status postgresql-13 --now
systemctl status postgresql-14 --now
systemctl status postgresql-15 --now
systemctl status postgresql-16 --now
systemctl status postgresql-17 --now
```

### check which is running (from admin)
```
systemctl status postgresql*
```

### check connection

#### local
```
psql -p 543(3-7)
```
#### remote
```
psql -h localhost -p 543(3-7)
```
