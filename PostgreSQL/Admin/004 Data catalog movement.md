### [Настройка дисков для Постгреса](https://github.com/AV-ghub/PostgreSQL-Cloud-Solutions/blob/main/Practice/OTUS/PGCS/lesson_003%20disks%20mount.md#%D0%BD%D0%B0%D1%81%D1%82%D1%80%D0%BE%D0%B9%D0%BA%D0%B0-%D0%B4%D0%B8%D1%81%D0%BA%D0%BE%D0%B2-%D0%B4%D0%BB%D1%8F-%D0%BF%D0%BE%D1%81%D1%82%D0%B3%D1%80%D0%B5%D1%81%D0%B0)

### [Прогрессивный вариант перемещения каталога (How to change PostgreSQL's data directory on Linux)](https://fitodic.github.io/how-to-change-postgresql-data-directory-on-linux)
#### Остановка и копирование
```
$ systemctl stop postgresql-13 --now
$ systemctl status postgresql*
$ sudo mv /var/lib/pgsql/13/data /var/lib/pgsql/13/data_bak
$ sudo cp -a /var/lib/pgsql/13/data /home/data/pgsql/13/data

cd /lib/systemd/system
sudo nano postgresql-13.service
```
#### Смена окружения
```
#Environment=PGDATA=/var/lib/pgsql/13/data/
Environment=PGDATA=/home/data/pgsql/13/data/
```
#### Перезапуск
```
$ sudo systemctl daemon-reload
```
#### Старт сервиса
```
$ systemctl start postgresql-13 --now
$ systemctl status postgresql*
```
#### Зачистка
```
sudo rm -Rf /var/lib/pgsql/13/data_bak
```














