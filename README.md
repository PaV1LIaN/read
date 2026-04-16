Ниже следующая пачка файлов.


---

/local/sitebuilder/components/disk/lib/DiskRootResolver.php

<?php

class DiskRootResolver
{
    public static function resolve(DiskContext $context, array $settings): ?int
    {
        if (!empty($settings['rootFolderId'])) {
            return (int)$settings['rootFolderId'];
        }

        if (!empty($settings['useSiteRootFallback'])) {
            $siteRootFolderId = SiteRepository::getRootDiskFolderId($context->siteId);
            if ($siteRootFolderId !== null && $siteRootFolderId > 0) {
                return $siteRootFolderId;
            }
        }

        return null;
    }

    public static function resolveWithSource(DiskContext $context, array $settings): array
    {
        if (!empty($settings['rootFolderId'])) {
            return [
                'rootFolderId' => (int)$settings['rootFolderId'],
                'source' => 'block',
            ];
        }

        if (!empty($settings['useSiteRootFallback'])) {
            $siteRootFolderId = SiteRepository::getRootDiskFolderId($context->siteId);
            if ($siteRootFolderId !== null && $siteRootFolderId > 0) {
                return [
                    'rootFolderId' => $siteRootFolderId,
                    'source' => 'site',
                ];
            }
        }

        return [
            'rootFolderId' => null,
            'source' => 'none',
        ];
    }
}


---

/local/sitebuilder/components/disk/lib/DiskValidator.php

<?php

class DiskValidator
{
    public static function assertContext(DiskContext $context): void
    {
        if ($context->siteId <= 0) {
            throw new RuntimeException('INVALID_SITE_ID');
        }

        if ($context->pageId <= 0) {
            throw new RuntimeException('INVALID_PAGE_ID');
        }

        if ($context->blockId <= 0) {
            throw new RuntimeException('INVALID_BLOCK_ID');
        }

        if ($context->currentUserId <= 0) {
            throw new RuntimeException('NOT_AUTHORIZED');
        }

        self::assertSiteExists($context->siteId);
        self::assertBlockBelongsToContext($context);
    }

    public static function assertSiteExists(int $siteId): void
    {
        $site = SiteRepository::getById($siteId);
        if (!$site) {
            throw new RuntimeException('SITE_NOT_FOUND');
        }
    }

    public static function assertBlockBelongsToContext(DiskContext $context): void
    {
        $block = BlockRepository::getDiskBlockByContext(
            $context->siteId,
            $context->pageId,
            $context->blockId
        );

        if (!$block) {
            throw new RuntimeException('BLOCK_CONTEXT_MISMATCH');
        }
    }

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

    public static function assertCan(array $permissions, string $permissionKey): void
    {
        if (empty($permissions[$permissionKey])) {
            throw new RuntimeException('ACCESS_DENIED');
        }
    }

    public static function assertNonEmptyString(string $value, string $errorCode): void
    {
        if (trim($value) === '') {
            throw new RuntimeException($errorCode);
        }
    }
}


---

/local/sitebuilder/components/disk/lib/DiskPermissionService.php

<?php

class DiskPermissionService
{
    public static function resolve(DiskContext $context, array $settings, ?int $rootFolderId = null): array
    {
        $rolePermissions = self::resolveRolePermissions($context);
        $blockRestrictions = self::resolveBlockRestrictions($settings);

        return [
            'canView' => $rolePermissions['canView'] && $blockRestrictions['canView'],
            'canUpload' => $rolePermissions['canUpload'] && $blockRestrictions['canUpload'],
            'canCreateFolder' => $rolePermissions['canCreateFolder'] && $blockRestrictions['canCreateFolder'],
            'canRename' => $rolePermissions['canRename'] && $blockRestrictions['canRename'],
            'canDelete' => $rolePermissions['canDelete'] && $blockRestrictions['canDelete'],
            'canDownload' => $rolePermissions['canDownload'] && $blockRestrictions['canDownload'],
            'canManageAccess' => $rolePermissions['canManageAccess'],
            'canEditSettings' => $rolePermissions['canEditSettings'],
        ];
    }

    protected static function resolveRolePermissions(DiskContext $context): array
    {
        if (DiskCurrentUser::isAdmin()) {
            return [
                'canView' => true,
                'canUpload' => true,
                'canCreateFolder' => true,
                'canRename' => true,
                'canDelete' => true,
                'canDownload' => true,
                'canManageAccess' => true,
                'canEditSettings' => true,
            ];
        }

        $role = SiteAccessRepository::getUserRole($context->siteId, $context->currentUserId);

        switch ($role) {
            case 'site_admin':
                return [
                    'canView' => true,
                    'canUpload' => true,
                    'canCreateFolder' => true,
                    'canRename' => true,
                    'canDelete' => true,
                    'canDownload' => true,
                    'canManageAccess' => true,
                    'canEditSettings' => true,
                ];

            case 'site_editor':
                return [
                    'canView' => true,
                    'canUpload' => true,
                    'canCreateFolder' => true,
                    'canRename' => true,
                    'canDelete' => true,
                    'canDownload' => true,
                    'canManageAccess' => false,
                    'canEditSettings' => true,
                ];

            case 'site_user':
                return [
                    'canView' => true,
                    'canUpload' => true,
                    'canCreateFolder' => false,
                    'canRename' => false,
                    'canDelete' => false,
                    'canDownload' => true,
                    'canManageAccess' => false,
                    'canEditSettings' => false,
                ];

            case 'site_viewer':
                return [
                    'canView' => true,
                    'canUpload' => false,
                    'canCreateFolder' => false,
                    'canRename' => false,
                    'canDelete' => false,
                    'canDownload' => true,
                    'canManageAccess' => false,
                    'canEditSettings' => false,
                ];

            default:
                return [
                    'canView' => false,
                    'canUpload' => false,
                    'canCreateFolder' => false,
                    'canRename' => false,
                    'canDelete' => false,
                    'canDownload' => false,
                    'canManageAccess' => false,
                    'canEditSettings' => false,
                ];
        }
    }

    protected static function resolveBlockRestrictions(array $settings): array
    {
        return [
            'canView' => true,
            'canUpload' => !empty($settings['allowUpload']),
            'canCreateFolder' => !empty($settings['allowCreateFolder']),
            'canRename' => !empty($settings['allowRename']),
            'canDelete' => !empty($settings['allowDelete']),
            'canDownload' => !empty($settings['allowDownload']),
        ];
    }
}


---

/local/sitebuilder/components/disk/lib/SiteDiskInitializer.php

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

        $folderName = $siteName !== ''
            ? ('Сайт: ' . $siteName)
            : ('Сайт: ' . (string)$site['name']);

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

        $storage = $driver->getStorageByUserId($currentUserId);
        if ($storage instanceof Storage) {
            return $storage;
        }

        return null;
    }
}


---

/local/sitebuilder/components/disk/lib/BlockDiskInitializer.php

<?php

use Bitrix\Disk\Folder;
use Bitrix\Disk\Driver;

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
        ], Driver::getInstance()->getFakeSecurityContext($currentUserId));

        if (!$created instanceof Folder) {
            throw new RuntimeException('BLOCK_ROOT_CREATE_ERROR');
        }

        DiskSettingsRepository::save($blockId, [
            'rootFolderId' => (int)$created->getId(),
            'useSiteRootFallback' => true,
        ]);

        return (int)$created->getId();
    }
}


---

/local/sitebuilder/components/disk/lib/DiskStorageAdapterInterface.php

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

/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php

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

        return '/local/sitebuilder/components/disk/api.php?action=download'
            . '&siteId=' . (int)$context->siteId
            . '&pageId=' . (int)$context->pageId
            . '&blockId=' . (int)$context->blockId
            . '&fileId=' . (int)$file->getId();
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
            'downloadUrl' => $this->getDownloadUrl($context, (int)$file->getId()),
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

Следом пришлю все actions/*.php.