Отлично. Делаем вариант B — адаптер под Bitrix Disk.

Ниже даю уже практический серверный слой под Bitrix Disk, который можно встроить в ваш компонент disk и дальше развивать.

Я исхожу из такой модели:

сам компонент живет в /local/sitebuilder/components/disk/;

корневая папка хранится как root_folder_id;

это ID папки Bitrix Disk;

все операции выполняются через Bitrix\Disk\Folder, Bitrix\Disk\File, Bitrix\Disk\Driver;

компонент по-прежнему жестко работает в контексте:

siteId

pageId

blockId

currentUserId




---

1. Что нужно подключить в bootstrap

Обновите bootstrap.php, чтобы там загружались модули Bitrix и новый адаптер.

bootstrap.php

<?php

if (!defined('SITEBUILDER_DISK_BOOTSTRAP')) {
    define('SITEBUILDER_DISK_BOOTSTRAP', true);

    if (!defined('NO_KEEP_STATISTIC')) {
        define('NO_KEEP_STATISTIC', true);
    }
    if (!defined('NO_AGENT_STATISTIC')) {
        define('NO_AGENT_STATISTIC', true);
    }
    if (!defined('DisableEventsCheck')) {
        define('DisableEventsCheck', true);
    }

    if (file_exists($_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php')) {
        require_once $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';
    }

    \Bitrix\Main\Loader::includeModule('disk');

    require_once __DIR__ . '/lib/helpers.php';
    require_once __DIR__ . '/lib/DiskDb.php';
    require_once __DIR__ . '/lib/DiskContext.php';
    require_once __DIR__ . '/lib/DiskResponse.php';
    require_once __DIR__ . '/lib/BlockRepository.php';
    require_once __DIR__ . '/lib/SiteRepository.php';
    require_once __DIR__ . '/lib/DiskSettingsRepository.php';
    require_once __DIR__ . '/lib/DiskRootResolver.php';
    require_once __DIR__ . '/lib/DiskValidator.php';
    require_once __DIR__ . '/lib/DiskPermissionService.php';
    require_once __DIR__ . '/lib/DiskStorageAdapterInterface.php';
    require_once __DIR__ . '/lib/DiskBitrixStorageAdapter.php';
}


---

2. Что меняется концептуально

Теперь:

root_folder_id = ID папки Bitrix Disk

все файлы и папки получаются напрямую из Disk

таблицы sitebuilder_disk_folder и sitebuilder_disk_file можно не заводить на первом этапе

ваша БД хранит только:

сайты

страницы

блоки

настройки компонента

привязку корневой папки



Это хороший и практичный путь для Bitrix-проекта.


---

3. Интерфейс storage оставляем тем же

lib/DiskStorageAdapterInterface.php

Можно оставить почти без изменений:

<?php

interface DiskStorageAdapterInterface
{
    public function listItems(DiskContext $context, int $folderId, array $options = []): array;

    public function createFolder(DiskContext $context, int $parentFolderId, string $name): array;

    public function uploadFiles(DiskContext $context, int $folderId, array $files, array $options = []): array;

    public function rename(DiskContext $context, string $entityType, int $entityId, string $newName): array;

    public function delete(DiskContext $context, array $items): array;

    public function move(DiskContext $context, array $items, int $targetFolderId): array;

    public function copy(DiskContext $context, array $items, int $targetFolderId): array;

    public function search(DiskContext $context, int $rootFolderId, string $query, array $options = []): array;

    public function getBreadcrumbs(DiskContext $context, int $folderId): array;

    public function getDownloadUrl(DiskContext $context, int $fileId): string;

    public function isFolderInsideRoot(DiskContext $context, int $folderId, int $rootFolderId): bool;
}


---

4. Главный файл: lib/DiskBitrixStorageAdapter.php

Это центральный файл варианта B.

lib/DiskBitrixStorageAdapter.php

<?php

use Bitrix\Disk\Driver;
use Bitrix\Disk\File;
use Bitrix\Disk\Folder;
use Bitrix\Disk\BaseObject;
use Bitrix\Main\Type\DateTime;

class DiskBitrixStorageAdapter implements DiskStorageAdapterInterface
{
    protected Driver $driver;
    protected \Bitrix\Disk\Security\SecurityContext $securityContext;

    public function __construct(?int $userId = null)
    {
        $this->driver = Driver::getInstance();

        global $USER;

        $userId = $userId ?: (int)($USER instanceof CUser ? $USER->GetID() : 0);
        if ($userId <= 0) {
            throw new RuntimeException('NOT_AUTHORIZED');
        }

        $this->securityContext = $this->driver->getFakeSecurityContext($userId);
    }

    public function listItems(DiskContext $context, int $folderId, array $options = []): array
    {
        $folder = $this->getFolderById($folderId);

        $sortBy = (string)($options['sortBy'] ?? 'updatedAt');
        $sortDir = strtolower((string)($options['sortDir'] ?? 'desc')) === 'asc' ? 'asc' : 'desc';

        $children = $folder->getChildren($this->securityContext, [
            'filter' => [
                '=DELETED_TYPE' => 0,
            ],
            'order' => $this->normalizeOrder($sortBy, $sortDir),
        ]);

        $items = [];

        foreach ($children as $child) {
            if ($child instanceof Folder) {
                $items[] = $this->normalizeFolder($context, $child);
                continue;
            }

            if ($child instanceof File) {
                $items[] = $this->normalizeFile($context, $child);
            }
        }

        return $items;
    }

    public function createFolder(DiskContext $context, int $parentFolderId, string $name): array
    {
        $parentFolder = $this->getFolderById($parentFolderId);

        $createdFolder = $parentFolder->addSubFolder([
            'NAME' => $name,
            'CREATED_BY' => $context->currentUserId,
        ], $this->securityContext);

        if (!$createdFolder instanceof Folder) {
            throw new RuntimeException('DISK_CREATE_FOLDER_ERROR');
        }

        return $this->normalizeFolder($context, $createdFolder);
    }

    public function uploadFiles(DiskContext $context, int $folderId, array $files, array $options = []): array
    {
        $folder = $this->getFolderById($folderId);

        $uploadedItems = [];

        foreach ($files as $file) {
            if (empty($file['tmp_name']) || !is_uploaded_file($file['tmp_name'])) {
                throw new RuntimeException('INVALID_UPLOADED_FILE');
            }

            $diskFile = $folder->uploadFile(
                $file,
                [
                    'NAME' => (string)$file['name'],
                    'CREATED_BY' => $context->currentUserId,
                ],
                [],
                $this->securityContext
            );

            if (!$diskFile instanceof File) {
                throw new RuntimeException('DISK_UPLOAD_ERROR');
            }

            $uploadedItems[] = $this->normalizeFile($context, $diskFile);
        }

        return [
            'uploaded' => count($uploadedItems),
            'items' => $uploadedItems,
        ];
    }

    public function rename(DiskContext $context, string $entityType, int $entityId, string $newName): array
    {
        $object = $this->getObjectByTypeAndId($entityType, $entityId);

        $result = $object->rename($newName, $context->currentUserId);
        if (!$result) {
            throw new RuntimeException('DISK_RENAME_ERROR');
        }

        $object = $this->reloadObject($object);

        if ($object instanceof Folder) {
            return $this->normalizeFolder($context, $object);
        }

        if ($object instanceof File) {
            return $this->normalizeFile($context, $object);
        }

        throw new RuntimeException('DISK_OBJECT_RELOAD_ERROR');
    }

    public function delete(DiskContext $context, array $items): array
    {
        $deleted = 0;

        foreach ($items as $item) {
            $entityType = (string)($item['entityType'] ?? '');
            $id = (int)($item['id'] ?? 0);

            if ($id <= 0 || !in_array($entityType, ['file', 'folder'], true)) {
                continue;
            }

            $object = $this->getObjectByTypeAndId($entityType, $id);
            $result = $object->delete($context->currentUserId);

            if ($result) {
                $deleted++;
            }
        }

        return [
            'deleted' => $deleted,
        ];
    }

    public function move(DiskContext $context, array $items, int $targetFolderId): array
    {
        $targetFolder = $this->getFolderById($targetFolderId);
        $moved = 0;

        foreach ($items as $item) {
            $entityType = (string)($item['entityType'] ?? '');
            $id = (int)($item['id'] ?? 0);

            if ($id <= 0 || !in_array($entityType, ['file', 'folder'], true)) {
                continue;
            }

            $object = $this->getObjectByTypeAndId($entityType, $id);
            $result = $object->moveTo($targetFolder, $context->currentUserId);

            if ($result) {
                $moved++;
            }
        }

        return [
            'moved' => $moved,
            'targetFolderId' => $targetFolderId,
        ];
    }

    public function copy(DiskContext $context, array $items, int $targetFolderId): array
    {
        $targetFolder = $this->getFolderById($targetFolderId);
        $copied = 0;

        foreach ($items as $item) {
            $entityType = (string)($item['entityType'] ?? '');
            $id = (int)($item['id'] ?? 0);

            if ($id <= 0 || !in_array($entityType, ['file', 'folder'], true)) {
                continue;
            }

            $object = $this->getObjectByTypeAndId($entityType, $id);
            $result = $object->copyTo($targetFolder, $context->currentUserId, true);

            if ($result) {
                $copied++;
            }
        }

        return [
            'copied' => $copied,
            'targetFolderId' => $targetFolderId,
        ];
    }

    public function search(DiskContext $context, int $rootFolderId, string $query, array $options = []): array
    {
        $query = trim($query);
        if ($query === '') {
            return [];
        }

        $rootFolder = $this->getFolderById($rootFolderId);

        $result = [];
        $this->searchRecursive($context, $rootFolder, mb_strtolower($query), $result);

        return $result;
    }

    public function getBreadcrumbs(DiskContext $context, int $folderId): array
    {
        $folder = $this->getFolderById($folderId);

        $breadcrumbs = [];
        $chain = $folder->getPath();

        foreach ($chain as $pathFolder) {
            if (!$pathFolder instanceof Folder) {
                continue;
            }

            $breadcrumbs[] = [
                'id' => (int)$pathFolder->getId(),
                'name' => (string)$pathFolder->getName(),
            ];
        }

        return $breadcrumbs;
    }

    public function getDownloadUrl(DiskContext $context, int $fileId): string
    {
        $file = $this->getFileById($fileId);
        return (string)$file->getDownloadUrl();
    }

    public function isFolderInsideRoot(DiskContext $context, int $folderId, int $rootFolderId): bool
    {
        if ($folderId === $rootFolderId) {
            return true;
        }

        $folder = $this->getFolderById($folderId);
        $path = $folder->getPath();

        foreach ($path as $pathFolder) {
            if ($pathFolder instanceof Folder && (int)$pathFolder->getId() === $rootFolderId) {
                return true;
            }
        }

        return false;
    }

    protected function searchRecursive(DiskContext $context, Folder $folder, string $query, array &$result): void
    {
        $children = $folder->getChildren($this->securityContext, [
            'filter' => [
                '=DELETED_TYPE' => 0,
            ],
            'order' => [
                'NAME' => 'ASC',
            ],
        ]);

        foreach ($children as $child) {
            $name = mb_strtolower((string)$child->getName());

            if (mb_stripos($name, $query) !== false) {
                if ($child instanceof Folder) {
                    $result[] = $this->normalizeFolder($context, $child);
                } elseif ($child instanceof File) {
                    $result[] = $this->normalizeFile($context, $child);
                }
            }

            if ($child instanceof Folder) {
                $this->searchRecursive($context, $child, $query, $result);
            }
        }
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

    protected function getObjectByTypeAndId(string $entityType, int $id): BaseObject
    {
        if ($entityType === 'folder') {
            return $this->getFolderById($id);
        }

        if ($entityType === 'file') {
            return $this->getFileById($id);
        }

        throw new RuntimeException('INVALID_ENTITY_TYPE');
    }

    protected function reloadObject(BaseObject $object): BaseObject
    {
        if ($object instanceof Folder) {
            return $this->getFolderById((int)$object->getId());
        }

        if ($object instanceof File) {
            return $this->getFileById((int)$object->getId());
        }

        throw new RuntimeException('DISK_OBJECT_RELOAD_ERROR');
    }

    protected function normalizeFolder(DiskContext $context, Folder $folder): array
    {
        return [
            'id' => (int)$folder->getId(),
            'entityType' => 'folder',
            'name' => (string)$folder->getName(),
            'extension' => null,
            'mimeType' => null,
            'size' => null,
            'updatedAt' => $this->formatDateTime($folder->getUpdateTime()),
            'createdAt' => $this->formatDateTime($folder->getCreateTime()),
            'createdBy' => (int)$folder->getCreatedBy(),
            'downloadUrl' => null,
            'previewUrl' => null,
        ];
    }

    protected function normalizeFile(DiskContext $context, File $file): array
    {
        $name = (string)$file->getName();
        $extension = strtolower(pathinfo($name, PATHINFO_EXTENSION));

        return [
            'id' => (int)$file->getId(),
            'entityType' => 'file',
            'name' => $name,
            'extension' => $extension ?: null,
            'mimeType' => (string)$file->getMimeType(),
            'size' => (int)$file->getSize(),
            'updatedAt' => $this->formatDateTime($file->getUpdateTime()),
            'createdAt' => $this->formatDateTime($file->getCreateTime()),
            'createdBy' => (int)$file->getCreatedBy(),
            'downloadUrl' => (string)$file->getDownloadUrl(),
            'previewUrl' => null,
        ];
    }

    protected function formatDateTime($value): string
    {
        if ($value instanceof DateTime) {
            return $value->format('d.m.Y H:i:s');
        }

        if ($value instanceof \Bitrix\Main\Type\Date) {
            return $value->format('d.m.Y');
        }

        return '';
    }

    protected function normalizeOrder(string $sortBy, string $sortDir): array
    {
        $sortDir = strtoupper($sortDir) === 'ASC' ? 'ASC' : 'DESC';

        switch ($sortBy) {
            case 'name':
                return ['NAME' => $sortDir];

            case 'size':
                return ['SIZE' => $sortDir];

            case 'createdAt':
                return ['CREATE_TIME' => $sortDir];

            case 'updatedAt':
            default:
                return ['UPDATE_TIME' => $sortDir];
        }
    }
}


---

5. Как заменить старый адаптер на Bitrix Disk

Где у вас сейчас было:

$adapter = new DiskStorageAdapter();

надо заменить на:

$adapter = new DiskBitrixStorageAdapter($context->currentUserId);


---

6. Как обновить DiskValidator::assertFolderInsideRoot()

Теперь проверка дерева может быть реальной.

lib/DiskValidator.php

Замените метод на такой:

public static function assertFolderInsideRoot(int $folderId, ?int $rootFolderId, ?DiskContext $context = null): void
{
    if ($folderId <= 0) {
        throw new RuntimeException('INVALID_FOLDER_ID');
    }

    if ($rootFolderId === null || $rootFolderId <= 0) {
        throw new RuntimeException('ROOT_FOLDER_NOT_RESOLVED');
    }

    if ($folderId === $rootFolderId) {
        return;
    }

    if (!$context instanceof DiskContext) {
        throw new RuntimeException('DISK_CONTEXT_REQUIRED');
    }

    $adapter = new DiskBitrixStorageAdapter($context->currentUserId);
    $ok = $adapter->isFolderInsideRoot($context, $folderId, $rootFolderId);

    if (!$ok) {
        throw new RuntimeException('FOLDER_OUT_OF_SCOPE');
    }
}

И тогда во всех action-файлах вместо:

DiskValidator::assertFolderInsideRoot($currentFolderId, $rootFolderId);

надо вызывать:

DiskValidator::assertFolderInsideRoot($currentFolderId, $rootFolderId, $context);


---

7. Обновленный actions/list.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1, // TODO: заменить на реального пользователя
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::ensureExistsForBlock(
    $context->blockId,
    $context->siteId,
    $context->pageId,
    $context->currentUserId
);

$rootFolderId = DiskRootResolver::resolve($context, $settings);
$permissions = DiskPermissionService::resolve($context, $settings, $rootFolderId);

DiskValidator::assertCan($permissions, 'canView');

if ($rootFolderId === null) {
    DiskResponse::success([
        'folder' => null,
        'breadcrumbs' => [],
        'items' => [],
    ], [
        'isEmpty' => true,
        'noRoot' => true,
        'total' => 0,
    ]);
}

$currentFolderId = (int)($data['currentFolderId'] ?? $rootFolderId);
DiskValidator::assertFolderInsideRoot($currentFolderId, $rootFolderId, $context);

$adapter = new DiskBitrixStorageAdapter($context->currentUserId);

$items = $adapter->listItems($context, $currentFolderId, [
    'sortBy' => $data['sortBy'] ?? 'updatedAt',
    'sortDir' => $data['sortDir'] ?? 'desc',
    'filters' => $data['filters'] ?? [],
]);

$breadcrumbs = $adapter->getBreadcrumbs($context, $currentFolderId);

DiskResponse::success([
    'folder' => [
        'id' => $currentFolderId,
        'name' => 'Текущая папка',
    ],
    'breadcrumbs' => $breadcrumbs,
    'items' => $items,
], [
    'isEmpty' => empty($items),
    'total' => count($items),
]);


---

8. Обновленный actions/create_folder.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1, // TODO: заменить
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::ensureExistsForBlock(
    $context->blockId,
    $context->siteId,
    $context->pageId,
    $context->currentUserId
);

$rootFolderId = DiskRootResolver::resolve($context, $settings);
$permissions = DiskPermissionService::resolve($context, $settings, $rootFolderId);

DiskValidator::assertCan($permissions, 'canCreateFolder');

$currentFolderId = (int)($data['currentFolderId'] ?? 0);
$name = trim((string)($data['name'] ?? ''));

DiskValidator::assertFolderInsideRoot($currentFolderId, $rootFolderId, $context);
DiskValidator::assertNonEmptyString($name, 'EMPTY_FOLDER_NAME');

$adapter = new DiskBitrixStorageAdapter($context->currentUserId);
$folder = $adapter->createFolder($context, $currentFolderId, $name);

DiskResponse::success([
    'folder' => $folder,
]);


---

9. Обновленный actions/upload.php

<?php

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($_POST['siteId'] ?? 0),
    'pageId' => (int)($_POST['pageId'] ?? 0),
    'blockId' => (int)($_POST['blockId'] ?? 0),
    'currentUserId' => 1, // TODO: заменить
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::ensureExistsForBlock(
    $context->blockId,
    $context->siteId,
    $context->pageId,
    $context->currentUserId
);

$rootFolderId = DiskRootResolver::resolve($context, $settings);
$permissions = DiskPermissionService::resolve($context, $settings, $rootFolderId);

DiskValidator::assertCan($permissions, 'canUpload');

$currentFolderId = (int)($_POST['currentFolderId'] ?? 0);
DiskValidator::assertFolderInsideRoot($currentFolderId, $rootFolderId, $context);

$files = $_FILES['files'] ?? null;
if (!$files) {
    throw new RuntimeException('NO_FILES');
}

$normalizedFiles = [];
$count = is_array($files['name']) ? count($files['name']) : 0;

for ($i = 0; $i < $count; $i++) {
    $normalizedFiles[] = [
        'name' => $files['name'][$i],
        'type' => $files['type'][$i],
        'tmp_name' => $files['tmp_name'][$i],
        'error' => $files['error'][$i],
        'size' => $files['size'][$i],
    ];
}

$allowedExtensions = $settings['allowedExtensions'] ?? [];
$maxFileSize = (int)($settings['maxFileSize'] ?? 0);

foreach ($normalizedFiles as $file) {
    if ((int)$file['error'] !== UPLOAD_ERR_OK) {
        throw new RuntimeException('UPLOAD_ERROR');
    }

    if ($maxFileSize > 0 && (int)$file['size'] > $maxFileSize) {
        throw new RuntimeException('FILE_TOO_LARGE');
    }

    if (!empty($allowedExtensions)) {
        $ext = strtolower(pathinfo((string)$file['name'], PATHINFO_EXTENSION));
        if (!in_array($ext, $allowedExtensions, true)) {
            throw new RuntimeException('EXTENSION_NOT_ALLOWED');
        }
    }
}

$adapter = new DiskBitrixStorageAdapter($context->currentUserId);
$result = $adapter->uploadFiles($context, $currentFolderId, $normalizedFiles, $settings);

DiskResponse::success([
    'uploadResult' => $result,
]);


---

10. Обновленный actions/search.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1, // TODO: заменить
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::ensureExistsForBlock(
    $context->blockId,
    $context->siteId,
    $context->pageId,
    $context->currentUserId
);

$rootFolderId = DiskRootResolver::resolve($context, $settings);
$permissions = DiskPermissionService::resolve($context, $settings, $rootFolderId);

DiskValidator::assertCan($permissions, 'canView');

$query = trim((string)($data['query'] ?? ''));
if ($query === '') {
    DiskResponse::success([
        'items' => [],
    ], [
        'total' => 0,
    ]);
}

$adapter = new DiskBitrixStorageAdapter($context->currentUserId);
$items = $adapter->search($context, (int)$rootFolderId, $query, []);

DiskResponse::success([
    'items' => $items,
], [
    'total' => count($items),
]);


---

11. Обновленный actions/rename.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1, // TODO: заменить
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::ensureExistsForBlock(
    $context->blockId,
    $context->siteId,
    $context->pageId,
    $context->currentUserId
);

$rootFolderId = DiskRootResolver::resolve($context, $settings);
$permissions = DiskPermissionService::resolve($context, $settings, $rootFolderId);

DiskValidator::assertCan($permissions, 'canRename');

$entityType = trim((string)($data['entityType'] ?? ''));
$entityId = (int)($data['entityId'] ?? 0);
$newName = trim((string)($data['newName'] ?? ''));

if (!in_array($entityType, ['file', 'folder'], true)) {
    throw new RuntimeException('INVALID_ENTITY_TYPE');
}

if ($entityId <= 0) {
    throw new RuntimeException('INVALID_ENTITY_ID');
}

DiskValidator::assertNonEmptyString($newName, 'EMPTY_NAME');

$adapter = new DiskBitrixStorageAdapter($context->currentUserId);
$item = $adapter->rename($context, $entityType, $entityId, $newName);

DiskResponse::success([
    'item' => $item,
]);


---

12. Обновленный actions/delete.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1, // TODO: заменить
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::ensureExistsForBlock(
    $context->blockId,
    $context->siteId,
    $context->pageId,
    $context->currentUserId
);

$rootFolderId = DiskRootResolver::resolve($context, $settings);
$permissions = DiskPermissionService::resolve($context, $settings, $rootFolderId);

DiskValidator::assertCan($permissions, 'canDelete');

$items = $data['items'] ?? [];
if (!is_array($items) || empty($items)) {
    throw new RuntimeException('EMPTY_ITEMS');
}

$adapter = new DiskBitrixStorageAdapter($context->currentUserId);
$result = $adapter->delete($context, $items);

DiskResponse::success([
    'result' => $result,
]);


---

13. Обновленный actions/move.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1, // TODO: заменить
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::ensureExistsForBlock(
    $context->blockId,
    $context->siteId,
    $context->pageId,
    $context->currentUserId
);

$rootFolderId = DiskRootResolver::resolve($context, $settings);
$permissions = DiskPermissionService::resolve($context, $settings, $rootFolderId);

DiskValidator::assertCan($permissions, 'canRename');

$items = $data['items'] ?? [];
$targetFolderId = (int)($data['targetFolderId'] ?? 0);

if (!is_array($items) || empty($items)) {
    throw new RuntimeException('EMPTY_ITEMS');
}

DiskValidator::assertFolderInsideRoot($targetFolderId, $rootFolderId, $context);

$adapter = new DiskBitrixStorageAdapter($context->currentUserId);
$result = $adapter->move($context, $items, $targetFolderId);

DiskResponse::success([
    'result' => $result,
]);


---

14. Обновленный actions/copy.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1, // TODO: заменить
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::ensureExistsForBlock(
    $context->blockId,
    $context->siteId,
    $context->pageId,
    $context->currentUserId
);

$rootFolderId = DiskRootResolver::resolve($context, $settings);
$permissions = DiskPermissionService::resolve($context, $settings, $rootFolderId);

DiskValidator::assertCan($permissions, 'canView');

$items = $data['items'] ?? [];
$targetFolderId = (int)($data['targetFolderId'] ?? 0);

if (!is_array($items) || empty($items)) {
    throw new RuntimeException('EMPTY_ITEMS');
}

DiskValidator::assertFolderInsideRoot($targetFolderId, $rootFolderId, $context);

$adapter = new DiskBitrixStorageAdapter($context->currentUserId);
$result = $adapter->copy($context, $items, $targetFolderId);

DiskResponse::success([
    'result' => $result,
]);


---

15. Как сделать download правильно

Для Bitrix Disk прямой URL можно брать у файла, но серверная проверка все равно нужна.

actions/download.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? ($_GET['siteId'] ?? 0)),
    'pageId' => (int)($data['pageId'] ?? ($_GET['pageId'] ?? 0)),
    'blockId' => (int)($data['blockId'] ?? ($_GET['blockId'] ?? 0)),
    'currentUserId' => 1, // TODO: заменить
]);

$fileId = (int)($data['fileId'] ?? ($_GET['fileId'] ?? 0));
if ($fileId <= 0) {
    throw new RuntimeException('INVALID_FILE_ID');
}

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::ensureExistsForBlock(
    $context->blockId,
    $context->siteId,
    $context->pageId,
    $context->currentUserId
);

$rootFolderId = DiskRootResolver::resolve($context, $settings);
$permissions = DiskPermissionService::resolve($context, $settings, $rootFolderId);

DiskValidator::assertCan($permissions, 'canDownload');

$adapter = new DiskBitrixStorageAdapter($context->currentUserId);
$url = $adapter->getDownloadUrl($context, $fileId);

LocalRedirect($url);
exit;

Важно: ваш фронт тогда должен отдавать в download не только fileId, но и siteId/pageId/blockId.


---

16. Что надо поправить на фронте

Сейчас downloadUrl у вас в JS может открываться напрямую. Для контекстной проверки лучше формировать URL через ваш API.

То есть вместо прямого file->getDownloadUrl() лучше вернуть, например:

'/local/sitebuilder/components/disk/api.php?action=download'
. '&siteId=' . $context->siteId
. '&pageId=' . $context->pageId
. '&blockId=' . $context->blockId
. '&fileId=' . $file->getId()

Тогда скачивание всегда будет идти через проверку контекста блока.

В normalizeFile() поменяйте downloadUrl на:

'downloadUrl' =>
    '/local/sitebuilder/components/disk/api.php?action=download'
    . '&siteId=' . (int)$context->siteId
    . '&pageId=' . (int)$context->pageId
    . '&blockId=' . (int)$context->blockId
    . '&fileId=' . (int)$file->getId(),


---

17. Что важно знать про права Bitrix Disk

Bitrix Disk сам умеет проверять доступы, но ваш компонент должен проверять дополнительно еще и свой контекст:

этот файл/папка не просто доступны пользователю;

они еще должны относиться к дереву данного блока;

блок должен принадлежать этому siteId/pageId.


То есть безопасность здесь двухслойная:

Слой 1

Ваш sitebuilder-контекст:

siteId

pageId

blockId

rootFolderId


Слой 2

Права самого Bitrix Disk:

SecurityContext

внутренние ACL Disk


Так и нужно делать.


---

18. Что уже умеет этот вариант B

После замены адаптера на Bitrix Disk у вас уже есть рабочая база для:

списка файлов и папок;

открытия папок;

breadcrumbs;

загрузки файлов;

создания папок;

удаления;

переименования;

перемещения;

копирования;

поиска;

скачивания;

реальной проверки “папка внутри root или нет”.



---

19. Что остается доделать обязательно

Сейчас остались 3 важных места.

1. Получение реального currentUserId

Во всех action-файлах заменить:

'currentUserId' => 1

на реального пользователя Bitrix:

global $USER;

$currentUserId = ($USER instanceof CUser) ? (int)$USER->GetID() : 0;

И дальше использовать его.


---

2. Реальная модель ролей сайта

Сейчас DiskPermissionService — ваш внутренний слой прав для блока. Его надо связать с ролью пользователя на сайте.


---

3. Инициализация root папки сайта

Если у сайта еще нет root_disk_folder_id, нужен отдельный helper, который:

создает папку в Bitrix Disk;

сохраняет ее ID в sitebuilder_site.root_disk_folder_id.



---

20. Практический helper для инициализации корня сайта

Ниже полезный стартовый вариант.

lib/SiteDiskInitializer.php

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

        $driver = Driver::getInstance();
        $securityContext = $driver->getFakeSecurityContext($currentUserId);

        $storage = self::resolveDefaultStorage($currentUserId);
        if (!$storage instanceof Storage) {
            throw new RuntimeException('DISK_STORAGE_NOT_FOUND');
        }

        $rootFolder = $storage->getRootObject();
        if (!$rootFolder instanceof Folder) {
            throw new RuntimeException('DISK_STORAGE_ROOT_NOT_FOUND');
        }

        $folderName = $siteName !== '' ? ('Сайт: ' . $siteName) : ('Сайт #' . $siteId);

        $siteFolder = $rootFolder->addSubFolder([
            'NAME' => $folderName,
            'CREATED_BY' => $currentUserId,
        ], $securityContext);

        if (!$siteFolder instanceof Folder) {
            throw new RuntimeException('DISK_SITE_ROOT_CREATE_ERROR');
        }

        SiteRepository::updateRootDiskFolderId($siteId, (int)$siteFolder->getId());

        return (int)$siteFolder->getId();
    }

    protected static function resolveDefaultStorage(int $currentUserId): ?Storage
    {
        $driver = Driver::getInstance();

        // Здесь можно выбрать нужную модель storage:
        // личное хранилище, общее хранилище, storage группы и т.п.
        return $driver->getStorageByUserId($currentUserId);
    }
}


---

21. Что я рекомендую сделать следующим шагом

Следующий реально полезный шаг — уже не общая архитектура, а доводка под эксплуатацию.

Лучший следующий пакет сейчас:

1. полный файл DiskPermissionService.php под роли сайта


2. helper получения текущего пользователя и sessid


3. инициализация корня сайта и корня блока


4. модальное окно настроек блока с сохранением в sitebuilder_disk_settings



Самое полезное сейчас — сделать именно инициализацию root папки + реальный currentUserId + права сайта, чтобы компонент можно было уже почти запускать в вашей системе.