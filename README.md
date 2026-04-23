Да, сейчас это ожидаемо: у тебя в SiteDiskInitializer используется логика вида

$storage = $driver->getStorageByUserId($currentUserId);

а это всегда ведет в «Мой диск».

Для sitebuilder правильнее не искать storage по пользователю, а создавать папки в заранее выбранной общей папке общего диска.

Как лучше сделать

Самый надежный вариант для твоего проекта:

не пытаться “угадывать” общий диск через API Bitrix

завести один корневой folderId общего диска

хранить его в настройке

все папки сайтов создавать внутри него


Так решение будет стабильным и не завязано на нюансы версии Bitrix Disk.


---

Что поменять

1. Добавь константу общего корня

Файл

/local/sitebuilder/components/disk/lib/SiteDiskInitializer.php

Замени на такой вариант:

<?php

use Bitrix\Disk\Folder;

class SiteDiskInitializer
{
    /**
     * Укажи ID папки в "Общий диск", внутри которой нужно создавать папки сайтов.
     * Например, заранее руками создай в Общем диске папку "SiteBuilder" и подставь ее ID сюда.
     */
    protected const SHARED_ROOT_FOLDER_ID = 0;

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

        $sharedRootFolder = self::getSharedRootFolder();
        if (!$sharedRootFolder instanceof Folder) {
            throw new RuntimeException('SHARED_DISK_ROOT_FOLDER_NOT_FOUND');
        }

        $folderBaseName = $siteName !== ''
            ? ('Сайт ' . $siteName)
            : ('Сайт ' . (string)$site['name']);

        $folderName = DiskNameSanitizer::sanitizeFolderName($folderBaseName, 'Сайт');

        $siteFolder = $sharedRootFolder->addSubFolder([
            'NAME' => $folderName,
            'CREATED_BY' => $currentUserId,
        ], [], true);

        if (!$siteFolder instanceof Folder) {
            $errors = [];

            if (method_exists($sharedRootFolder, 'getErrors')) {
                foreach ((array)$sharedRootFolder->getErrors() as $error) {
                    if (is_object($error) && method_exists($error, 'getMessage')) {
                        $errors[] = $error->getMessage();
                    } else {
                        $errors[] = (string)$error;
                    }
                }
            }

            throw new RuntimeException(
                'DISK_SITE_ROOT_CREATE_ERROR' . (!empty($errors) ? ': ' . implode(' | ', $errors) : '')
            );
        }

        SiteRepository::updateRootDiskFolderId($siteId, (int)$siteFolder->getId());

        return (int)$siteFolder->getId();
    }

    protected static function getSharedRootFolder(): Folder
    {
        $folderId = (int)self::SHARED_ROOT_FOLDER_ID;
        if ($folderId <= 0) {
            throw new RuntimeException('SHARED_ROOT_FOLDER_ID_NOT_CONFIGURED');
        }

        $folder = Folder::loadById($folderId);
        if (!$folder instanceof Folder) {
            throw new RuntimeException('SHARED_ROOT_FOLDER_NOT_FOUND');
        }

        return $folder;
    }
}


---

2. Что нужно сделать руками в Bitrix

1. Открой Общий диск


2. Создай там папку, например:

SiteBuilder


3. Узнай ее ID


4. Подставь этот ID сюда:



protected const SHARED_ROOT_FOLDER_ID = 123;


---

Что получится

Было:

Мой диск
└── Сайт ...

Станет:

Общий диск
└── SiteBuilder
    ├── Сайт Портал продаж
    ├── Сайт HR
    └── Сайт Документы


---

Почему это лучше, чем storage по пользователю

Потому что тогда:

папки сайтов не зависят от конкретного пользователя

если сайт создал один пользователь, другой все равно работает с тем же корнем

sitebuilder становится действительно общим пространством



---

Что еще проверить после правки

После замены:

1. очисти disk_folder_id у тестового сайта, если он уже указывает на папку в “Моем диске”


2. обнови страницу с блоком disk


3. корневая папка должна создаться уже в общем диске



Например SQL:

UPDATE sitebuilder.site
SET disk_folder_id = NULL
WHERE id = 7;


---

Если хочешь сделать совсем правильно

Следующий шаг лучше такой:

не держать SHARED_ROOT_FOLDER_ID хардкодом в PHP

хранить его в настройках sitebuilder


Например:

в sitebuilder.settings

или в settings_json какого-то системного сайта

или в отдельном config-файле


Но на первом этапе константа — самый быстрый и надежный способ.

Если хочешь, я следующим сообщением дам вариант без хардкода, через настройку в PostgreSQL или config.php.