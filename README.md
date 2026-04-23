Причина точная: в DiskBitrixStorageAdapter.php где-то вызывается метод:

$this->isFolderInsideRoot(...)

а в текущей версии файла после наших замен этот метод отсутствует.

Нужно просто вернуть его.

Файл

/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php

Добавь в класс этот метод

Лучше вставить его перед getFolderById():

protected function isFolderInsideRoot(int $folderId, int $rootFolderId): bool
{
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


---

Если у тебя есть метод проверки перед list/search/move/delete

И он выглядит примерно так:

if (!$this->isFolderInsideRoot($folderId, $rootFolderId)) {
    throw new RuntimeException('FOLDER_OUT_OF_ROOT_SCOPE');
}

то после добавления метода все должно заработать.


---

Почему это чинит

Метод просто поднимается по цепочке родителей папки:

текущая папка

ее родитель

родитель родителя

и так до корня


Если по дороге встречается rootFolderId, значит папка действительно внутри корня сайта.


---

После этого

1. сохрани DiskBitrixStorageAdapter.php


2. Ctrl + F5


3. снова кликни по папке




---

Что ожидаю дальше

С высокой вероятностью вход в папку заработает.

Если после этого будет новая ошибка, пришли новый response api.php?action=list.