Обработка заняла 2m 37s

Отлично: этот вывод подтверждает, что миграция этапа 5 полностью откатилась — столбцы version не появились, данные не испорчены. Восстанавливать backup не нужно.

Наиболее вероятная причина: пользователь PostgreSQL, которым SiteBuilder подключается через pg_master.php, не является владельцем старых таблиц page и block. Создавать новые таблицы он может, поэтому этап 13 установился, но ALTER TABLE PostgreSQL разрешает только владельцу.

1. Посмотри пользователя подключения SiteBuilder

Открой:

/local/php_interface/lib/pg_master.php

Найди логин PostgreSQL, например:

user=sitebuilder_user

Пароль присылать не нужно. Нужен только логин.

2. В pgAdmin выполни запрос

Сначала без изменений в базе:

SELECT
    current_user AS pgadmin_user,
    c.relname AS table_name,
    pg_get_userbyid(c.relowner) AS table_owner
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE n.nspname = 'sitebuilder'
  AND c.relname IN ('site', 'page', 'block', 'menu', 'layout', 'access')
  AND c.relkind IN ('r', 'p')
ORDER BY c.relname;

Результат будет примерно таким:

pgadmin_user | table_name | table_owner
postgres     | page       | postgres
postgres     | block      | postgres

Сравни table_owner с пользователем из pg_master.php.

3. Дополнительная проверка

Подставь логин из pg_master.php вместо SITEBUILDER_DB_USER:

SELECT
    c.relname AS table_name,
    pg_get_userbyid(c.relowner) AS table_owner,
    'SITEBUILDER_DB_USER' AS application_user,
    (
        pg_get_userbyid(c.relowner) = 'SITEBUILDER_DB_USER'
        OR pg_has_role(
            'SITEBUILDER_DB_USER',
            pg_get_userbyid(c.relowner),
            'MEMBER'
        )
    ) AS can_alter_table
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE n.nspname = 'sitebuilder'
  AND c.relname IN ('site', 'page', 'block', 'menu', 'layout', 'access')
  AND c.relkind IN ('r', 'p')
ORDER BY c.relname;

Если в can_alter_table будет false, причина найдена.

Пока повторно миграцию не запускай. Пришли логин из pg_master.php и результат первого запроса — после этого дам точный безопасный SQL для исправления владельцев всех таблиц и sequences схемы sitebuilder.