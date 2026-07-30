Да. Сначала убедимся, что новая нода действительно мастер.

В pgAdmin на новой ноде выполни:

SELECT
    inet_server_addr()::text AS server_ip,
    current_user AS database_user,
    pg_is_in_recovery() AS is_replica,
    current_setting('transaction_read_only') AS transaction_read_only;

Нужно получить:

is_replica = false
transaction_read_only = off

После этого не запускай миграцию сразу. Сначала выполни диагностический файл:

Скачать диагностику миграции этапа 5

В pgAdmin:

ProjectsDB → Tools → Query Tool → Open File → Execute

Скрипт завершится ROLLBACK, поэтому базу не изменит.

Возможные результаты

Если во всех строках:

ok = true

тогда снова открывай:

/local/sitebuilder/tools/apply_stage13_migration.php

и запускай миграции один раз.

Если появится строка:

ok = false

пришли значения:

step_name
sqlstate
error_message

Если диагностический SQL снова сообщит про режим «только чтение», значит выбрана не мастер-нода.