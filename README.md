Ниже — следующий пакет, уже ближе к реальному запуску:

реальный currentUserId из Bitrix;

проверка sessid;

инициализация root папки сайта;

инициализация root папки блока;

внутренние права сайта/блока;

готовые helper-файлы и обновленные action-файлы.



---

1. Что добавляем в структуру

/local/sitebuilder/components/disk/
├── bootstrap.php
├── lib/
│   ├── DiskCurrentUser.php
│   ├── DiskCsrf.php
│   ├── SiteAccessRepository.php
│   ├── SiteDiskInitializer.php
│   ├── BlockDiskInitializer.php
│   ├── DiskPermissionService.php
│   └── ...


---

2. Helper текущего пользователя

lib/DiskCurrentUser.php

<?php

class DiskCurrentUser
{
    public static function getId(): int
    {
        global $USER;

        if ($USER instanceof CUser) {
            return (int)$USER->GetID();
        }

        return 0;
    }

    public static function requireId(): int
    {
        $userId = self::getId();
        if ($userId <= 0) {
            throw new RuntimeException('NOT_AUTHORIZED');
        }

        return $userId;
    }

    public static function isAdmin(): bool
    {
        global $USER;

        if (!($USER instanceof CUser)) {
            return false;
        }

        return $USER->IsAdmin();
    }

    public static function getGroupIds(): array
    {
        global $USER;

        if (!($USER instanceof CUser)) {
            return [];
        }

        $groups = $USER->GetUserGroupArray();
        if (!is_array($groups)) {
            return [];
        }

        return array_values(array_map('intval', $groups));
    }
}


---

3. Проверка sessid

lib/DiskCsrf.php

<?php

class DiskCsrf
{
    public static function validateFromRequest(): void
    {
        $sessid = '';

        if (isset($_POST['sessid'])) {
            $sessid = (string)$_POST['sessid'];
        } elseif (isset($_REQUEST['sessid'])) {
            $sessid = (string)$_REQUEST['sessid'];
        } else {
            $json = disk_read_json_body();
            if (isset($json['sessid'])) {
                $sessid = (string)$json['sessid'];
            }
        }

        if ($sessid === '') {
            throw new RuntimeException('EMPTY_SESSID');
        }

        if (!check_bitrix_sessid($sessid)) {
            throw new RuntimeException('BAD_SESSID');
        }
    }
}


---

4. Обновленный bootstrap.php

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

    require_once $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

    \Bitrix\Main\Loader::includeModule('disk');

    require_once __DIR__ . '/lib/helpers.php';
    require_once __DIR__ . '/lib/DiskDb.php';
    require_once __DIR__ . '/lib/DiskContext.php';
    require_once __DIR__ . '/lib/DiskResponse.php';
    require_once __DIR__ . '/lib/DiskCurrentUser.php';
    require_once __DIR__ . '/lib/DiskCsrf.php';
    require_once __DIR__ . '/lib/BlockRepository.php';
    require_once __DIR__ . '/lib/SiteRepository.php';
    require_once __DIR__ . '/lib/SiteAccessRepository.php';
    require_once __DIR__ . '/lib/DiskSettingsRepository.php';
    require_once __DIR__ . '/lib/DiskRootResolver.php';
    require_once __DIR__ . '/lib/DiskValidator.php';
    require_once __DIR__ . '/lib/DiskPermissionService.php';
    require_once __DIR__ . '/lib/SiteDiskInitializer.php';
    require_once __DIR__ . '/lib/BlockDiskInitializer.php';
    require_once __DIR__ . '/lib/DiskStorageAdapterInterface.php';
    require_once __DIR__ . '/lib/DiskBitrixStorageAdapter.php';
}


---

5. Таблица ролей сайта

Ниже простая и практичная модель.

SQL

CREATE TABLE sitebuilder_site_user_access (
    id                 BIGSERIAL PRIMARY KEY,
    site_id            BIGINT NOT NULL,
    user_id            BIGINT NOT NULL,
    role_code          VARCHAR(50) NOT NULL,
    created_at         TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at         TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE UNIQUE INDEX uq_sb_site_user_access
    ON sitebuilder_site_user_access(site_id, user_id);

CREATE INDEX idx_sb_site_user_access_site_id
    ON sitebuilder_site_user_access(site_id);

CREATE INDEX idx_sb_site_user_access_user_id
    ON sitebuilder_site_user_access(user_id);

Роли:

site_admin

site_editor

site_user

site_viewer



---

6. Репозиторий доступа к сайту

lib/SiteAccessRepository.php

<?php

class SiteAccessRepository
{
    public static function getUserRole(int $siteId, int $userId): ?string
    {
        if ($siteId <= 0 || $userId <= 0) {
            return null;
        }

        $sql = "
            SELECT role_code
            FROM sitebuilder_site_user_access
            WHERE site_id = :site_id
              AND user_id = :user_id
            LIMIT 1
        ";

        $row = DiskDb::fetchOne($sql, [
            ':site_id' => $siteId,
            ':user_id' => $userId,
        ]);

        return $row ? (string)$row['role_code'] : null;
    }

    public static function setUserRole(int $siteId, int $userId, string $roleCode): bool
    {
        $roleCode = trim($roleCode);

        if (!in_array($roleCode, ['site_admin', 'site_editor', 'site_user', 'site_viewer'], true)) {
            throw new RuntimeException('INVALID_ROLE_CODE');
        }

        $existing = self::getUserRole($siteId, $userId);

        if ($existing === null) {
            $sql = "
                INSERT INTO sitebuilder_site_user_access (
                    site_id,
                    user_id,
                    role_code,
                    created_at,
                    updated_at
                ) VALUES (
                    :site_id,
                    :user_id,
                    :role_code,
                    CURRENT_TIMESTAMP,
                    CURRENT_TIMESTAMP
                )
            ";

            return DiskDb::execute($sql, [
                ':site_id' => $siteId,
                ':user_id' => $userId,
                ':role_code' => $roleCode,
            ]);
        }

        $sql = "
            UPDATE sitebuilder_site_user_access
            SET role_code = :role_code,
                updated_at = CURRENT_TIMESTAMP
            WHERE site_id = :site_id
              AND user_id = :user_id
        ";

        return DiskDb::execute($sql, [
            ':site_id' => $siteId,
            ':user_id' => $userId,
            ':role_code' => $roleCode,
        ]);
    }

    public static function hasAnyAccess(int $siteId, int $userId): bool
    {
        return self::getUserRole($siteId, $userId) !== null;
    }
}


---

7. Реальный DiskPermissionService.php

Теперь он учитывает:

админа Bitrix;

роль пользователя на сайте;

ограничения блока.


lib/DiskPermissionService.php

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

8. Инициализация корня сайта

Ниже улучшенная версия. Она создаст корневую папку сайта в Bitrix Disk, если ее еще нет.

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

9. Инициализация отдельного корня блока

Если конкретному блоку нужна собственная подпапка — делаем helper.

lib/BlockDiskInitializer.php

<?php

use Bitrix\Disk\Folder;

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

        $adapter = new DiskBitrixStorageAdapter($currentUserId);

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
        ], \Bitrix\Disk\Driver::getInstance()->getFakeSecurityContext($currentUserId));

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

10. Экшен инициализации root сайта

Полезный action для админа/редактора.

actions/init_site_root.php

<?php

DiskCsrf::validateFromRequest();
$data = disk_read_json_body();

$currentUserId = DiskCurrentUser::requireId();

$siteId = (int)($data['siteId'] ?? 0);
if ($siteId <= 0) {
    throw new RuntimeException('INVALID_SITE_ID');
}

$site = SiteRepository::getById($siteId);
if (!$site) {
    throw new RuntimeException('SITE_NOT_FOUND');
}

$role = SiteAccessRepository::getUserRole($siteId, $currentUserId);
if (!DiskCurrentUser::isAdmin() && !in_array($role, ['site_admin', 'site_editor'], true)) {
    throw new RuntimeException('ACCESS_DENIED');
}

$folderId = SiteDiskInitializer::ensureSiteRootFolder(
    $siteId,
    $currentUserId,
    (string)$site['name']
);

DiskResponse::success([
    'rootFolderId' => $folderId,
]);


---

11. Экшен инициализации root блока

actions/init_block_root.php

<?php

DiskCsrf::validateFromRequest();
$data = disk_read_json_body();

$currentUserId = DiskCurrentUser::requireId();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => $currentUserId,
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::ensureExistsForBlock(
    $context->blockId,
    $context->siteId,
    $context->pageId,
    $context->currentUserId
);

$permissions = DiskPermissionService::resolve($context, $settings, null);
DiskValidator::assertCan($permissions, 'canEditSettings');

$folderId = BlockDiskInitializer::ensureBlockRootFolder(
    $context->siteId,
    $context->pageId,
    $context->blockId,
    $context->currentUserId,
    (string)($settings['title'] ?? '')
);

DiskResponse::success([
    'rootFolderId' => $folderId,
]);


---

12. Обновленный api.php

Добавьте новые action:

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

        case 'initSiteRoot':
            require __DIR__ . '/actions/init_site_root.php';
            break;

        case 'initBlockRoot':
            require __DIR__ . '/actions/init_block_root.php';
            break;

        default:
            DiskResponse::error('UNKNOWN_ACTION', 'Неизвестное действие');
    }
} catch (Throwable $e) {
    DiskResponse::error('SERVER_ERROR', $e->getMessage());
}


---

13. Как заменить currentUserId => 1 в action-файлах

Теперь во всех action-файлах делайте одинаково:

DiskCsrf::validateFromRequest();

$currentUserId = DiskCurrentUser::requireId();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => $currentUserId,
]);


---

14. Пример обновленного actions/list.php

<?php

DiskCsrf::validateFromRequest();
$data = disk_read_json_body();

$currentUserId = DiskCurrentUser::requireId();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => $currentUserId,
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

15. Пример кнопок на фронте для инициализации root

Если root не настроен, можно показать 2 варианта:

создать корень сайта;

создать корень блока.


Пример JS-вызова:

async function initSiteRoot(siteId) {
  const res = await fetch('/local/sitebuilder/components/disk/api.php?action=initSiteRoot', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      siteId,
      sessid: BX.bitrix_sessid()
    })
  });

  return await res.json();
}

async function initBlockRoot(siteId, pageId, blockId) {
  const res = await fetch('/local/sitebuilder/components/disk/api.php?action=initBlockRoot', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      siteId,
      pageId,
      blockId,
      sessid: BX.bitrix_sessid()
    })
  });

  return await res.json();
}


---

16. Что теперь уже закрыто

После этого пакета у вас уже есть:

реальный текущий пользователь Bitrix;

проверка sessid;

реальная модель ролей сайта;

внутренние права блока;

создание root папки сайта;

создание root папки конкретного блока;

связка Disk + sitebuilder + block context.



---

17. Что еще осталось самым важным

Сейчас осталось два особенно полезных шага:

1. Настройки блока в UI

Нужно модальное окно/панель:

заголовок;

собственная root папка;

режим table/grid;

allowUpload;

allowCreateFolder;

allowDelete;

allowRename;

allowDownload;

showSearch;

showBreadcrumbs;

allowedExtensions;

maxFileSize.


2. Реальная интеграция в ваш editor/sitebuilder

Нужно:

зарегистрировать блок disk;

добавить его в список блоков;

сделать создание блока через API;

отрисовывать компонент на странице.


Следующим сообщением логичнее всего сделать готовый UI настроек блока + action сохранения + JS для открытия модалки.