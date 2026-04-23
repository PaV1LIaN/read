Значит метод вызывается извне класса, из DiskValidator, поэтому protected не подходит.

Что исправить

Файл:

/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php

Найди метод:

protected function isFolderInsideRoot(int $folderId, int $rootFolderId): bool

И замени protected на public:

public function isFolderInsideRoot(int $folderId, int $rootFolderId): bool
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

После этого:

1. сохрани файл


2. Ctrl + F5


3. снова открой папку



Если вылезет следующая ошибка — пришли новый response api.php?action=list.