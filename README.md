Значит компонент уже рендерится, но DiskPermissionService возвращает canView = false.

Нужно проверить 4 вещи.


---

1. Твой пользователь реально авторизован

В test.php временно выведи:

<?php
global $USER;
echo '<pre>';
echo 'USER_ID=' . (int)$USER->GetID() . PHP_EOL;
echo 'AUTHORIZED=' . ($USER->IsAuthorized() ? 'Y' : 'N') . PHP_EOL;
echo '</pre>';
?>

Если USER_ID=0, проблема в авторизации.


---

2. В таблице sitebuilder_site_user_access есть роль именно для этого site_id и этого user_id

Проверь SQL:

SELECT *
FROM sitebuilder_site_user_access
WHERE site_id = 1
  AND user_id = 1;

Подставь свои реальные site_id и user_id.

Должна быть строка с role_code, например:

site_admin

site_editor

site_user

site_viewer


Если строки нет, компонент и покажет “нет доступа”.


---

3. В test.php совпадают siteId/pageId/blockId с БД

Проверь, что в test.php ты указал именно те значения, которые реально связаны между собой:

$siteId = 1;
$pageId = 1;
$blockId = 1;

И в БД должно быть так:

SELECT *
FROM sitebuilder_block
WHERE id = 1;

У блока должно быть:

site_id = 1

page_id = 1

type = 'disk'

is_active = 1



---

4. Сам SiteAccessRepository вообще что-то возвращает

Самая частая причина именно здесь.

В test.php временно добавь:

<?php
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/components/disk/bootstrap.php';

global $USER;

$userId = (int)$USER->GetID();
$siteId = 1;

echo '<pre>';
echo 'CURRENT USER: ' . $userId . PHP_EOL;
echo 'ROLE: ';
var_dump(SiteAccessRepository::getUserRole($siteId, $userId));
echo '</pre>';
?>

Если там NULL, значит проблема точно в таблице sitebuilder_site_user_access или в данных.


---

Быстрый способ проверить, что проблема именно в роли

Временно в DiskPermissionService.php замени метод resolveRolePermissions() на такой:

protected static function resolveRolePermissions(DiskContext $context): array
{
    return [
        'canView' => true,
        'canUpload' => true,
        'canCreateFolder' => true,
        'canRename' => true,
        'canDelete' => true,
        'canDownload' => true,
        'canManageAccess' => true,
        'canEditSettings' => true,
    ];
}

Если после этого блок откроется — проблема точно в роли/доступе, а не в Disk или UI.


---

Что я бы сделал прямо сейчас

Вариант 1 — правильный

Добавить себе запись в sitebuilder_site_user_access.

Например:

INSERT INTO sitebuilder_site_user_access (site_id, user_id, role_code, created_at, updated_at)
VALUES (1, 1, 'site_admin', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);

Если запись уже есть, но роль другая:

UPDATE sitebuilder_site_user_access
SET role_code = 'site_admin',
    updated_at = CURRENT_TIMESTAMP
WHERE site_id = 1
  AND user_id = 1;


---

Вариант 2 — для диагностики

Временно открыть доступ всем через DiskPermissionService.


---

Самая вероятная причина

С вероятностью почти наверняка одно из двух:

в sitebuilder_site_user_access нет записи для твоего пользователя;

user_id в БД не совпадает с тем, кто реально авторизован в Bitrix.



---

Чтобы быстро локализовать

В test.php вставь вот это целиком над рендером компонента:

<?php
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/components/disk/bootstrap.php';

global $USER;

$userId = (int)$USER->GetID();
$siteId = 1;
$pageId = 1;
$blockId = 1;

echo '<pre>';
echo 'USER_ID=' . $userId . PHP_EOL;
echo 'AUTHORIZED=' . ($USER->IsAuthorized() ? 'Y' : 'N') . PHP_EOL;
echo 'ROLE=';
var_dump(SiteAccessRepository::getUserRole($siteId, $userId));
echo 'BLOCK=';
var_dump(BlockRepository::getDiskBlockByContext($siteId, $pageId, $blockId));
echo '</pre>';
?>

Пришли вывод этого блока — и я сразу скажу, где именно ломается.