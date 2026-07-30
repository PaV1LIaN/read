Обработка заняла 4m 21s

Отлично: владелец таблиц совпадает с пользователем подключения bx_user. Значит, права владельца менять не нужно — предыдущая версия причины исключена.

Теперь проверим две наиболее вероятные причины: остаточная таблица от старой попытки или блокировка таблиц.

1. Проверь, осталась ли entity_revision

В pgAdmin выполни:

SELECT to_regclass('sitebuilder.entity_revision') AS entity_revision;

Если результат:

NULL

миграция этапа 5 откатилась полностью.

Если будет:

sitebuilder.entity_revision

выполни дополнительно:

SELECT
    ordinal_position,
    column_name,
    data_type,
    is_nullable
FROM information_schema.columns
WHERE table_schema = 'sitebuilder'
  AND table_name = 'entity_revision'
ORDER BY ordinal_position;

2. Проверь блокировки page и block

Закрой вкладку с главной страницей SiteBuilder, чтобы она не выполняла AJAX-запросы. Затем выполни:

SELECT
    a.pid,
    a.usename,
    a.state,
    a.xact_start,
    a.wait_event_type,
    a.wait_event,
    c.relname AS table_name,
    l.mode,
    l.granted,
    LEFT(a.query, 200) AS query
FROM pg_locks l
JOIN pg_class c
    ON c.oid = l.relation
JOIN pg_namespace n
    ON n.oid = c.relnamespace
JOIN pg_stat_activity a
    ON a.pid = l.pid
WHERE n.nspname = 'sitebuilder'
  AND c.relname IN ('page', 'block')
  AND a.pid <> pg_backend_pid()
ORDER BY a.xact_start NULLS LAST;

Если запрос не вернул строк либо нет старых сессий idle in transaction, можно сделать одну повторную попытку:

/local/sitebuilder/tools/apply_stage13_migration.php

Перед повтором:

закрой остальные страницы SiteBuilder;

worker пока не запускай;

не открывай редактор;

оставь только страницу миграции.


Если снова появится MIGRATION_FAILED, больше не повторяй. Тогда нужна точная ошибка PHP. На сервере выполни:

sudo grep -R \
  "SiteBuilder migration 20260729_001_entity_versions failed" \
  /var/log /home/bitrix 2>/dev/null | tail -20

Нужна строка после failed: — она покажет конкретную команду PostgreSQL, на которой остановилась миграция.