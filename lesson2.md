Установка и настройка PostgteSQL в контейнере Docker
========================

Создаем проект в ЯО: postgres2023-19720207.
Создаем каталог и сервис Compute Cloud.
-------------------------

Создаем виртуальную машину
-------------------------

    Идентификатор: epdkuglh8pcn3hf97gue
    Имя: otus-db-pg-vm-1
    Внутренний FQDN: otus-db-pg-vm-1.ru-central1.internal
    Зона доступности: ru-central1-b
    Сервисный аккаунт: otus

Параметры

    Платформа: Intel Ice Lake
    Гарантированная доля vCPU: 100%
    vCPU: 2
    RAM: 4 ГБ
    Объём дискового пространства: 20 ГБ

SSH ключ сгенерен с хостовой Win машины через PuTTY (pageant.exe) и внедрен в VM на этапе ее создания.
В PuTTY ключ цепляем здесь: 

    Connection/SSH/Auth/Credentials/Private key file for authentication
        
Не забываем сохраняться.
С PuTTY проброс происходит без проблем, цепляет ключ и идентифицирует.

Однако сессия дохнет через неск сек бездействия.
Решение тут https://www.thegeekdiary.com/how-to-configure-timeout-on-ssh-client-putty/

Работает.

Проверяем что нам поставили:

    cat /etc/os-release
    PRETTY_NAME="Ubuntu 22.04.2 LTS"

Обновляемся

    sudo apt update

Ставим 

    sudo apt install apt-transport-https

Ставим ключик докера

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

Задаемся целью качать только стабильные версии
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

И еще раз
sudo apt update

Ставим бесплатный докер
sudo apt install docker-ce

Проверяем что докер жив
sudo systemctl status docker

Версия
sudo docker -v
Docker version 23.0.5, build bc4487a

Чтобы не париться дальше с sudo кидаем себя в группу докера
sudo usermod -aG docker $USER

Перелогиниваемся
exit

И проверяем
otus@otus-db-pg-vm-1:~$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

Все ок. Можно начинать работать.

Делаем пробник
otus@otus-db-pg-vm-1:~$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
719385e32844: Pull complete
Digest: sha256:9eabfcf6034695c4f6208296be9090b0a3487e20fb6a5cb056525242621cf73d
Status: Downloaded newer image for hello-world:latest

Hello from Docker!

otus@otus-db-pg-vm-1:~$ docker images
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
hello-world   latest    9c7a54a9a43c   14 hours ago   13.3kB

Теперь postgres.
Как развернуть postgres читаем здесь https://habr.com/ru/articles/578744/

Пробуем простую установку
docker run --name maindb -e POSTGRES_PASSWORD=pg -d postgres
Нужная нам версия
docker run --name maindb -p 5432:5432 -e POSTGRES_PASSWORD=pg -d postgres:14

После удаления контейнера и попытки повторной установки на тот же порт вариант не прокатит
docker: Error response from daemon: driver failed programming external connectivity on endpoint Error starting userland proxy: listen tcp4 0.0.0.0:5432: bind: address already in use.
Решение тут https://stackoverflow.com/questions/38249434/docker-postgres-failed-to-bind-tcp-0-0-0-05432-address-already-in-use
sudo ss -lptn 'sport = :5432'
kill <pid>
То же про залипающий после манипуляций контейнер - удалаяем по id
docker rm eaaa4fa65abc705ade4f7b0061bf707f5a2b35738ac672f0e5ade08d33e70620

Запуститься можем изнутри
docker exec -it maindb bash
Далее обычно
psql -U postgres -d postgres
postgres=# select version();
                                                           version
PostgreSQL 14.7 (Debian 14.7-1.pgdg110+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 10.2.1-6) 10.2.1 20210110, 64-bit


И снаружи, напр через PgAdmin (public IP/5432/postgres/pg).

Далее делаем проброс PGDATA (проброс, как и с портами, читаем справа налево: что монтируем наружу <- что берем изнутри)
docker run --name maindb -p 5432:5432 -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=pg -e POSTGRES_DB=postgres -e PGDATA=/var/lib/postgresql/data/maindb -v "/var/lib/postgresql/data/maindb":/var/lib/postgresql/data/maindb -d postgres:14

Все, на этом основной инстанс жив и железно заперсистчен на основной машине.
Т.е. если грохнули контейнер и заново подняли с теми же буквами, то получим базы кластера как оставляли до удаления.

Проверяем это.
Создаем базу test, делаем табличку ttt, кидаем туда строчку по красоте.
Удаляем контейнер, пересоздаем - все на месте. Ок.

Теперь задача пробиться не просто снаружи с клиента a`la PgAdmin, а из другого контейнера.
Ессно контейнер должен либо уметь пускать psql, либо обладать соотв дровами в комплекте. 
Т.е. вариант только поставить либо оно же, либо контейнер с соотв фичей.
Поставим оно же.

Поставить клиентский контейнер не сложно - просто кидаем на другой порт:
docker run --name clientdb -p 5433:5432 -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=pg -e POSTGRES_DB=postgres -e PGDATA=/var/lib/postgresql/data/clientdb -v "/var/lib/postgresql/data/clientdb":/var/lib/postgresql/data/clientdb -d postgres:14

Пытаемся из него пробиться в maindb, и понимаем, что у нас проблема с хостом. Мы не понимаем кто здесь хост.
Читаем внимательно https://stackoverflow.com/questions/37694987/connecting-to-postgresql-in-a-docker-container-from-outside
и понимаем что хост - это 
So now you have mapped the port 5432 of your container to port 5432 of your server. -p <host_port>:<container_port>.
So now your postgres is accessible from your public-server-ip:5432

Ну и отсюда уже прямой ход до истины
otus@otus-db-pg-vm-1:~$ docker exec -it clientdb bash
root@f05d761c16d6:/# psql -U postgres -d postgres -p 5432 -h 158.160.20.140
Password for user postgres:
psql (14.7 (Debian 14.7-1.pgdg110+1))
Type "help" for help.

postgres=# \c
You are now connected to database "postgres" as user "postgres".
postgres=# \c test
You are now connected to database "test" as user "postgres".
test=# select * from ttt;
 id | val
----+-----
  1 | qq
(1 row)

test=#
