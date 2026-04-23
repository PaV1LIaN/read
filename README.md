Значит проблема уже не в sessid и не в сигнатуре метода.
Сейчас нужно увидеть настоящую ошибку Bitrix Disk, потому что DISK_SITE_ROOT_CREATE_ERROR — это только наша обертка.

Что сделать сейчас

1. Временно вывести реальную ошибку в SiteDiskInitializer.php

Файл:

/local/sitebuilder/components/disk/lib/SiteDiskInitializer.php

Найди кусок:

$siteFolder = $rootFolder->addSubFolder([
    'NAME' => $folderName,
    'CREATED_BY' => $currentUserId,
], [], true);

if (!$siteFolder instanceof Folder) {
    throw new RuntimeException('DISK_SITE_ROOT_CREATE_ERROR');
}

И замени на это:

$siteFolder = $rootFolder->addSubFolder([
    'NAME' => $folderName,
    'CREATED_BY' => $currentUserId,
], [], true);

if (!$siteFolder instanceof Folder) {
    $errors = [];

    if (is_object($rootFolder) && method_exists($rootFolder, 'getErrors')) {
        foreach ((array)$rootFolder->getErrors() as $error) {
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


---

2. Сделать то же самое для BlockDiskInitializer.php

Файл:

/local/sitebuilder/components/disk/lib/BlockDiskInitializer.php

Найди:

$created = $siteRootFolder->addSubFolder([
    'NAME' => $folderName,
    'CREATED_BY' => $currentUserId,
], [], true);

if (!$created instanceof Folder) {
    throw new RuntimeException('BLOCK_ROOT_CREATE_ERROR');
}

И замени на:

$created = $siteRootFolder->addSubFolder([
    'NAME' => $folderName,
    'CREATED_BY' => $currentUserId,
], [], true);

if (!$created instanceof Folder) {
    $errors = [];

    if (is_object($siteRootFolder) && method_exists($siteRootFolder, 'getErrors')) {
        foreach ((array)$siteRootFolder->getErrors() as $error) {
            if (is_object($error) && method_exists($error, 'getMessage')) {
                $errors[] = $error->getMessage();
            } else {
                $errors[] = (string)$error;
            }
        }
    }

    throw new RuntimeException(
        'BLOCK_ROOT_CREATE_ERROR' . (!empty($errors) ? ': ' . implode(' | ', $errors) : '')
    );
}


---

3. Открыть страницу еще раз и посмотреть новый bootstrap response

Теперь в message должна прийти уже реальная причина.
Самые вероятные варианты:

нет прав на создание папки в этом storage

storage не тот

нельзя создавать в корне этого storage

конфликт имени

диск пользователя не инициализирован



---

Что очень вероятно у тебя

С учетом Bitrix24 и того, что ты работаешь из публичной части, самая частая причина здесь такая:

getStorageByUserId($currentUserId) возвращает storage, но в его корне нельзя создавать папки таким образом
или у пользователя личный диск не инициализирован.

Тогда правильнее будет создавать не в личном storage пользователя, а в заранее выбранном общем storage.


---

Быстрая дополнительная проверка

Создай временный файл:

/local/sitebuilder/tools/test_disk_storage.php

и вставь:

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

use Bitrix\Disk\Driver;
use Bitrix\Disk\Storage;
use Bitrix\Disk\Folder;

global $USER;

echo '<pre>';

$userId = (int)$USER->GetID();
echo "USER_ID: " . $userId . PHP_EOL;

$driver = Driver::getInstance();
$storage = $driver->getStorageByUserId($userId);

if (!$storage instanceof Storage) {
    echo "STORAGE: NOT FOUND" . PHP_EOL;
    echo '</pre>';
    return;
}

echo "STORAGE_ID: " . $storage->getId() . PHP_EOL;
echo "STORAGE_NAME: " . $storage->getName() . PHP_EOL;

$root = $storage->getRootObject();

if (!$root instanceof Folder) {
    echo "ROOT: NOT FOUND" . PHP_EOL;
    echo '</pre>';
    return;
}

echo "ROOT_ID: " . $root->getId() . PHP_EOL;
echo "ROOT_NAME: " . $root->getName() . PHP_EOL;

$testFolder = $root->addSubFolder([
    'NAME' => 'SB_TEST_' . date('Ymd_His'),
    'CREATED_BY' => $userId,
], [], true);

if ($testFolder instanceof Folder) {
    echo "CREATE: OK" . PHP_EOL;
    echo "FOLDER_ID: " . $testFolder->getId() . PHP_EOL;
} else {
    echo "CREATE: FAIL" . PHP_EOL;

    if (method_exists($root, 'getErrors')) {
        foreach ((array)$root->getErrors() as $error) {
            if (is_object($error) && method_exists($error, 'getMessage')) {
                echo "ERROR: " . $error->getMessage() . PHP_EOL;
            } else {
                echo "ERROR: " . print_r($error, true) . PHP_EOL;
            }
        }
    }
}

echo '</pre>';

Это покажет:

находится ли storage

можно ли реально создавать подпапку вручную



---

Что прислать мне следующим сообщением

Пришли:

1. новый bootstrap response после правки getErrors()


2. или вывод test_disk_storage.php



После этого уже будет видно, это:

проблема прав

проблема storage

проблема конкретного Bitrix Disk API поведения на твоем сервере.