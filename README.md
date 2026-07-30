Обработка заняла 3m 30s

Сейчас видно следующее:

этап 13 применился — реестр миграций создан;

этап 5 завершился ошибкой;

этапы 6–12 поэтому не запускались;

изменения этапа 5 выполнялись в транзакции, поэтому при ошибке должны были откатиться.


Надпись «Fingerprint: не найден» не означает, что SQL-файл потерян. Она означает, что ожидаемые таблицы и столбцы этой миграции ещё не появились в базе.

Пока не нажимай кнопку повторно и не работай в SiteBuilder.

1. Посмотреть, какая миграция упала

В pgAdmin открой:

projectsDB → Tools → Query Tool

Выполни:

SELECT
    id,
    mode,
    status,
    applied_count,
    baseline_count,
    skipped_count,
    failed_migration_key,
    error_code,
    started_at,
    finished_at
FROM sitebuilder.deployment_run
ORDER BY id DESC
LIMIT 5;

Скорее всего, будет:

failed_migration_key:
20260729_001_entity_versions

error_code:
MIGRATION_FAILED

2. Проверить текущую структуру таблиц

Затем выполни:

SELECT
    table_name,
    ordinal_position,
    column_name,
    data_type,
    is_nullable
FROM information_schema.columns
WHERE table_schema = 'sitebuilder'
  AND table_name IN ('page', 'block')
ORDER BY table_name, ordinal_position;

Пришли фотографию или скопированный результат этого запроса.

3. Найти точную ошибку на сервере

При наличии SSH выполни:

sudo grep -R \
  "SiteBuilder migration 20260729_001_entity_versions failed" \
  /var/log /home/bitrix 2>/dev/null | tail -30

Если ничего не найдено:

sudo journalctl --since "30 minutes ago" \
  | grep -i "SiteBuilder migration"

Нужна строка примерно такого вида:

SiteBuilder migration 20260729_001_entity_versions failed: ...

Именно текст после failed: покажет настоящую причину PostgreSQL.

После исправления запускать нужно будет уже страницу:

/local/sitebuilder/deployment.php

и кнопку «Применить ожидающие миграции», а не восстанавливать базу из backup.