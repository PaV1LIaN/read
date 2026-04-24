Да, значит у тебя таблица сайтов называется не sites, а:

sitebuilder.site

А таблица прав, скорее всего:

sitebuilder.access

То есть предыдущий SQL надо адаптировать под твою структуру.

Выполни вот это

ALTER TABLE sitebuilder.site
ADD COLUMN IF NOT EXISTS bitrix_group_id integer DEFAULT NULL;

ALTER TABLE sitebuilder.site
ADD COLUMN IF NOT EXISTS bitrix_group_created_by integer DEFAULT NULL;

ALTER TABLE sitebuilder.site
ADD COLUMN IF NOT EXISTS bitrix_group_created_at timestamp DEFAULT NULL;

CREATE UNIQUE INDEX IF NOT EXISTS uq_sitebuilder_site_bitrix_group_id
ON sitebuilder.site(bitrix_group_id)
WHERE bitrix_group_id IS NOT NULL;

Почему так

У тебя сейчас есть две группы таблиц:

access
block
layout
menu
page
site

и отдельно:

sitebuilder_block
sitebuilder_page
sitebuilder_site
...

Судя по твоим прошлым ошибкам и работе API, основные рабочие таблицы сейчас, скорее всего, именно короткие:

sitebuilder.site
sitebuilder.page
sitebuilder.block
sitebuilder.access

Поэтому поле bitrix_group_id добавляем именно в:

sitebuilder.site

Проверь после выполнения

SELECT
    column_name,
    data_type
FROM information_schema.columns
WHERE table_schema = 'sitebuilder'
  AND table_name = 'site'
ORDER BY ordinal_position;

Там должны появиться:

bitrix_group_id
bitrix_group_created_by
bitrix_group_created_at

Важный момент для кода

В дальнейшем в site.php обновление должно быть не такое:

UPDATE sitebuilder.sites

а такое:

UPDATE sitebuilder.site

То есть этот кусок будет таким:

sb_db_execute("
    UPDATE sitebuilder.site
    SET
        bitrix_group_id = :bitrix_group_id,
        bitrix_group_created_by = :created_by,
        bitrix_group_created_at = now()
    WHERE id = :site_id
", [
    ':bitrix_group_id' => $bitrixGroupId,
    ':created_by' => (int)$USER->GetID(),
    ':site_id' => $siteId,
]);

Следующим шагом лучше пришли мне текущий файл:

/local/sitebuilder/api/handlers/site.php

Я вставлю туда создание группы аккуратно под твою реальную структуру таблиц.