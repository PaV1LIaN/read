Отлично, bootstrap уже работает.
Текущая проблема теперь в list:

Call to undefined method Bitrix\Disk\Folder::getPath()

Значит в DiskBitrixStorageAdapter.php где-то используется Folder::getPath(), а в твоей версии Bitrix такого метода у Folder нет.


---

Что нужно исправить

Нужно убрать использование getPath() и не рассчитывать путь через этот метод.

Скорее всего ошибка сидит:

либо в listItems()

либо в getBreadcrumbs()

либо в normalizeFolder()



---

Самый надежный фикс

Файл

/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php

Замени его целиком на этот вариант:

<?php

use Bitrix\Disk\BaseObject;
use Bitrix\Disk\File;
use Bitrix\Disk\Folder;
use Bitrix\Disk\Internals\ObjectTable;

class DiskBitrixStorageAdapter
{
    protected int $currentUserId;

    public function __construct(int $currentUserId)
    {
        $this->currentUserId = $currentUserId;
    }

    public function listItems(DiskContext $context, int $folderId, array $options = []): array
    {
        $folder = $this->getFolderById($folderId);

        $children = $folder->getChildren(
            \Bitrix\Disk\Driver::getInstance()->getFakeSecurityContext($this->currentUserId)
        );

        $items = [];

        foreach ($children as $child) {
            if ($child instanceof Folder) {
                $items[] = $this->normalizeFolder($context, $child);
            } elseif ($child instanceof File) {
                $items[] = $this->normalizeFile($context, $child);
            }
        }

        $sortBy = (string)($options['sortBy'] ?? 'updatedAt');
        $sortDir = strtolower((string)($options['sortDir'] ?? 'desc')) === 'asc' ? 'asc' : 'desc';

        usort($items, function ($a, $b) use ($sortBy, $sortDir) {
            $aVal = $a[$sortBy] ?? null;
            $bVal = $b[$sortBy] ?? null;

            if ($sortBy === 'name') {
                $cmp = strnatcasecmp((string)$aVal, (string)$bVal);
            } elseif ($sortBy === 'size') {
                $cmp = (int)$aVal <=> (int)$bVal;
            } else {
                $cmp = strcmp((string)$aVal, (string)$bVal);
            }

            return $sortDir === 'asc' ? $cmp : -$cmp;
        });

        return $items;
    }

    public function getBreadcrumbs(DiskContext $context, int $folderId): array
    {
        $breadcrumbs = [];
        $current = $this->getFolderById($folderId);

        while ($current instanceof Folder) {
            array_unshift($breadcrumbs, [
                'id' => (int)$current->getId(),
                'name' => (string)$current->getName(),
            ]);

            $parentId = (int)$current->getParentId();
            if ($parentId <= 0) {
                break;
            }

            $parent = Folder::loadById($parentId);
            if (!$parent instanceof Folder) {
                break;
            }

            $current = $parent;
        }

        return $breadcrumbs;
    }

    public function uploadFiles(DiskContext $context, int $folderId, array $files, array $settings = []): array
    {
        $folder = $this->getFolderById($folderId);
        $uploaded = [];

        foreach ($files as $file) {
            $safeName = DiskNameSanitizer::sanitizeFolderName((string)$file['name'], 'file');

            $createdFile = $folder->uploadFile(
                [
                    'NAME' => $safeName,
                    'TMP_NAME' => $file['tmp_name'],
                    'TYPE' => $file['type'] ?? '',
                    'SIZE' => (int)($file['size'] ?? 0),
                    'CREATED_BY' => $context->currentUserId,
                ],
                [],
                true
            );

            if ($createdFile instanceof File) {
                $uploaded[] = $this->normalizeFile($context, $createdFile);
            }
        }

        return $uploaded;
    }

    public function createFolder(DiskContext $context, int $parentFolderId, string $name): array
    {
        $parentFolder = $this->getFolderById($parentFolderId);

        $safeName = DiskNameSanitizer::sanitizeFolderName($name, 'Новая папка');

        $createdFolder = $parentFolder->addSubFolder([
            'NAME' => $safeName,
            'CREATED_BY' => $context->currentUserId,
        ], [], true);

        if (!$createdFolder instanceof Folder) {
            throw new RuntimeException('DISK_CREATE_FOLDER_ERROR');
        }

        return $this->normalizeFolder($context, $createdFolder);
    }

    public function rename(DiskContext $context, string $entityType, int $entityId, string $newName): array
    {
        $safeName = DiskNameSanitizer::sanitizeFolderName($newName, 'Новый объект');

        if ($entityType === 'folder') {
            $folder = $this->getFolderById($entityId);
            $folder->rename($safeName, $context->currentUserId);

            return $this->normalizeFolder($context, $folder);
        }

        if ($entityType === 'file') {
            $file = $this->getFileById($entityId);
            $file->rename($safeName, $context->currentUserId);

            return $this->normalizeFile($context, $file);
        }

        throw new RuntimeException('INVALID_ENTITY_TYPE');
    }

    public function delete(DiskContext $context, array $items): array
    {
        $deleted = [];

        foreach ($items as $item) {
            $entityType = (string)($item['entityType'] ?? '');
            $id = (int)($item['id'] ?? 0);

            if ($id <= 0) {
                continue;
            }

            if ($entityType === 'folder') {
                $folder = $this->getFolderById($id);
                $folder->markDeleted($context->currentUserId);
                $deleted[] = ['entityType' => 'folder', 'id' => $id];
            } elseif ($entityType === 'file') {
                $file = $this->getFileById($id);
                $file->markDeleted($context->currentUserId);
                $deleted[] = ['entityType' => 'file', 'id' => $id];
            }
        }

        return $deleted;
    }

    public function move(DiskContext $context, array $items, int $targetFolderId): array
    {
        $targetFolder = $this->getFolderById($targetFolderId);
        $result = [];

        foreach ($items as $item) {
            $entityType = (string)($item['entityType'] ?? '');
            $id = (int)($item['id'] ?? 0);

            if ($entityType === 'folder') {
                $folder = $this->getFolderById($id);
                $folder->moveTo($targetFolder, $context->currentUserId);
                $result[] = ['entityType' => 'folder', 'id' => $id];
            } elseif ($entityType === 'file') {
                $file = $this->getFileById($id);
                $file->moveTo($targetFolder, $context->currentUserId);
                $result[] = ['entityType' => 'file', 'id' => $id];
            }
        }

        return $result;
    }

    public function copy(DiskContext $context, array $items, int $targetFolderId): array
    {
        $targetFolder = $this->getFolderById($targetFolderId);
        $result = [];

        foreach ($items as $item) {
            $entityType = (string)($item['entityType'] ?? '');
            $id = (int)($item['id'] ?? 0);

            if ($entityType === 'folder') {
                $folder = $this->getFolderById($id);
                $copy = $folder->copyTo($targetFolder, $context->currentUserId, true);
                if ($copy instanceof Folder) {
                    $result[] = ['entityType' => 'folder', 'id' => (int)$copy->getId()];
                }
            } elseif ($entityType === 'file') {
                $file = $this->getFileById($id);
                $copy = $file->copyTo($targetFolder, $context->currentUserId, true);
                if ($copy instanceof File) {
                    $result[] = ['entityType' => 'file', 'id' => (int)$copy->getId()];
                }
            }
        }

        return $result;
    }

    public function search(DiskContext $context, int $rootFolderId, string $query, array $options = []): array
    {
        $query = mb_strtolower(trim($query));
        if ($query === '') {
            return [];
        }

        $result = [];
        $this->searchRecursive($context, $rootFolderId, $query, $result);

        return $result;
    }

    public function getDownloadUrl(DiskContext $context, int $fileId): string
    {
        $file = $this->getFileById($fileId);
        return (string)$file->getDownloadUrl();
    }

    protected function searchRecursive(DiskContext $context, int $folderId, string $query, array &$result): void
    {
        $folder = $this->getFolderById($folderId);

        $children = $folder->getChildren(
            \Bitrix\Disk\Driver::getInstance()->getFakeSecurityContext($this->currentUserId)
        );

        foreach ($children as $child) {
            $name = mb_strtolower((string)$child->getName());

            if (mb_strpos($name, $query) !== false) {
                if ($child instanceof Folder) {
                    $result[] = $this->normalizeFolder($context, $child);
                } elseif ($child instanceof File) {
                    $result[] = $this->normalizeFile($context, $child);
                }
            }

            if ($child instanceof Folder) {
                $this->searchRecursive($context, (int)$child->getId(), $query, $result);
            }
        }
    }

    protected function normalizeFolder(DiskContext $context, Folder $folder): array
    {
        return [
            'id' => (int)$folder->getId(),
            'entityType' => 'folder',
            'name' => (string)$folder->getName(),
            'extension' => '',
            'mimeType' => 'inode/directory',
            'size' => 0,
            'downloadUrl' => '',
            'previewUrl' => '',
            'createdAt' => $this->normalizeDate($folder->getCreateTime()),
            'updatedAt' => $this->normalizeDate($folder->getUpdateTime()),
            'createdBy' => (int)$folder->getCreatedBy(),
        ];
    }

    protected function normalizeFile(DiskContext $context, File $file): array
    {
        return [
            'id' => (int)$file->getId(),
            'entityType' => 'file',
            'name' => (string)$file->getName(),
            'originalName' => (string)$file->getName(),
            'extension' => (string)$file->getExtension(),
            'mimeType' => (string)$file->getMimeType(),
            'size' => (int)$file->getSize(),
            'downloadUrl' => (string)$file->getDownloadUrl(),
            'previewUrl' => '',
            'createdAt' => $this->normalizeDate($file->getCreateTime()),
            'updatedAt' => $this->normalizeDate($file->getUpdateTime()),
            'createdBy' => (int)$file->getCreatedBy(),
        ];
    }

    protected function normalizeDate($value): string
    {
        if ($value instanceof \Bitrix\Main\Type\DateTime) {
            return $value->format('Y-m-d H:i:s');
        }

        if ($value instanceof \DateTimeInterface) {
            return $value->format('Y-m-d H:i:s');
        }

        if (is_string($value)) {
            return $value;
        }

        return '';
    }

    protected function getFolderById(int $folderId): Folder
    {
        $folder = Folder::loadById($folderId);
        if (!$folder instanceof Folder) {
            throw new RuntimeException('DISK_FOLDER_NOT_FOUND');
        }

        return $folder;
    }

    protected function getFileById(int $fileId): File
    {
        $file = File::loadById($fileId);
        if (!$file instanceof File) {
            throw new RuntimeException('DISK_FILE_NOT_FOUND');
        }

        return $file;
    }
}


---

Что произошло

Где-то в старой версии адаптера использовался getPath(), но в твоей версии Bitrix\Disk\Folder такого метода нет.
В этом новом варианте путь вообще не нужен — для UI достаточно:

id

name

createdAt

updatedAt

downloadUrl

entityType



---

Что сделать после замены

1. сохранить файл


2. сделать Ctrl + F5


3. снова открыть страницу с блоком disk




---

Что ожидаю дальше

После этого ошибка getPath() должна исчезнуть, и блок уже должен:

открыть корневую папку

показать список файлов/папок

дать создавать папки и грузить файлы


Если после этого всплывет следующая ошибка — пришли ее response/console, и добьем уже до полностью рабочего состояния.