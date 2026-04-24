Отлично, синхронизация заработала.

Сейчас она увидела в группе Битрикс24 одного участника:

U1 → OWNER

и оставила права в sitebuilder.access без изменений:

created: 0
updated: 0
removed: 0
kept: 1

Следующий правильный шаг — поправить нормализацию сайта, чтобы site.get и site.list тоже возвращали:

"bitrixGroupId": 2

Сейчас это поле есть в БД, но не попадает в ответ API. Из-за этого мы были вынуждены дочитывать bitrix_group_id прямо в site.syncAccess.

Нужно найти файл, где формируется сайт из строки БД. Скорее всего это один из файлов:

/local/sitebuilder/lib/storage_db.php
/local/sitebuilder/lib/storage_db_extra.php
/local/sitebuilder/lib/storage.php

И там будет функция типа:

sb_normalize_site_record()

или чтение SELECT * FROM sitebuilder.site.

Пришли мне файл, где объявлены функции:

sb_read_sites()
sb_find_site()

Я поправлю его так, чтобы bitrix_group_id нормально мапился в bitrixGroupId, и после этого можно будет убрать временное дочитывание из site.syncAccess.