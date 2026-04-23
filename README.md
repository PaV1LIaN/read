Проще всего — через временный PHP-файл.

Создай, например:

/local/sitebuilder/tools/disk_folder_id.php

И вставь туда:

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

use Bitrix\Disk\Folder;

$folderId = (int)($_GET['id'] ?? 0);

echo '<pre>';

if ($folderId > 0) {
    $folder = Folder::loadById($folderId);

    if ($folder instanceof Folder) {
        echo "ID: " . $folder->getId() . PHP_EOL;
        echo "NAME: " . $folder->getName() . PHP_EOL;
        echo "PARENT_ID: " . $folder->getParentId() . PHP_EOL;
    } else {
        echo "Folder not found" . PHP_EOL;
    }

    echo '</pre>';
    return;
}

echo "Передай ?id=..." . PHP_EOL;
echo '</pre>';

Но чтобы именно узнать ID нужной папки из общего диска, удобнее сделать список.

Создай файл:

/local/sitebuilder/tools/disk_list_root.php

С таким кодом:

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

use Bitrix\Disk\Driver;
use Bitrix\Disk\Storage;
use Bitrix\Disk\Folder;

global $USER;

echo '<pre>';

$userId = (int)$USER->GetID();
echo "USER_ID: " . $userId . PHP_EOL . PHP_EOL;

$driver = Driver::getInstance();
$storage = $driver->getStorageByUserId($userId);

if (!$storage instanceof Storage) {
    echo "Storage not found" . PHP_EOL;
    echo '</pre>';
    return;
}

$root = $storage->getRootObject();

if (!$root instanceof Folder) {
    echo "Root folder not found" . PHP_EOL;
    echo '</pre>';
    return;
}

echo "ROOT_ID: " . $root->getId() . PHP_EOL;
echo "ROOT_NAME: " . $root->getName() . PHP_EOL;
echo PHP_EOL;
echo "CHILD FOLDERS:" . PHP_EOL;

$children = $root->getChildren(\Bitrix\Disk\Driver::getInstance()->getFakeSecurityContext($userId));

foreach ($children as $child) {
    if ($child instanceof Folder) {
        echo 'ID=' . $child->getId() . ' | NAME=' . $child->getName() . PHP_EOL;
    }
}

echo '</pre>';

Но это покажет “Мой диск” для пользователя, а тебе нужен именно “Общий диск”.

Поэтому самый рабочий способ — открыть нужную папку в Bitrix и взять ID через отладочный файл по имени.


---

Самый удобный способ для общего диска

Если ты уже создал в Общем диске папку, например SiteBuilder, то можно написать поиск по имени:

/local/sitebuilder/tools/find_disk_folder.php

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

use Bitrix\Disk\Internals\ObjectTable;

$name = trim((string)($_GET['name'] ?? ''));

echo '<pre>';

if ($name === '') {
    echo "Передай ?name=SiteBuilder" . PHP_EOL;
    echo '</pre>';
    return;
}

$rows = ObjectTable::getList([
    'filter' => [
        '=NAME' => $name,
        '=TYPE' => 2, // папка
        '=DELETED_TYPE' => 0,
    ],
    'select' => ['ID', 'NAME', 'PARENT_ID', 'REAL_OBJECT_ID']
])->fetchAll();

if (!$rows) {
    echo "Ничего не найдено" . PHP_EOL;
    echo '</pre>';
    return;
}

foreach ($rows as $row) {
    echo 'ID=' . $row['ID']
        . ' | NAME=' . $row['NAME']
        . ' | PARENT_ID=' . $row['PARENT_ID']
        . ' | REAL_OBJECT_ID=' . $row['REAL_OBJECT_ID']
        . PHP_EOL;
}

echo '</pre>';

Открой:

/local/sitebuilder/tools/find_disk_folder.php?name=SiteBuilder

и он покажет ID папки.


---

Самый простой способ без кода

Если в интерфейсе Bitrix открыть папку, иногда ее objectId или folderId видно:

в URL

в ajax-запросах DevTools

в параметрах popup/sidepanel


Но через PHP-файл обычно быстрее и надежнее.


---

Что я советую

Сделай именно find_disk_folder.php?name=SiteBuilder, это самый практичный путь.

Если хочешь, я могу сразу дать тебе готовый файл, который ищет только папки общего диска и выводит их деревом.

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