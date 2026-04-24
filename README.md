Отлично, группа Битрикс24 создалась и привязалась к сайту:

"bitrixGroupId": 2,
"bitrixGroupError": ""

То есть этап 1 работает.

Проверь в БД:

SELECT
    id,
    name,
    slug,
    bitrix_group_id,
    bitrix_group_created_by,
    bitrix_group_created_at
FROM sitebuilder.site
WHERE id = 11;

Должно быть:

bitrix_group_id = 2

И в Битрикс24 в разделе Группы и проекты должна появиться группа:

SiteBuilder: 321

или похожее название.

Что делаем дальше

Следующий правильный этап — связать участников группы Битрикс24 с правами SiteBuilder.

Логика такая:

Владелец группы Битрикс24   → OWNER в sitebuilder.access
Модератор группы            → EDITOR
Участник группы             → VIEWER

То есть sitebuilder.access остается основной таблицей прав конструктора, но мы сможем синхронизировать её с участниками группы.

Следующим шагом нужно сделать сервис:

/local/sitebuilder/lib/SiteAccessSyncService.php

Он будет брать bitrixGroupId сайта, читать участников группы Битрикс24 и обновлять таблицу sitebuilder.access.

Сначала лучше проверить, что группа реально создана в Битрикс24 и bitrix_group_id записался в таблицу. Потом перейдем к синхронизации прав.