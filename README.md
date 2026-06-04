Да, значит script.js уже работает правильно, но API диска отдаёт только ID, например:

createdBy: 99

А нужно, чтобы /local/sitebuilder/components/disk/api.php отдавал ещё имя пользователя:

createdById: 99,
createdByName: "Павел Евгеньевич Махов"

1. В api.php добавь helper для имени пользователя

Файл:

/local/sitebuilder/components/disk/api.php

Вставь где-нибудь сверху, после require/use, но до обработки action:

if (!function_exists('sb_disk_user_name_by_id')) {
    function sb_disk_user_name_by_id(int $userId): string
    {
        static $cache = [];

        if ($userId <= 0) {
            return '';
        }

        if (isset($cache[$userId])) {
            return $cache[$userId];
        }

        $name = '';

        $rs = \CUser::GetByID($userId);
        if ($user = $rs->Fetch()) {
            $lastName = trim((string)($user['LAST_NAME'] ?? ''));
            $firstName = trim((string)($user['NAME'] ?? ''));
            $secondName = trim((string)($user['SECOND_NAME'] ?? ''));

            $name = trim($lastName . ' ' . $firstName . ' ' . $secondName);

            if ($name === '') {
                $name = trim((string)($user['LOGIN'] ?? ''));
            }

            if ($name === '') {
                $name = trim((string)($user['EMAIL'] ?? ''));
            }
        }

        if ($name === '') {
            $name = 'ID ' . $userId;
        }

        $cache[$userId] = $name;

        return $name;
    }
}

if (!function_exists('sb_disk_object_created_by_id')) {
    function sb_disk_object_created_by_id($object): int
    {
        if (!is_object($object)) {
            return 0;
        }

        if (method_exists($object, 'getCreatedBy')) {
            return (int)$object->getCreatedBy();
        }

        if (method_exists($object, 'getCreateUserId')) {
            return (int)$object->getCreateUserId();
        }

        if (method_exists($object, 'getCreatedById')) {
            return (int)$object->getCreatedById();
        }

        return 0;
    }
}


---

2. В месте, где API собирает items, добавь имя

В api.php найди место, где формируется элемент списка, примерно так:

$items[] = [
    'id' => (int)$object->getId(),
    'name' => (string)$object->getName(),
    ...
];

Перед массивом добавь:

$createdById = sb_disk_object_created_by_id($object);
$createdByName = sb_disk_user_name_by_id($createdById);

И внутрь массива добавь:

'createdById' => $createdById,
'createdByName' => $createdByName,
'createdBy' => $createdByName,

Должно получиться примерно так:

$createdById = sb_disk_object_created_by_id($object);
$createdByName = sb_disk_user_name_by_id($createdById);

$items[] = [
    'id' => (int)$object->getId(),
    'name' => (string)$object->getName(),
    'entityType' => $object instanceof \Bitrix\Disk\Folder ? 'folder' : 'file',

    'createdById' => $createdById,
    'createdByName' => $createdByName,
    'createdBy' => $createdByName,

    // остальные поля как были
];


---

3. Важно сделать это и для list, и для search

Если в api.php есть отдельная логика для:

list
search

то добавь эти поля в обоих местах.
Иначе в обычной папке имя будет, а в поиске снова будет ID.


---

4. Обнови версии файлов

В public_page.php:

<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js?v=20"></script>

Если CSS не менял, его версию можно не трогать.

После этого сделай Ctrl + F5.

Если не найдёшь место, где формируется $items[], пришли мне целиком:

/local/sitebuilder/components/disk/api.php

и я сразу вставлю в правильные места.