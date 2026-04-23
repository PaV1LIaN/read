Ошибка точная: у тебя в текущей версии Bitrix метод addSubFolder() не принимает SecurityContext вторым аргументом.

Сейчас у тебя передается так:

$rootFolder->addSubFolder([...], $securityContext);

А по факту второй аргумент должен быть массивом прав.


---

Что сломано

Нужно исправить как минимум 3 места:

1. SiteDiskInitializer.php


2. BlockDiskInitializer.php


3. DiskBitrixStorageAdapter.php в createFolder()




---

1. Исправь SiteDiskInitializer.php

Файл:

/local/sitebuilder/components/disk/lib/SiteDiskInitializer.php

Замени целиком на это:

<?php

use Bitrix\Disk\Driver;
use Bitrix\Disk\Storage;
use Bitrix\Disk\Folder;

class SiteDiskInitializer
{
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
        ]);

        if (!$siteFolder instanceof Folder) {
            throw new RuntimeException('DISK_SITE_ROOT_CREATE_ERROR');
        }

        SiteRepository::updateRootDiskFolderId($siteId, (int)$siteFolder->getId());

        return (int)$siteFolder->getId();
    }

    protected static function resolveDefaultStorage(int $currentUserId): ?Storage
    {
        $driver = Driver::getInstance();

        $storage = $driver->getStorageByUserId($currentUserId);
        if ($storage instanceof Storage) {
            return $storage;
        }

        return null;
    }
}


---

2. Исправь BlockDiskInitializer.php

Файл:

/local/sitebuilder/components/disk/lib/BlockDiskInitializer.php

Замени целиком на это:

<?php

use Bitrix\Disk\Folder;

class BlockDiskInitializer
{
    public static function ensureBlockRootFolder(
        int $siteId,
        int $pageId,
        int $blockId,
        int $currentUserId,
        string $blockTitle = ''
    ): int {
        $settings = DiskSettingsRepository::ensureExistsForBlock($blockId, $siteId, $pageId, $currentUserId);

        if (!empty($settings['rootFolderId'])) {
            return (int)$settings['rootFolderId'];
        }

        $site = SiteRepository::getById($siteId);
        if (!$site) {
            throw new RuntimeException('SITE_NOT_FOUND');
        }

        $siteRootFolderId = SiteDiskInitializer::ensureSiteRootFolder(
            $siteId,
            $currentUserId,
            (string)$site['name']
        );

        $siteRootFolder = Folder::loadById($siteRootFolderId);
        if (!$siteRootFolder instanceof Folder) {
            throw new RuntimeException('SITE_ROOT_FOLDER_NOT_FOUND');
        }

        $folderName = $blockTitle !== ''
            ? ('Блок: ' . $blockTitle)
            : ('Блок #' . $blockId);

        $created = $siteRootFolder->addSubFolder([
            'NAME' => $folderName,
            'CREATED_BY' => $currentUserId,
        ]);

        if (!$created instanceof Folder) {
            throw new RuntimeException('BLOCK_ROOT_CREATE_ERROR');
        }

        DiskSettingsRepository::save($blockId, [
            'rootMode' => 'block',
            'rootFolderId' => (int)$created->getId(),
            'useSiteRootFallback' => true,
        ]);

        return (int)$created->getId();
    }
}


---

3. Исправь DiskBitrixStorageAdapter.php

Там у тебя такая же ошибка будет при создании папки.

Файл:

/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php

Найди метод:

public function createFolder(DiskContext $context, int $parentFolderId, string $name): array

и замени его на:

public function createFolder(DiskContext $context, int $parentFolderId, string $name): array
{
    $parentFolder = $this->getFolderById($parentFolderId);

    $createdFolder = $parentFolder->addSubFolder([
        'NAME' => $name,
        'CREATED_BY' => $context->currentUserId,
    ]);

    if (!$createdFolder instanceof Folder) {
        throw new RuntimeException('DISK_CREATE_FOLDER_ERROR');
    }

    return $this->normalizeFolder($context, $createdFolder);
}


---

Почему это произошло

Я раньше ориентировался на схему, где в некоторые методы Disk передают security context.
Но по твоей ошибке видно, что именно в этой версии Bitrix addSubFolder() ожидает:

1 аргумент: массив полей

2 аргумент: массив прав


а не SecurityContext.


---

Что делать дальше

После замены этих 3 мест:

1. обнови страницу


2. снова открой публичную страницу с disk


3. посмотри новый bootstrap response




---

Что, скорее всего, будет дальше

Очень вероятно, что bootstrap уже станет ok: true и появится rootFolderId.

Если нет, то следующая ошибка будет уже следующего уровня:

storage

права

list

folder scope


Пришли следующий response, если появится новая ошибка.