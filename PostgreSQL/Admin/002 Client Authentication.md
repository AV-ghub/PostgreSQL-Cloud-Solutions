## [Аутентификация после установки](https://postgrespro.ru/docs/postgresql/16/client-authentication)
> [Исходная проблема](https://github.com/AV-ghub/PostgreSQL-Cloud-Solutions/blob/main/Practice/OTUS/PGCS/lesson_003%20disks%20mount.md#postgres)
>
> [Оно же](https://stackoverflow.com/questions/69676009/psql-error-connection-to-server-on-socket-var-run-postgresql-s-pgsql-5432)
>
> [Документация](https://postgrespro.ru/docs/postgresql/16/client-authentication)


**Аутентификация** это процесс идентификации клиента сервером базы данных, а также определение того, может ли клиентское приложение (или пользователь запустивший приложение) подключиться с указанным именем пользователя.

PostgreSQL предлагает несколько различных методов аутентификации клиентов. **Метод аутентификации** конкретного клиентского соединения может основываться на адресе компьютера клиента, имени базы данных, имени пользователя.

Сервер, принимающий **удалённые подключения**, может иметь большое количество пользователей базы данных, у которых нет учётной записи в ОС. В таких случаях не требуется соответствие между именами пользователей базы данных и именами пользователей операционной системы.

## [Файл pg_hba.conf](https://postgrespro.ru/docs/postgresql/16/auth-pg-hba-conf)
Расположен **в каталоге с данными** кластера базы данных.

Запись состоит из нескольких полей, разделённых пробелами и/или табуляциями. 
Поля могут содержать пробелы, если содержимое этих полей заключено в кавычки.

Если выбрана запись и аутентификация не прошла, последующие записи не рассматриваются.

Каждая запись может быть директивой включения или записью аутентификации. **Директивы включения** указывают, что нужно обработать дополнительные файлы с записями. 
Для формы **include_dir** будут включены все файлы, не начинающиеся с . и заканчивающиеся на .conf.

> Удалённое соединение по TCP/IP невозможно, если сервер запущен без определения соответствующих значений для параметра конфигурации listen_addresses, поскольку по умолчанию система принимает подключения по TCP/IP только для локального адреса замыкания localhost.
































