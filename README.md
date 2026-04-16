Ниже даю стартовый комплект файлов компонента “Диск” в практически готовом виде.

Я сделаю набор так, чтобы:

его можно было положить в /local/sitebuilder/components/disk/;

он уже был похож на реальный компонент;

дальше можно было спокойно подключать вашу БД, Bitrix Disk или другое хранилище;

структура соответствовала вашему sitebuilder.



---

1. class.php

<?php

require_once __DIR__ . '/bootstrap.php';

class SitebuilderDiskComponent
{
    protected array $params = [];
    protected array $result = [];

    public function __construct(array $params = [])
    {
        $this->params = $params;
    }

    public function execute(): void
    {
        try {
            $context = DiskContextFactory::fromArray([
                'siteId' => (int)($this->params['SITE_ID'] ?? 0),
                'pageId' => (int)($this->params['PAGE_ID'] ?? 0),
                'blockId' => (int)($this->params['BLOCK_ID'] ?? 0),
                'currentUserId' => (int)($this->params['CURRENT_USER_ID'] ?? 0),
            ]);

            DiskValidator::assertContext($context);

            $settings = DiskSettingsRepository::getByBlockId($context->blockId);
            $rootFolderId = DiskRootResolver::resolve($context, $settings);
            $permissions = DiskPermissionService::resolve($context, $settings, $rootFolderId);

            $this->result = [
                'SITE_ID' => $context->siteId,
                'PAGE_ID' => $context->pageId,
                'BLOCK_ID' => $context->blockId,
                'CURRENT_USER_ID' => $context->currentUserId,
                'SETTINGS' => $settings,
                'ROOT_FOLDER_ID' => $rootFolderId,
                'PERMISSIONS' => $permissions,
                'TITLE' => $settings['title'] ?? 'Файлы',
                'INITIAL_STATE' => [
                    'siteId' => $context->siteId,
                    'pageId' => $context->pageId,
                    'blockId' => $context->blockId,
                    'rootFolderId' => $rootFolderId,
                    'currentFolderId' => $rootFolderId,
                    'settings' => $settings,
                    'permissions' => $permissions,
                ],
                'ERROR' => null,
            ];
        } catch (Throwable $e) {
            $this->result = [
                'SITE_ID' => (int)($this->params['SITE_ID'] ?? 0),
                'PAGE_ID' => (int)($this->params['PAGE_ID'] ?? 0),
                'BLOCK_ID' => (int)($this->params['BLOCK_ID'] ?? 0),
                'CURRENT_USER_ID' => (int)($this->params['CURRENT_USER_ID'] ?? 0),
                'SETTINGS' => [],
                'ROOT_FOLDER_ID' => null,
                'PERMISSIONS' => [
                    'canView' => false,
                    'canUpload' => false,
                    'canCreateFolder' => false,
                    'canRename' => false,
                    'canDelete' => false,
                    'canDownload' => false,
                    'canManageAccess' => false,
                    'canEditSettings' => false,
                ],
                'TITLE' => 'Файлы',
                'INITIAL_STATE' => [
                    'siteId' => (int)($this->params['SITE_ID'] ?? 0),
                    'pageId' => (int)($this->params['PAGE_ID'] ?? 0),
                    'blockId' => (int)($this->params['BLOCK_ID'] ?? 0),
                    'rootFolderId' => null,
                    'currentFolderId' => null,
                    'settings' => [],
                    'permissions' => [
                        'canView' => false,
                        'canUpload' => false,
                        'canCreateFolder' => false,
                        'canRename' => false,
                        'canDelete' => false,
                        'canDownload' => false,
                        'canManageAccess' => false,
                        'canEditSettings' => false,
                    ],
                ],
                'ERROR' => $e->getMessage(),
            ];
        }

        $arResult = $this->result;
        include __DIR__ . '/template.php';
    }
}


---

2. bootstrap.php

<?php

if (!defined('SITEBUILDER_DISK_BOOTSTRAP')) {
    define('SITEBUILDER_DISK_BOOTSTRAP', true);

    require_once __DIR__ . '/lib/helpers.php';
    require_once __DIR__ . '/lib/DiskContext.php';
    require_once __DIR__ . '/lib/DiskResponse.php';
    require_once __DIR__ . '/lib/DiskValidator.php';
    require_once __DIR__ . '/lib/DiskSettingsRepository.php';
    require_once __DIR__ . '/lib/DiskRootResolver.php';
    require_once __DIR__ . '/lib/DiskPermissionService.php';
    require_once __DIR__ . '/lib/DiskStorageAdapterInterface.php';
    require_once __DIR__ . '/lib/DiskStorageAdapter.php';
}


---

3. lib/helpers.php

<?php

function disk_h(?string $value): string
{
    return htmlspecialchars((string)$value, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
}

function disk_read_json_body(): array
{
    $raw = file_get_contents('php://input');
    if ($raw === false || $raw === '') {
        return [];
    }

    $data = json_decode($raw, true);
    return is_array($data) ? $data : [];
}

function disk_normalize_bool($value): bool
{
    if (is_bool($value)) {
        return $value;
    }

    if (is_int($value)) {
        return $value === 1;
    }

    if (is_string($value)) {
        $value = strtolower(trim($value));
        return in_array($value, ['1', 'y', 'yes', 'true', 'on'], true);
    }

    return false;
}


---

4. lib/DiskContext.php

<?php

class DiskContext
{
    public int $siteId;
    public int $pageId;
    public int $blockId;
    public int $currentUserId;

    public function __construct(int $siteId, int $pageId, int $blockId, int $currentUserId)
    {
        $this->siteId = $siteId;
        $this->pageId = $pageId;
        $this->blockId = $blockId;
        $this->currentUserId = $currentUserId;
    }

    public function toArray(): array
    {
        return [
            'siteId' => $this->siteId,
            'pageId' => $this->pageId,
            'blockId' => $this->blockId,
            'currentUserId' => $this->currentUserId,
        ];
    }
}

class DiskContextFactory
{
    public static function fromArray(array $data): DiskContext
    {
        return new DiskContext(
            (int)($data['siteId'] ?? 0),
            (int)($data['pageId'] ?? 0),
            (int)($data['blockId'] ?? 0),
            (int)($data['currentUserId'] ?? 0)
        );
    }
}


---

5. lib/DiskResponse.php

<?php

class DiskResponse
{
    public static function success(array $data = [], array $meta = []): void
    {
        self::send([
            'ok' => true,
            'data' => $data,
            'meta' => $meta,
            'error' => null,
            'message' => '',
        ]);
    }

    public static function error(string $errorCode, string $message = '', array $details = []): void
    {
        self::send([
            'ok' => false,
            'data' => [],
            'meta' => [],
            'error' => $errorCode,
            'message' => $message,
            'details' => $details,
        ]);
    }

    protected static function send(array $payload): void
    {
        header('Content-Type: application/json; charset=UTF-8');
        echo json_encode($payload, JSON_UNESCAPED_UNICODE);
        exit;
    }
}


---

6. lib/DiskValidator.php

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

        self::assertBlockBelongsToContext($context);
    }

    public static function assertBlockBelongsToContext(DiskContext $context): void
    {
        // TODO:
        // Проверить в БД, что:
        // - блок существует
        // - блок принадлежит siteId
        // - блок принадлежит pageId
        // - тип блока = disk
        $ok = true;

        if (!$ok) {
            throw new RuntimeException('BLOCK_CONTEXT_MISMATCH');
        }
    }

    public static function assertFolderInsideRoot(int $folderId, ?int $rootFolderId): void
    {
        if ($folderId <= 0) {
            throw new RuntimeException('INVALID_FOLDER_ID');
        }

        if ($rootFolderId === null || $rootFolderId <= 0) {
            throw new RuntimeException('ROOT_FOLDER_NOT_RESOLVED');
        }

        // TODO:
        // Здесь обязательна реальная проверка,
        // что folderId лежит внутри дерева rootFolderId.
        $ok = true;

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

7. lib/DiskSettingsRepository.php

<?php

class DiskSettingsRepository
{
    public static function getByBlockId(int $blockId): array
    {
        // TODO:
        // Заменить на реальный SELECT из sitebuilder_disk_settings

        return [
            'block_id' => $blockId,
            'title' => 'Файлы',
            'rootFolderId' => null,
            'viewMode' => 'table',
            'allowUpload' => true,
            'allowCreateFolder' => true,
            'allowRename' => true,
            'allowDelete' => false,
            'allowDownload' => true,
            'showSearch' => true,
            'showBreadcrumbs' => true,
            'defaultSort' => 'updatedAt',
            'defaultSortDirection' => 'desc',
            'allowedExtensions' => [],
            'maxFileSize' => 52428800,
            'permissionMode' => 'inherit_site',
            'useSiteRootFallback' => true,
        ];
    }

    public static function createDefault(array $data): bool
    {
        // TODO: INSERT
        return true;
    }

    public static function save(int $blockId, array $settings): bool
    {
        // TODO: UPDATE
        return true;
    }
}


---

8. lib/DiskRootResolver.php

<?php

class DiskRootResolver
{
    public static function resolve(DiskContext $context, array $settings): ?int
    {
        if (!empty($settings['rootFolderId'])) {
            return (int)$settings['rootFolderId'];
        }

        if (!empty($settings['useSiteRootFallback'])) {
            $siteRootFolderId = self::getSiteRootFolderId($context->siteId);
            if ($siteRootFolderId > 0) {
                return (int)$siteRootFolderId;
            }
        }

        return null;
    }

    protected static function getSiteRootFolderId(int $siteId): ?int
    {
        // TODO:
        // Подтянуть rootDiskFolderId из таблицы site / sitebuilder_site_disk
        return null;
    }
}


---

9. lib/DiskPermissionService.php

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
        // TODO:
        // Реально определить роль пользователя на уровне сайта.
        // Пока временно считаем пользователя редактором сайта.
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

10. lib/DiskStorageAdapterInterface.php

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
}


---

11. lib/DiskStorageAdapter.php

<?php

class DiskStorageAdapter implements DiskStorageAdapterInterface
{
    public function listItems(DiskContext $context, int $folderId, array $options = []): array
    {
        // TODO:
        // Здесь должна быть реальная работа с хранилищем.
        // Пока тестовые данные.
        return [
            [
                'id' => 1001,
                'entityType' => 'folder',
                'name' => 'Документы',
                'extension' => null,
                'mimeType' => null,
                'size' => null,
                'updatedAt' => '2026-04-16 10:00:00',
                'downloadUrl' => null,
            ],
            [
                'id' => 1002,
                'entityType' => 'folder',
                'name' => 'Шаблоны',
                'extension' => null,
                'mimeType' => null,
                'size' => null,
                'updatedAt' => '2026-04-15 18:40:00',
                'downloadUrl' => null,
            ],
            [
                'id' => 2001,
                'entityType' => 'file',
                'name' => 'report.pdf',
                'extension' => 'pdf',
                'mimeType' => 'application/pdf',
                'size' => 124000,
                'updatedAt' => '2026-04-16 09:55:00',
                'downloadUrl' => '/local/sitebuilder/components/disk/api.php?action=download&fileId=2001',
            ],
        ];
    }

    public function createFolder(DiskContext $context, int $parentFolderId, string $name): array
    {
        return [
            'id' => random_int(5000, 9999),
            'entityType' => 'folder',
            'name' => $name,
        ];
    }

    public function uploadFiles(DiskContext $context, int $folderId, array $files, array $options = []): array
    {
        return [
            'uploaded' => count($files),
            'items' => [],
        ];
    }

    public function rename(DiskContext $context, string $entityType, int $entityId, string $newName): array
    {
        return [
            'id' => $entityId,
            'entityType' => $entityType,
            'name' => $newName,
        ];
    }

    public function delete(DiskContext $context, array $items): array
    {
        return [
            'deleted' => count($items),
        ];
    }

    public function move(DiskContext $context, array $items, int $targetFolderId): array
    {
        return [
            'moved' => count($items),
            'targetFolderId' => $targetFolderId,
        ];
    }

    public function copy(DiskContext $context, array $items, int $targetFolderId): array
    {
        return [
            'copied' => count($items),
            'targetFolderId' => $targetFolderId,
        ];
    }

    public function search(DiskContext $context, int $rootFolderId, string $query, array $options = []): array
    {
        $all = $this->listItems($context, $rootFolderId, $options);
        $query = mb_strtolower(trim($query));

        if ($query === '') {
            return [];
        }

        return array_values(array_filter($all, static function ($item) use ($query) {
            return mb_stripos((string)($item['name'] ?? ''), $query) !== false;
        }));
    }

    public function getBreadcrumbs(DiskContext $context, int $folderId): array
    {
        return [
            ['id' => $folderId, 'name' => 'Корень'],
        ];
    }

    public function getDownloadUrl(DiskContext $context, int $fileId): string
    {
        return '/local/sitebuilder/components/disk/api.php?action=download&fileId=' . $fileId;
    }
}


---

12. api.php

<?php

require_once __DIR__ . '/bootstrap.php';

try {
    $action = $_GET['action'] ?? '';

    switch ($action) {
        case 'resolveRoot':
            require __DIR__ . '/actions/resolve_root.php';
            break;

        case 'getSettings':
            require __DIR__ . '/actions/get_settings.php';
            break;

        case 'saveSettings':
            require __DIR__ . '/actions/save_settings.php';
            break;

        case 'getPermissions':
            require __DIR__ . '/actions/get_permissions.php';
            break;

        case 'list':
            require __DIR__ . '/actions/list.php';
            break;

        case 'upload':
            require __DIR__ . '/actions/upload.php';
            break;

        case 'createFolder':
            require __DIR__ . '/actions/create_folder.php';
            break;

        case 'rename':
            require __DIR__ . '/actions/rename.php';
            break;

        case 'delete':
            require __DIR__ . '/actions/delete.php';
            break;

        case 'move':
            require __DIR__ . '/actions/move.php';
            break;

        case 'copy':
            require __DIR__ . '/actions/copy.php';
            break;

        case 'search':
            require __DIR__ . '/actions/search.php';
            break;

        case 'download':
            require __DIR__ . '/actions/download.php';
            break;

        default:
            DiskResponse::error('UNKNOWN_ACTION', 'Неизвестное действие');
    }
} catch (Throwable $e) {
    DiskResponse::error('SERVER_ERROR', $e->getMessage());
}


---

13. actions/resolve_root.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1, // TODO: заменить на реального пользователя
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::getByBlockId($context->blockId);
$rootFolderId = DiskRootResolver::resolve($context, $settings);

DiskResponse::success([
    'rootFolderId' => $rootFolderId,
    'source' => !empty($settings['rootFolderId']) ? 'block' : 'site',
]);


---

14. actions/get_settings.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1,
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::getByBlockId($context->blockId);

DiskResponse::success([
    'settings' => $settings,
]);


---

15. actions/get_permissions.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1,
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::getByBlockId($context->blockId);
$rootFolderId = DiskRootResolver::resolve($context, $settings);
$permissions = DiskPermissionService::resolve($context, $settings, $rootFolderId);

DiskResponse::success([
    'permissions' => $permissions,
]);


---

16. actions/list.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1,
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::getByBlockId($context->blockId);
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
DiskValidator::assertFolderInsideRoot($currentFolderId, $rootFolderId);

$adapter = new DiskStorageAdapter();

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

17. actions/create_folder.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1,
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::getByBlockId($context->blockId);
$rootFolderId = DiskRootResolver::resolve($context, $settings);
$permissions = DiskPermissionService::resolve($context, $settings, $rootFolderId);

DiskValidator::assertCan($permissions, 'canCreateFolder');

$currentFolderId = (int)($data['currentFolderId'] ?? 0);
$name = trim((string)($data['name'] ?? ''));

DiskValidator::assertFolderInsideRoot($currentFolderId, $rootFolderId);
DiskValidator::assertNonEmptyString($name, 'EMPTY_FOLDER_NAME');

$adapter = new DiskStorageAdapter();
$folder = $adapter->createFolder($context, $currentFolderId, $name);

DiskResponse::success([
    'folder' => $folder,
]);


---

18. actions/upload.php

<?php

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($_POST['siteId'] ?? 0),
    'pageId' => (int)($_POST['pageId'] ?? 0),
    'blockId' => (int)($_POST['blockId'] ?? 0),
    'currentUserId' => 1,
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::getByBlockId($context->blockId);
$rootFolderId = DiskRootResolver::resolve($context, $settings);
$permissions = DiskPermissionService::resolve($context, $settings, $rootFolderId);

DiskValidator::assertCan($permissions, 'canUpload');

$currentFolderId = (int)($_POST['currentFolderId'] ?? 0);
DiskValidator::assertFolderInsideRoot($currentFolderId, $rootFolderId);

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

$adapter = new DiskStorageAdapter();
$result = $adapter->uploadFiles($context, $currentFolderId, $normalizedFiles, $settings);

DiskResponse::success([
    'uploadResult' => $result,
]);


---

19. actions/search.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1,
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::getByBlockId($context->blockId);
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

$adapter = new DiskStorageAdapter();
$items = $adapter->search($context, (int)$rootFolderId, $query, []);

DiskResponse::success([
    'items' => $items,
], [
    'total' => count($items),
]);


---

20. actions/rename.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1,
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::getByBlockId($context->blockId);
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

$adapter = new DiskStorageAdapter();
$item = $adapter->rename($context, $entityType, $entityId, $newName);

DiskResponse::success([
    'item' => $item,
]);


---

21. actions/delete.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1,
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::getByBlockId($context->blockId);
$rootFolderId = DiskRootResolver::resolve($context, $settings);
$permissions = DiskPermissionService::resolve($context, $settings, $rootFolderId);

DiskValidator::assertCan($permissions, 'canDelete');

$items = $data['items'] ?? [];
if (!is_array($items) || empty($items)) {
    throw new RuntimeException('EMPTY_ITEMS');
}

$adapter = new DiskStorageAdapter();
$result = $adapter->delete($context, $items);

DiskResponse::success([
    'result' => $result,
]);


---

22. actions/move.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1,
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::getByBlockId($context->blockId);
$rootFolderId = DiskRootResolver::resolve($context, $settings);
$permissions = DiskPermissionService::resolve($context, $settings, $rootFolderId);

DiskValidator::assertCan($permissions, 'canRename');

$items = $data['items'] ?? [];
$targetFolderId = (int)($data['targetFolderId'] ?? 0);

if (!is_array($items) || empty($items)) {
    throw new RuntimeException('EMPTY_ITEMS');
}

DiskValidator::assertFolderInsideRoot($targetFolderId, $rootFolderId);

$adapter = new DiskStorageAdapter();
$result = $adapter->move($context, $items, $targetFolderId);

DiskResponse::success([
    'result' => $result,
]);


---

23. actions/copy.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1,
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::getByBlockId($context->blockId);
$rootFolderId = DiskRootResolver::resolve($context, $settings);
$permissions = DiskPermissionService::resolve($context, $settings, $rootFolderId);

DiskValidator::assertCan($permissions, 'canView');

$items = $data['items'] ?? [];
$targetFolderId = (int)($data['targetFolderId'] ?? 0);

if (!is_array($items) || empty($items)) {
    throw new RuntimeException('EMPTY_ITEMS');
}

DiskValidator::assertFolderInsideRoot($targetFolderId, $rootFolderId);

$adapter = new DiskStorageAdapter();
$result = $adapter->copy($context, $items, $targetFolderId);

DiskResponse::success([
    'result' => $result,
]);


---

24. actions/save_settings.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1,
]);

DiskValidator::assertContext($context);

$currentSettings = DiskSettingsRepository::getByBlockId($context->blockId);
$rootFolderId = DiskRootResolver::resolve($context, $currentSettings);
$permissions = DiskPermissionService::resolve($context, $currentSettings, $rootFolderId);

DiskValidator::assertCan($permissions, 'canEditSettings');

$settings = $data['settings'] ?? [];
if (!is_array($settings)) {
    throw new RuntimeException('INVALID_SETTINGS_PAYLOAD');
}

$normalized = [
    'title' => trim((string)($settings['title'] ?? 'Файлы')),
    'rootFolderId' => isset($settings['rootFolderId']) ? (int)$settings['rootFolderId'] : null,
    'viewMode' => in_array(($settings['viewMode'] ?? 'table'), ['table', 'grid'], true) ? $settings['viewMode'] : 'table',
    'allowUpload' => disk_normalize_bool($settings['allowUpload'] ?? true),
    'allowCreateFolder' => disk_normalize_bool($settings['allowCreateFolder'] ?? true),
    'allowRename' => disk_normalize_bool($settings['allowRename'] ?? true),
    'allowDelete' => disk_normalize_bool($settings['allowDelete'] ?? false),
    'allowDownload' => disk_normalize_bool($settings['allowDownload'] ?? true),
    'showSearch' => disk_normalize_bool($settings['showSearch'] ?? true),
    'showBreadcrumbs' => disk_normalize_bool($settings['showBreadcrumbs'] ?? true),
    'defaultSort' => trim((string)($settings['defaultSort'] ?? 'updatedAt')),
    'defaultSortDirection' => trim((string)($settings['defaultSortDirection'] ?? 'desc')),
    'allowedExtensions' => is_array($settings['allowedExtensions'] ?? null) ? array_values($settings['allowedExtensions']) : [],
    'maxFileSize' => (int)($settings['maxFileSize'] ?? 52428800),
    'permissionMode' => trim((string)($settings['permissionMode'] ?? 'inherit_site')),
    'useSiteRootFallback' => disk_normalize_bool($settings['useSiteRootFallback'] ?? true),
];

DiskSettingsRepository::save($context->blockId, $normalized);

DiskResponse::success([
    'settings' => $normalized,
]);


---

25. actions/download.php

Пока как заглушка, чтобы было понятно место логики:

<?php

$fileId = (int)($_GET['fileId'] ?? 0);
if ($fileId <= 0) {
    throw new RuntimeException('INVALID_FILE_ID');
}

// TODO:
// Здесь должна быть полноценная логика:
// - определить пользователя
// - определить контекст блока
// - проверить canDownload
// - проверить принадлежность файла root-дереву
// - отдать файл

header('Content-Type: text/plain; charset=UTF-8');
echo 'TODO: download file #' . $fileId;
exit;


---

26. template.php

<?php
$initialStateJson = disk_h(json_encode($arResult['INITIAL_STATE'], JSON_UNESCAPED_UNICODE));
?>
<div class="sb-disk"
     id="sb-disk-<?= (int)$arResult['BLOCK_ID'] ?>"
     data-site-id="<?= (int)$arResult['SITE_ID'] ?>"
     data-page-id="<?= (int)$arResult['PAGE_ID'] ?>"
     data-block-id="<?= (int)$arResult['BLOCK_ID'] ?>"
     data-initial-state="<?= $initialStateJson ?>">

    <div class="sb-disk__header">
        <div class="sb-disk__header-main">
            <div class="sb-disk__title-wrap">
                <h3 class="sb-disk__title"><?= disk_h($arResult['TITLE']) ?></h3>
                <div class="sb-disk__subtitle" data-role="subtitle"></div>
            </div>
        </div>

        <div class="sb-disk__header-actions">
            <button type="button" class="sb-disk__btn sb-disk__btn--ghost" data-action="refresh">Обновить</button>

            <?php if (!empty($arResult['PERMISSIONS']['canEditSettings'])): ?>
                <button type="button" class="sb-disk__btn sb-disk__btn--ghost" data-action="settings">Настройки</button>
            <?php endif; ?>
        </div>
    </div>

    <?php if (!empty($arResult['SETTINGS']['showBreadcrumbs'])): ?>
        <div class="sb-disk__breadcrumbs" data-role="breadcrumbs"></div>
    <?php endif; ?>

    <div class="sb-disk__toolbar">
        <div class="sb-disk__toolbar-left">
            <?php if (!empty($arResult['SETTINGS']['showSearch'])): ?>
                <div class="sb-disk__search">
                    <input type="text"
                           class="sb-disk__search-input"
                           data-role="search-input"
                           placeholder="Поиск файлов и папок">
                </div>
            <?php endif; ?>

            <select class="sb-disk__select" data-role="sort-select">
                <option value="updatedAt:desc">Сначала новые</option>
                <option value="updatedAt:asc">Сначала старые</option>
                <option value="name:asc">По имени А–Я</option>
                <option value="name:desc">По имени Я–А</option>
                <option value="size:desc">По размеру</option>
            </select>
        </div>

        <div class="sb-disk__toolbar-right">
            <?php if (!empty($arResult['PERMISSIONS']['canUpload'])): ?>
                <button type="button" class="sb-disk__btn" data-action="upload">Загрузить</button>
            <?php endif; ?>

            <?php if (!empty($arResult['PERMISSIONS']['canCreateFolder'])): ?>
                <button type="button" class="sb-disk__btn" data-action="create-folder">Новая папка</button>
            <?php endif; ?>

            <div class="sb-disk__view-switch">
                <button type="button" class="sb-disk__view-btn is-active" data-view="table">Таблица</button>
                <button type="button" class="sb-disk__view-btn" data-view="grid">Плитка</button>
            </div>
        </div>
    </div>

    <div class="sb-disk__bulkbar" data-role="bulkbar" hidden>
        <span class="sb-disk__bulkbar-text" data-role="bulkbar-text">Выбрано: 0</span>
        <div class="sb-disk__bulkbar-actions">
            <button type="button" class="sb-disk__btn" data-action="download-selected">Скачать</button>
            <button type="button" class="sb-disk__btn sb-disk__btn--danger" data-action="delete-selected">Удалить</button>
        </div>
    </div>

    <div class="sb-disk__content">
        <div class="sb-disk__state" data-state="loading" hidden>Загрузка...</div>
        <div class="sb-disk__state" data-state="empty" hidden>Здесь пока нет файлов и папок.</div>
        <div class="sb-disk__state" data-state="error" hidden>Не удалось загрузить содержимое.</div>
        <div class="sb-disk__state" data-state="no-access" hidden>У вас нет доступа к этому разделу.</div>
        <div class="sb-disk__state" data-state="no-root" hidden>Для блока не настроена корневая папка.</div>

        <div class="sb-disk__view sb-disk__view--table" data-view-container="table">
            <table class="sb-disk__table">
                <thead>
                    <tr>
                        <th class="sb-disk__col sb-disk__col--checkbox">
                            <input type="checkbox" data-role="select-all">
                        </th>
                        <th class="sb-disk__col sb-disk__col--name">Название</th>
                        <th class="sb-disk__col">Тип</th>
                        <th class="sb-disk__col">Размер</th>
                        <th class="sb-disk__col">Изменен</th>
                        <th class="sb-disk__col sb-disk__col--actions"></th>
                    </tr>
                </thead>
                <tbody data-role="items-table"></tbody>
            </table>
        </div>

        <div class="sb-disk__view sb-disk__view--grid" data-view-container="grid" hidden></div>
    </div>

    <input type="file" class="sb-disk__file-input" data-role="upload-input" multiple hidden>
</div>


---

27. styles.css

.sb-disk {
    display: flex;
    flex-direction: column;
    gap: 16px;
    padding: 20px;
    background: #ffffff;
    border: 1px solid #dfe3ea;
    border-radius: 18px;
    min-height: 280px;
    box-sizing: border-box;
}

.sb-disk__header,
.sb-disk__toolbar,
.sb-disk__bulkbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
}

.sb-disk__header-main,
.sb-disk__header-actions,
.sb-disk__toolbar-left,
.sb-disk__toolbar-right,
.sb-disk__bulkbar-actions {
    display: flex;
    align-items: center;
    gap: 12px;
}

.sb-disk__title-wrap {
    display: flex;
    flex-direction: column;
    gap: 4px;
}

.sb-disk__title {
    margin: 0;
    font-size: 20px;
    line-height: 1.2;
    font-weight: 600;
    color: #111827;
}

.sb-disk__subtitle {
    font-size: 13px;
    line-height: 1.4;
    color: #6b7280;
}

.sb-disk__breadcrumbs {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 8px;
    min-height: 20px;
    font-size: 13px;
    line-height: 1.4;
}

.sb-disk__crumb {
    appearance: none;
    border: none;
    background: transparent;
    padding: 0;
    cursor: pointer;
    color: #2563eb;
    font-size: 13px;
    line-height: 1.4;
}

.sb-disk__crumb:hover {
    text-decoration: underline;
}

.sb-disk__search-input,
.sb-disk__select {
    height: 40px;
    border: 1px solid #d1d5db;
    border-radius: 10px;
    background: #ffffff;
    padding: 0 12px;
    font-size: 14px;
    line-height: 1;
    box-sizing: border-box;
}

.sb-disk__search-input {
    min-width: 260px;
}

.sb-disk__btn,
.sb-disk__view-btn {
    appearance: none;
    height: 40px;
    padding: 0 14px;
    border: 1px solid #d1d5db;
    border-radius: 10px;
    background: #ffffff;
    color: #111827;
    font-size: 14px;
    line-height: 1;
    cursor: pointer;
    box-sizing: border-box;
}

.sb-disk__btn:hover,
.sb-disk__view-btn:hover {
    background: #f8fafc;
}

.sb-disk__btn:disabled,
.sb-disk__view-btn:disabled {
    opacity: .5;
    cursor: default;
}

.sb-disk__btn--ghost {
    background: #f8fafc;
}

.sb-disk__btn--danger {
    border-color: #ef4444;
    color: #b91c1c;
}

.sb-disk__view-switch {
    display: inline-flex;
    align-items: center;
    gap: 8px;
}

.sb-disk__view-btn.is-active {
    font-weight: 600;
    border-color: #93c5fd;
    background: #eff6ff;
}

.sb-disk__bulkbar {
    padding: 12px 14px;
    border: 1px solid #dbeafe;
    border-radius: 12px;
    background: #f8fbff;
}

.sb-disk__bulkbar-text {
    font-size: 14px;
    color: #1f2937;
}

.sb-disk__content {
    display: block;
    min-height: 220px;
}

.sb-disk__state {
    padding: 40px 20px;
    border: 1px dashed #d1d5db;
    border-radius: 12px;
    text-align: center;
    color: #6b7280;
    font-size: 14px;
    line-height: 1.5;
}

.sb-disk__table {
    width: 100%;
    border-collapse: collapse;
    table-layout: fixed;
}

.sb-disk__table thead th {
    padding: 12px 10px;
    border-bottom: 1px solid #e5e7eb;
    color: #6b7280;
    font-size: 13px;
    line-height: 1.4;
    font-weight: 500;
    text-align: left;
}

.sb-disk__table tbody td {
    padding: 12px 10px;
    border-bottom: 1px solid #eef2f7;
    color: #111827;
    font-size: 14px;
    line-height: 1.4;
    vertical-align: middle;
}

.sb-disk__col--checkbox {
    width: 44px;
}

.sb-disk__col--actions {
    width: 110px;
}

.sb-disk__row.is-selected {
    background: #f5f9ff;
}

.sb-disk__item-name {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    min-width: 0;
}

.sb-disk__item-name-label {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.sb-disk__badge {
    display: inline-flex;
    align-items: center;
    height: 22px;
    padding: 0 8px;
    border-radius: 999px;
    background: #f3f4f6;
    color: #374151;
    font-size: 12px;
    line-height: 1;
}

.sb-disk__grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
    gap: 16px;
}

.sb-disk__card {
    display: flex;
    flex-direction: column;
    gap: 12px;
    min-height: 150px;
    padding: 14px;
    border: 1px solid #e5e7eb;
    border-radius: 14px;
    background: #ffffff;
    box-sizing: border-box;
}

.sb-disk__card.is-selected {
    border-color: #93c5fd;
    background: #f8fbff;
}

.sb-disk__card-top,
.sb-disk__card-meta,
.sb-disk__card-actions {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
}

.sb-disk__card-name {
    font-size: 14px;
    font-weight: 500;
    line-height: 1.4;
    color: #111827;
    word-break: break-word;
}

.sb-disk__card-sub {
    font-size: 12px;
    line-height: 1.4;
    color: #6b7280;
}

.sb-disk__row-btn {
    appearance: none;
    border: 1px solid #d1d5db;
    background: #ffffff;
    border-radius: 8px;
    padding: 6px 10px;
    cursor: pointer;
    font-size: 13px;
}

.sb-disk__row-btn:hover {
    background: #f8fafc;
}

.sb-disk__file-input {
    display: none;
}

.is-hidden {
    display: none !important;
}

@media (max-width: 980px) {
    .sb-disk__header,
    .sb-disk__toolbar,
    .sb-disk__bulkbar {
        flex-direction: column;
        align-items: stretch;
    }

    .sb-disk__toolbar-left,
    .sb-disk__toolbar-right,
    .sb-disk__header-actions,
    .sb-disk__bulkbar-actions {
        flex-wrap: wrap;
    }

    .sb-disk__search-input {
        min-width: 100%;
        width: 100%;
    }
}


---

28. script.js

(function () {
  function DiskComponent(root) {
    this.root = root;
    this.state = this.readInitialState();
  }

  DiskComponent.prototype.readInitialState = function () {
    var raw = this.root.getAttribute('data-initial-state') || '{}';
    var parsed = {};

    try {
      parsed = JSON.parse(raw);
    } catch (e) {
      parsed = {};
    }

    return {
      siteId: Number(parsed.siteId || this.root.dataset.siteId || 0),
      pageId: Number(parsed.pageId || this.root.dataset.pageId || 0),
      blockId: Number(parsed.blockId || this.root.dataset.blockId || 0),
      rootFolderId: parsed.rootFolderId || null,
      currentFolderId: parsed.currentFolderId || null,
      settings: parsed.settings || {},
      permissions: parsed.permissions || {},
      breadcrumbs: [],
      items: [],
      selectedIds: [],
      searchQuery: '',
      viewMode: (parsed.settings && parsed.settings.viewMode) || 'table',
      loading: false,
      error: null
    };
  };

  DiskComponent.prototype.init = async function () {
    this.bindStaticEvents();
    this.applyInitialViewMode();

    if (!this.state.permissions.canView) {
      this.renderState('no-access');
      return;
    }

    if (!this.state.rootFolderId) {
      this.renderState('no-root');
      return;
    }

    await this.loadFolder(this.state.rootFolderId);
  };

  DiskComponent.prototype.applyInitialViewMode = function () {
    this.setViewMode(this.state.viewMode || 'table');
  };

  DiskComponent.prototype.setViewMode = function (mode) {
    this.state.viewMode = mode === 'grid' ? 'grid' : 'table';

    var tableContainer = this.root.querySelector('[data-view-container="table"]');
    var gridContainer = this.root.querySelector('[data-view-container="grid"]');
    var buttons = this.root.querySelectorAll('.sb-disk__view-btn');

    if (tableContainer) {
      tableContainer.hidden = this.state.viewMode !== 'table';
    }

    if (gridContainer) {
      gridContainer.hidden = this.state.viewMode !== 'grid';
    }

    buttons.forEach(function (btn) {
      btn.classList.toggle('is-active', btn.getAttribute('data-view') === mode);
    });
  };

  DiskComponent.prototype.api = async function (action, payload, isFormData) {
    if (isFormData) {
      var response = await fetch('/local/sitebuilder/components/disk/api.php?action=' + encodeURIComponent(action), {
        method: 'POST',
        body: payload
      });
      return await response.json();
    }

    var response = await fetch('/local/sitebuilder/components/disk/api.php?action=' + encodeURIComponent(action), {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    });

    return await response.json();
  };

  DiskComponent.prototype.getBasePayload = function () {
    return {
      siteId: this.state.siteId,
      pageId: this.state.pageId,
      blockId: this.state.blockId
    };
  };

  DiskComponent.prototype.getSortValue = function () {
    var select = this.root.querySelector('[data-role="sort-select"]');
    return select && select.value ? String(select.value) : 'updatedAt:desc';
  };

  DiskComponent.prototype.getSortBy = function () {
    return this.getSortValue().split(':')[0] || 'updatedAt';
  };

  DiskComponent.prototype.getSortDir = function () {
    return this.getSortValue().split(':')[1] || 'desc';
  };

  DiskComponent.prototype.loadFolder = async function (folderId) {
    try {
      this.setLoading(true);

      var payload = this.getBasePayload();
      payload.currentFolderId = folderId;
      payload.sortBy = this.getSortBy();
      payload.sortDir = this.getSortDir();
      payload.filters = {};

      var res = await this.api('list', payload);

      if (!res.ok) {
        throw new Error(res.message || res.error || 'LIST_ERROR');
      }

      this.state.currentFolderId = folderId;
      this.state.items = Array.isArray(res.data.items) ? res.data.items : [];
      this.state.breadcrumbs = Array.isArray(res.data.breadcrumbs) ? res.data.breadcrumbs : [];
      this.state.selectedIds = [];

      this.renderAll();

      if (res.meta && res.meta.noRoot) {
        this.renderState('no-root');
        return;
      }

      if (!this.state.items.length) {
        this.renderState('empty');
      } else {
        this.renderState(null);
      }
    } catch (e) {
      console.error(e);
      this.state.error = e.message || 'LIST_ERROR';
      this.renderState('error');
    } finally {
      this.setLoading(false);
    }
  };

  DiskComponent.prototype.search = async function (query) {
    try {
      var payload = this.getBasePayload();
      payload.query = query;

      var res = await this.api('search', payload);

      if (!res.ok) {
        throw new Error(res.message || res.error || 'SEARCH_ERROR');
      }

      this.state.items = Array.isArray(res.data.items) ? res.data.items : [];
      this.state.selectedIds = [];
      this.renderAll();

      if (!this.state.items.length) {
        this.renderState('empty');
      } else {
        this.renderState(null);
      }
    } catch (e) {
      console.error(e);
      this.renderState('error');
    }
  };

  DiskComponent.prototype.bindStaticEvents = function () {
    var self = this;

    var refreshBtn = this.root.querySelector('[data-action="refresh"]');
    if (refreshBtn) {
      refreshBtn.addEventListener('click', function () {
        self.loadFolder(self.state.currentFolderId || self.state.rootFolderId);
      });
    }

    var createFolderBtn = this.root.querySelector('[data-action="create-folder"]');
    if (createFolderBtn) {
      createFolderBtn.addEventListener('click', async function () {
        if (!self.state.permissions.canCreateFolder) return;

        var name = window.prompt('Название папки');
        if (!name) return;

        var payload = self.getBasePayload();
        payload.currentFolderId = self.state.currentFolderId;
        payload.name = name;

        var res = await self.api('createFolder', payload);
        if (!res.ok) {
          window.alert(res.message || res.error || 'Ошибка создания папки');
          return;
        }

        await self.loadFolder(self.state.currentFolderId);
      });
    }

    var uploadBtn = this.root.querySelector('[data-action="upload"]');
    var uploadInput = this.root.querySelector('[data-role="upload-input"]');

    if (uploadBtn && uploadInput) {
      uploadBtn.addEventListener('click', function () {
        if (!self.state.permissions.canUpload) return;
        uploadInput.click();
      });

      uploadInput.addEventListener('change', async function (e) {
        var files = Array.prototype.slice.call(e.target.files || []);
        if (!files.length) return;

        var formData = new FormData();
        formData.append('siteId', self.state.siteId);
        formData.append('pageId', self.state.pageId);
        formData.append('blockId', self.state.blockId);
        formData.append('currentFolderId', self.state.currentFolderId);

        files.forEach(function (file) {
          formData.append('files[]', file);
        });

        var res = await self.api('upload', formData, true);
        if (!res.ok) {
          window.alert(res.message || res.error || 'Ошибка загрузки');
          return;
        }

        uploadInput.value = '';
        await self.loadFolder(self.state.currentFolderId);
      });
    }

    var sortSelect = this.root.querySelector('[data-role="sort-select"]');
    if (sortSelect) {
      sortSelect.addEventListener('change', function () {
        self.loadFolder(self.state.currentFolderId || self.state.rootFolderId);
      });
    }

    var searchInput = this.root.querySelector('[data-role="search-input"]');
    if (searchInput) {
      var searchTimer = null;

      searchInput.addEventListener('input', function () {
        var value = String(searchInput.value || '').trim();

        clearTimeout(searchTimer);
        searchTimer = setTimeout(function () {
          self.state.searchQuery = value;

          if (value === '') {
            self.loadFolder(self.state.currentFolderId || self.state.rootFolderId);
            return;
          }

          self.search(value);
        }, 250);
      });
    }

    var selectAll = this.root.querySelector('[data-role="select-all"]');
    if (selectAll) {
      selectAll.addEventListener('change', function () {
        var checked = !!selectAll.checked;
        var checkboxes = self.root.querySelectorAll('.sb-disk__item-check');

        self.state.selectedIds = [];

        checkboxes.forEach(function (checkbox) {
          checkbox.checked = checked;
          var id = Number(checkbox.getAttribute('data-id') || 0);

          if (checked && id > 0) {
            self.state.selectedIds.push(id);
          }
        });

        self.syncSelectedState();
      });
    }

    var viewButtons = this.root.querySelectorAll('.sb-disk__view-btn');
    viewButtons.forEach(function (btn) {
      btn.addEventListener('click', function () {
        var mode = btn.getAttribute('data-view') || 'table';
        self.setViewMode(mode);
      });
    });

    this.root.addEventListener('click', async function (e) {
      var crumb = e.target.closest('.sb-disk__crumb');
      if (crumb) {
        var crumbFolderId = Number(crumb.getAttribute('data-folder-id') || 0);
        if (crumbFolderId > 0) {
          await self.loadFolder(crumbFolderId);
        }
        return;
      }

      var openBtn = e.target.closest('[data-row-action="open"]');
      if (openBtn) {
        var row = e.target.closest('[data-id][data-entity-type]');
        if (!row) return;

        var entityType = row.getAttribute('data-entity-type');
        var entityId = Number(row.getAttribute('data-id') || 0);

        if (entityType === 'folder' && entityId > 0) {
          await self.loadFolder(entityId);
        } else if (entityType === 'file') {
          var downloadUrl = row.getAttribute('data-download-url') || '';
          if (downloadUrl) {
            window.open(downloadUrl, '_blank');
          }
        }
        return;
      }

      var renameBtn = e.target.closest('[data-row-action="rename"]');
      if (renameBtn) {
        var renameRow = e.target.closest('[data-id][data-entity-type]');
        if (!renameRow) return;

        var renameEntityType = renameRow.getAttribute('data-entity-type');
        var renameEntityId = Number(renameRow.getAttribute('data-id') || 0);
        var currentName = renameRow.getAttribute('data-name') || '';

        var newName = window.prompt('Новое название', currentName);
        if (!newName) return;

        var renamePayload = self.getBasePayload();
        renamePayload.entityType = renameEntityType;
        renamePayload.entityId = renameEntityId;
        renamePayload.newName = newName;

        var renameRes = await self.api('rename', renamePayload);
        if (!renameRes.ok) {
          window.alert(renameRes.message || renameRes.error || 'Ошибка переименования');
          return;
        }

        await self.loadFolder(self.state.currentFolderId);
        return;
      }

      var deleteBtn = e.target.closest('[data-row-action="delete"]');
      if (deleteBtn) {
        var deleteRow = e.target.closest('[data-id][data-entity-type]');
        if (!deleteRow) return;

        var confirmDelete = window.confirm('Удалить элемент?');
        if (!confirmDelete) return;

        var deletePayload = self.getBasePayload();
        deletePayload.items = [{
          id: Number(deleteRow.getAttribute('data-id') || 0),
          entityType: deleteRow.getAttribute('data-entity-type')
        }];

        var deleteRes = await self.api('delete', deletePayload);
        if (!deleteRes.ok) {
          window.alert(deleteRes.message || deleteRes.error || 'Ошибка удаления');
          return;
        }

        await self.loadFolder(self.state.currentFolderId);
        return;
      }

      var deleteSelectedBtn = e.target.closest('[data-action="delete-selected"]');
      if (deleteSelectedBtn) {
        if (!self.state.selectedIds.length) return;

        var confirmBulkDelete = window.confirm('Удалить выбранные элементы?');
        if (!confirmBulkDelete) return;

        var selectedItems = self.collectSelectedItemsPayload();
        if (!selectedItems.length) return;

        var bulkDeletePayload = self.getBasePayload();
        bulkDeletePayload.items = selectedItems;

        var bulkDeleteRes = await self.api('delete', bulkDeletePayload);
        if (!bulkDeleteRes.ok) {
          window.alert(bulkDeleteRes.message || bulkDeleteRes.error || 'Ошибка удаления');
          return;
        }

        await self.loadFolder(self.state.currentFolderId);
        return;
      }

      var downloadSelectedBtn = e.target.closest('[data-action="download-selected"]');
      if (downloadSelectedBtn) {
        var rows = self.root.querySelectorAll('[data-id][data-entity-type="file"]');
        rows.forEach(function (row) {
          var id = Number(row.getAttribute('data-id') || 0);
          var downloadUrl = row.getAttribute('data-download-url') || '';

          if (self.state.selectedIds.indexOf(id) !== -1 && downloadUrl) {
            window.open(downloadUrl, '_blank');
          }
        });
      }
    });

    this.root.addEventListener('change', function (e) {
      var checkbox = e.target.closest('.sb-disk__item-check');
      if (!checkbox) return;

      var id = Number(checkbox.getAttribute('data-id') || 0);
      if (id <= 0) return;

      if (checkbox.checked) {
        if (self.state.selectedIds.indexOf(id) === -1) {
          self.state.selectedIds.push(id);
        }
      } else {
        self.state.selectedIds = self.state.selectedIds.filter(function (value) {
          return value !== id;
        });
      }

      self.syncSelectedState();
    });
  };

  DiskComponent.prototype.collectSelectedItemsPayload = function () {
    var rows = this.root.querySelectorAll('[data-id][data-entity-type]');
    var items = [];

    rows.forEach(function (row) {
      var id = Number(row.getAttribute('data-id') || 0);
      if (id <= 0) return;

      if (this.state.selectedIds.indexOf(id) !== -1) {
        items.push({
          id: id,
          entityType: row.getAttribute('data-entity-type')
        });
      }
    }, this);

    return items;
  };

  DiskComponent.prototype.syncSelectedState = function () {
    var rows = this.root.querySelectorAll('[data-id][data-entity-type]');
    rows.forEach(function (row) {
      var id = Number(row.getAttribute('data-id') || 0);
      var selected = this.state.selectedIds.indexOf(id) !== -1;
      row.classList.toggle('is-selected', selected);
    }, this);

    var cards = this.root.querySelectorAll('.sb-disk__card[data-id]');
    cards.forEach(function (card) {
      var id = Number(card.getAttribute('data-id') || 0);
      var selected = this.state.selectedIds.indexOf(id) !== -1;
      card.classList.toggle('is-selected', selected);
    }, this);

    var bulkbar = this.root.querySelector('[data-role="bulkbar"]');
    var bulkbarText = this.root.querySelector('[data-role="bulkbar-text"]');

    if (bulkbar && bulkbarText) {
      bulkbar.hidden = !this.state.selectedIds.length;
      bulkbarText.textContent = 'Выбрано: ' + this.state.selectedIds.length;
    }
  };

  DiskComponent.prototype.renderAll = function () {
    this.renderSubtitle();
    this.renderBreadcrumbs();
    this.renderItemsTable();
    this.renderItemsGrid();
    this.syncSelectedState();
  };

  DiskComponent.prototype.renderSubtitle = function () {
    var node = this.root.querySelector('[data-role="subtitle"]');
    if (!node) return;

    node.textContent = this.state.items.length + ' эл.';
  };

  DiskComponent.prototype.renderBreadcrumbs = function () {
    var container = this.root.querySelector('[data-role="breadcrumbs"]');
    if (!container) return;

    container.innerHTML = this.state.breadcrumbs.map(function (item) {
      return '<button type="button" class="sb-disk__crumb" data-folder-id="' + escapeHtml(item.id) + '">' +
        escapeHtml(item.name) +
      '</button>';
    }).join('<span>/</span>');
  };

  DiskComponent.prototype.renderItemsTable = function () {
    var tbody = this.root.querySelector('[data-role="items-table"]');
    if (!tbody) return;

    tbody.innerHTML = this.state.items.map(function (item) {
      var typeText = item.entityType === 'folder' ? 'Папка' : (item.extension || 'Файл');
      var sizeText = item.size ? formatBytes(item.size) : '';
      var badge = item.entityType === 'folder'
        ? '<span class="sb-disk__badge">Папка</span>'
        : '<span class="sb-disk__badge">' + escapeHtml(item.extension || 'Файл') + '</span>';

      return '' +
        '<tr class="sb-disk__row" ' +
          'data-id="' + escapeHtml(item.id) + '" ' +
          'data-entity-type="' + escapeHtml(item.entityType) + '" ' +
          'data-name="' + escapeHtml(item.name) + '" ' +
          'data-download-url="' + escapeHtml(item.downloadUrl || '') + '">' +
            '<td>' +
              '<input type="checkbox" class="sb-disk__item-check" data-id="' + escapeHtml(item.id) + '">' +
            '</td>' +
            '<td>' +
              '<div class="sb-disk__item-name">' +
                badge +
                '<span class="sb-disk__item-name-label">' + escapeHtml(item.name) + '</span>' +
              '</div>' +
            '</td>' +
            '<td>' + escapeHtml(typeText) + '</td>' +
            '<td>' + escapeHtml(sizeText) + '</td>' +
            '<td>' + escapeHtml(item.updatedAt || '') + '</td>' +
            '<td>' +
              '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button> ' +
              '<button type="button" class="sb-disk__row-btn" data-row-action="rename">Переим.</button> ' +
              '<button type="button" class="sb-disk__row-btn" data-row-action="delete">Удалить</button>' +
            '</td>' +
        '</tr>';
    }).join('');
  };

  DiskComponent.prototype.renderItemsGrid = function () {
    var container = this.root.querySelector('[data-view-container="grid"]');
    if (!container) return;

    container.innerHTML = this.state.items.map(function (item) {
      var typeText = item.entityType === 'folder' ? 'Папка' : (item.extension || 'Файл');
      var sizeText = item.size ? formatBytes(item.size) : '—';

      return '' +
        '<div class="sb-disk__card" ' +
             'data-id="' + escapeHtml(item.id) + '" ' +
             'data-entity-type="' + escapeHtml(item.entityType) + '" ' +
             'data-name="' + escapeHtml(item.name) + '" ' +
             'data-download-url="' + escapeHtml(item.downloadUrl || '') + '">' +
            '<div class="sb-disk__card-top">' +
              '<label>' +
                '<input type="checkbox" class="sb-disk__item-check" data-id="' + escapeHtml(item.id) + '">' +
              '</label>' +
              '<span class="sb-disk__badge">' + escapeHtml(typeText) + '</span>' +
            '</div>' +
            '<div class="sb-disk__card-name">' + escapeHtml(item.name) + '</div>' +
            '<div class="sb-disk__card-meta">' +
              '<span class="sb-disk__card-sub">Размер: ' + escapeHtml(sizeText) + '</span>' +
            '</div>' +
            '<div class="sb-disk__card-meta">' +
              '<span class="sb-disk__card-sub">' + escapeHtml(item.updatedAt || '') + '</span>' +
            '</div>' +
            '<div class="sb-disk__card-actions">' +
              '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>' +
              '<button type="button" class="sb-disk__row-btn" data-row-action="rename">Переим.</button>' +
              '<button type="button" class="sb-disk__row-btn" data-row-action="delete">Удалить</button>' +
            '</div>' +
        '</div>';
    }).join('');
  };

  DiskComponent.prototype.setLoading = function (loading) {
    this.state.loading = !!loading;
    this.renderState(loading ? 'loading' : null);
  };

  DiskComponent.prototype.renderState = function (stateName) {
    var nodes = this.root.querySelectorAll('[data-state]');
    nodes.forEach(function (node) {
      node.hidden = true;
    });

    if (!stateName) {
      return;
    }

    var node = this.root.querySelector('[data-state="' + stateName + '"]');
    if (node) {
      node.hidden = false;
    }
  };

  function formatBytes(bytes) {
    bytes = Number(bytes || 0);
    if (bytes <= 0) return '0 Б';

    var units = ['Б', 'КБ', 'МБ', 'ГБ', 'ТБ'];
    var unitIndex = 0;

    while (bytes >= 1024 && unitIndex < units.length - 1) {
      bytes /= 1024;
      unitIndex++;
    }

    var value = unitIndex === 0 ? Math.round(bytes) : bytes.toFixed(1);
    return String(value) + ' ' + units[unitIndex];
  }

  function escapeHtml(value) {
    return String(value == null ? '' : value)
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#039;');
  }

  document.addEventListener('DOMContentLoaded', function () {
    document.querySelectorAll('.sb-disk').forEach(function (root) {
      var component = new DiskComponent(root);
      component.init();
    });
  });
})();


---

29. Как это подключать в страницу

Пример:

<?php

$component = new SitebuilderDiskComponent([
    'SITE_ID' => (int)$siteId,
    'PAGE_ID' => (int)$pageId,
    'BLOCK_ID' => (int)$block['id'],
    'CURRENT_USER_ID' => (int)$USER->GetID(),
]);

$component->execute();

И не забыть подключить стили и скрипт:

<link rel="stylesheet" href="/local/sitebuilder/components/disk/styles.css">
<script src="/local/sitebuilder/components/disk/script.js"></script>


---

30. Что в этом комплекте уже есть

Уже есть:

каркас компонента;

контекст siteId/pageId/blockId/currentUserId;

стартовый шаблон;

UI таблица и плитка;

breadcrumbs;

поиск;

массовое выделение;

загрузка файлов;

создание папки;

переименование;

удаление;

API-каркас;

заготовка прав;

заготовка root resolver;

адаптер хранилища.



---

31. Что нужно заменить у вас в проекте в первую очередь

Самые важные места, которые надо будет сразу связать с вашим проектом:

1. Пользователь

Сейчас в action-файлах стоит:

'currentUserId' => 1

Нужно заменить на реальный ID текущего пользователя.


---

2. Проверка принадлежности блока

В DiskValidator::assertBlockBelongsToContext() нужно сделать реальный SELECT по вашему sitebuilder.


---

3. Получение настроек

В DiskSettingsRepository вместо заглушек должен быть реальный SELECT / INSERT / UPDATE.


---

4. Получение корня сайта

В DiskRootResolver::getSiteRootFolderId() нужно подставить вашу таблицу сайта или диск-сущность.


---

5. Проверка дерева папок

assertFolderInsideRoot() сейчас заглушка. Это одно из самых важных мест.


---

6. Хранилище

В DiskStorageAdapter надо подставить либо:

Bitrix Disk,

либо ваше файловое хранилище,

либо метаданные из БД + физические файлы.



---

32. Что логично сделать следующим шагом

Следующий хороший шаг — я могу собрать вам второй пакет, уже более близкий к продакшену:

полноценный DiskSettingsRepository с SQL-логикой;

BlockRepository / SiteRepository под проверку контекста;

DiskRootResolver с реальным резолвом;

Bitrix Disk adapter или адаптер под вашу структуру.


Самый полезный следующий шаг здесь — сделать именно серверную часть без заглушек под ваш sitebuilder.