Теперь причина точная: DiskValidator вызывает метод по старой сигнатуре.

Сейчас у тебя метод сделан так:

isFolderInsideRoot(int $folderId, int $rootFolderId)

А DiskValidator вызывает его так:

$adapter->isFolderInsideRoot($context, $folderId, $rootFolderId)

То есть первым аргументом передается DiskContext.

Что исправить

Файл

/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php

Замени метод isFolderInsideRoot(...) целиком на этот

public function isFolderInsideRoot($arg1, $arg2, $arg3 = null): bool
{
    if ($arg1 instanceof DiskContext) {
        $folderId = (int)$arg2;
        $rootFolderId = (int)$arg3;
    } else {
        $folderId = (int)$arg1;
        $rootFolderId = (int)$arg2;
    }

    if ($folderId <= 0 || $rootFolderId <= 0) {
        return false;
    }

    if ($folderId === $rootFolderId) {
        return true;
    }

    $current = Folder::loadById($folderId);

    while ($current instanceof Folder) {
        $currentId = (int)$current->getId();
        if ($currentId === $rootFolderId) {
            return true;
        }

        $parentId = (int)$current->getParentId();
        if ($parentId <= 0) {
            break;
        }

        $current = Folder::loadById($parentId);
    }

    return false;
}

Почему так

Это самый быстрый безопасный фикс:

поддержит старый вызов из DiskValidator

не сломает новый прямой вызов

не требует сейчас лезть в другие файлы


После этого

1. сохрани файл


2. Ctrl + F5


3. снова открой папку



Если снова будет ошибка, пришли новый api.php?action=list response.