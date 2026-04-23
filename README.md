Сейчас ошибка уже почти наверняка из-за конфликта имени папки.

addSubFolder() у тебя теперь вызывается правильно, но если папка с таким именем уже существует, Bitrix часто возвращает false, и ты получаешь:

DISK_SITE_ROOT_CREATE_ERROR

Для корня сайта это очень похоже на ситуацию:

сайт уже пытался создать папку раньше

или такая папка уже есть в Disk

или Bitrix не дает создать папку с тем же именем повторно


Что исправить

Нужно включить автогенерацию уникального имени.


---

1. Исправь SiteDiskInitializer.php

Файл:

/local/sitebuilder/components/disk/lib/SiteDiskInitializer.php

Замени кусок:

$siteFolder = $rootFolder->addSubFolder([
    'NAME' => $folderName,
    'CREATED_BY' => $currentUserId,
]);

на:

$siteFolder = $rootFolder->addSubFolder([
    'NAME' => $folderName,
    'CREATED_BY' => $currentUserId,
], [], true);

И лучше сразу весь метод сделать таким:

public static function ensureSiteRootFolder(int $siteId, int $currentUserId, string $siteName = ''): int
{
    $existing = SiteRepository::getRootDiskFolderId($siteId);
    if ($existing !== null && $existing > 0) {
        return $existing;
    }

    $site = SiteRepository::getById($siteId);
    if (!$site) {
        throw new RuntimeException('SITE_NOT_FOUND');
    }

    $storage = self::resolveDefaultStorage($currentUserId);
    if (!$storage instanceof Storage) {
        throw new RuntimeException('DISK_STORAGE_NOT_FOUND');
    }

    $rootFolder = $storage->getRootObject();
    if (!$rootFolder instanceof Folder) {
        throw new RuntimeException('DISK_STORAGE_ROOT_NOT_FOUND');
    }

    $folderName = $siteName !== ''
        ? ('Сайт: ' . $siteName)
        : ('Сайт: ' . (string)$site['name']);

    $siteFolder = $rootFolder->addSubFolder([
        'NAME' => $folderName,
        'CREATED_BY' => $currentUserId,
    ], [], true);

    if (!$siteFolder instanceof Folder) {
        throw new RuntimeException('DISK_SITE_ROOT_CREATE_ERROR');
    }

    SiteRepository::updateRootDiskFolderId($siteId, (int)$siteFolder->getId());

    return (int)$siteFolder->getId();
}


---

2. Исправь BlockDiskInitializer.php

Там тоже нужно то же самое.

Замени:

$created = $siteRootFolder->addSubFolder([
    'NAME' => $folderName,
    'CREATED_BY' => $currentUserId,
]);

на:

$created = $siteRootFolder->addSubFolder([
    'NAME' => $folderName,
    'CREATED_BY' => $currentUserId,
], [], true);


---

3. Исправь DiskBitrixStorageAdapter.php

В createFolder() тоже замени:

$createdFolder = $parentFolder->addSubFolder([
    'NAME' => $name,
    'CREATED_BY' => $context->currentUserId,
]);

на:

$createdFolder = $parentFolder->addSubFolder([
    'NAME' => $name,
    'CREATED_BY' => $context->currentUserId,
], [], true);


---

После этого

Сделай обновление страницы и снова открой bootstrap.

Ожидаемо дальше будет либо:

ok: true с rootFolderId

либо уже следующая конкретная ошибка, но не эта


Если после этого все еще будет DISK_SITE_ROOT_CREATE_ERROR, тогда следующим шагом надо будет вывести реальную причину из Disk, а не общую заглушку.