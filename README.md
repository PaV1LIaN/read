Обработка заняла 3m 3s

Понял. Серверные команды не нужны.

Эта ошибка означает, что подключение pgAdmin сейчас работает в режиме только чтения. Диагностический скрипт даже не дошёл до проверки миграции — он остановился на создании временной таблицы.

В pgAdmin выполни только этот безопасный запрос:

SELECT
    current_database() AS database_name,
    current_user AS database_user,
    inet_server_addr()::text AS server_ip,
    inet_server_port() AS server_port,
    pg_is_in_recovery() AS is_replica,
    current_setting('transaction_read_only') AS transaction_read_only,
    current_setting('default_transaction_read_only') AS default_transaction_read_only;

Пришли результат целиком.

Как понимать результат

Если будет:

is_replica = true
transaction_read_only = on

значит pgAdmin подключён к реплике PostgreSQL. На ней миграции выполнить невозможно — нужен адрес основного сервера PostgreSQL.

Если будет:

is_replica = false
default_transaction_read_only = on

значит режим только чтения установлен для базы или пользователя bx_user. Это должен отключить администратор PostgreSQL.

Если будет:

is_replica = false
transaction_read_only = off

значит pgAdmin подключён не к тому серверу, что SiteBuilder, либо ошибка миграции вызвана другим SQL-запросом.

Также посмотри в:

/local/php_interface/lib/pg_master.php

и пришли без пароля только значения:

DB_HOST
DB_PORT
DB_NAME
DB_USER

Миграцию пока больше не запускай. Базу восстанавливать не нужно — неудачная попытка откатилась.