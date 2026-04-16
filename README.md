Отлично. Ниже даю уже практически готовый каркас реализации компонента «Диск» для вашего проекта в стиле sitebuilder, чтобы это можно было брать как основу разработки.


---

1. Рекомендуемое место в проекте

Для вашего проекта логично положить компонент сюда:

/local/sitebuilder/components/disk/

И отдельно зарегистрировать обработчик API в общем API проекта, либо временно использовать локальный API компонента.

Рекомендуемая структура:

/local/sitebuilder/components/disk/
├── class.php
├── template.php
├── styles.css
├── script.js
├── api.php
├── bootstrap.php
├── context.php
├── settings.php
├── permissions.php
├── renderer.php
├── storage.php
├── lib/
│   ├── DiskContext.php
│   ├── DiskSettingsRepository.php
│   ├── DiskRootResolver.php
│   ├── DiskPermissionService.php
│   ├── DiskStorageAdapterInterface.php
│   ├── DiskStorageAdapter.php
│   ├── DiskFolderTreeService.php
│   ├── DiskValidator.php
│   ├── DiskResponse.php
│   └── helpers.php
├── actions/
│   ├── resolve_root.php
│   ├── get_settings.php
│   ├── save_settings.php
│   ├── get_permissions.php
│   ├── list.php
│   ├── upload.php
│   ├── create_folder.php
│   ├── rename.php
│   ├── delete.php
│   ├── move.php
│   ├── copy.php
│   ├── search.php
│   └── download.php
└── templates/
    ├── table_row.php
    ├── grid_card.php
    ├── state_empty.php
    ├── state_error.php
    ├── state_no_access.php
    ├── state_no_root.php
    └── modals/
        ├── create_folder.php
        ├── rename.php
        ├── delete_confirm.php
        ├── move.php
        └── settings.php


---

2. SQL-структура таблиц

Ниже дам SQL в нейтральном виде. Под ваш проект потом можно адаптировать под конкретную СУБД.


---

2.1. Таблица блоков конструктора

Если уже есть таблица блоков — не дублировать. Тогда просто использовать существующую. Ниже для полноты:

CREATE TABLE sitebuilder_block (
    id                BIGSERIAL PRIMARY KEY,
    site_id           BIGINT NOT NULL,
    page_id           BIGINT NOT NULL,
    type              VARCHAR(50) NOT NULL,
    sort              INTEGER NOT NULL DEFAULT 500,
    settings_json     TEXT NULL,
    is_active         SMALLINT NOT NULL DEFAULT 1,
    created_by        BIGINT NULL,
    created_at        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_sb_block_site_id ON sitebuilder_block(site_id);
CREATE INDEX idx_sb_block_page_id ON sitebuilder_block(page_id);
CREATE INDEX idx_sb_block_type ON sitebuilder_block(type);


---

2.2. Таблица настроек компонента Disk

CREATE TABLE sitebuilder_disk_settings (
    id                         BIGSERIAL PRIMARY KEY,
    block_id                   BIGINT NOT NULL UNIQUE,
    site_id                    BIGINT NOT NULL,
    page_id                    BIGINT NOT NULL,
    title                      VARCHAR(255) NOT NULL DEFAULT 'Файлы',
    root_folder_id             BIGINT NULL,
    view_mode                  VARCHAR(20) NOT NULL DEFAULT 'table',
    allow_upload               SMALLINT NOT NULL DEFAULT 1,
    allow_create_folder        SMALLINT NOT NULL DEFAULT 1,
    allow_rename               SMALLINT NOT NULL DEFAULT 1,
    allow_delete               SMALLINT NOT NULL DEFAULT 0,
    allow_download             SMALLINT NOT NULL DEFAULT 1,
    show_search                SMALLINT NOT NULL DEFAULT 1,
    show_breadcrumbs           SMALLINT NOT NULL DEFAULT 1,
    default_sort               VARCHAR(50) NOT NULL DEFAULT 'updatedAt',
    default_sort_direction     VARCHAR(10) NOT NULL DEFAULT 'desc',
    allowed_extensions_json    TEXT NULL,
    max_file_size              BIGINT NOT NULL DEFAULT 52428800,
    permission_mode            VARCHAR(30) NOT NULL DEFAULT 'inherit_site',
    use_site_root_fallback     SMALLINT NOT NULL DEFAULT 1,
    created_by                 BIGINT NULL,
    created_at                 TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at                 TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_sb_disk_settings_site_id ON sitebuilder_disk_settings(site_id);
CREATE INDEX idx_sb_disk_settings_page_id ON sitebuilder_disk_settings(page_id);


---

2.3. Корневой диск сайта

Если у сайта уже есть поле rootDiskFolderId, используйте его. Если нет — отдельная таблица:

CREATE TABLE sitebuilder_site_disk (
    site_id              BIGINT PRIMARY KEY,
    root_folder_id       BIGINT NULL,
    storage_type         VARCHAR(30) NOT NULL DEFAULT 'bitrix_disk',
    created_at           TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at           TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);


---

2.4. Таблица логических папок

Если будете поверх внешнего хранилища вести свои метаданные:

CREATE TABLE sitebuilder_disk_folder (
    id                  BIGSERIAL PRIMARY KEY,
    external_id         BIGINT NULL,
    parent_id           BIGINT NULL,
    site_id             BIGINT NOT NULL,
    block_id            BIGINT NULL,
    name                VARCHAR(255) NOT NULL,
    path                TEXT NULL,
    depth               INTEGER NOT NULL DEFAULT 0,
    is_deleted          SMALLINT NOT NULL DEFAULT 0,
    created_by          BIGINT NULL,
    created_at          TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_sb_disk_folder_parent_id ON sitebuilder_disk_folder(parent_id);
CREATE INDEX idx_sb_disk_folder_site_id ON sitebuilder_disk_folder(site_id);
CREATE INDEX idx_sb_disk_folder_block_id ON sitebuilder_disk_folder(block_id);


---

2.5. Таблица логических файлов

CREATE TABLE sitebuilder_disk_file (
    id                  BIGSERIAL PRIMARY KEY,
    external_id         BIGINT NULL,
    folder_id           BIGINT NOT NULL,
    site_id             BIGINT NOT NULL,
    block_id            BIGINT NULL,
    name                VARCHAR(255) NOT NULL,
    original_name       VARCHAR(255) NULL,
    extension           VARCHAR(20) NULL,
    mime_type           VARCHAR(255) NULL,
    size                BIGINT NOT NULL DEFAULT 0,
    path                TEXT NULL,
    hash                VARCHAR(64) NULL,
    is_deleted          SMALLINT NOT NULL DEFAULT 0,
    created_by          BIGINT NULL,
    created_at          TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    download_url        TEXT NULL,
    preview_url         TEXT NULL
);

CREATE INDEX idx_sb_disk_file_folder_id ON sitebuilder_disk_file(folder_id);
CREATE INDEX idx_sb_disk_file_site_id ON sitebuilder_disk_file(site_id);
CREATE INDEX idx_sb_disk_file_block_id ON sitebuilder_disk_file(block_id);


---

2.6. Таблица прав доступа

Если хотите гибкость по сайту / блоку / папке:

CREATE TABLE sitebuilder_disk_permission (
    id                    BIGSERIAL PRIMARY KEY,
    site_id               BIGINT NOT NULL,
    block_id              BIGINT NULL,
    folder_id             BIGINT NULL,
    subject_type          VARCHAR(20) NOT NULL, -- user / group / role
    subject_id            VARCHAR(100) NOT NULL,
    can_view              SMALLINT NOT NULL DEFAULT 1,
    can_upload            SMALLINT NOT NULL DEFAULT 0,
    can_create_folder     SMALLINT NOT NULL DEFAULT 0,
    can_rename            SMALLINT NOT NULL DEFAULT 0,
    can_delete            SMALLINT NOT NULL DEFAULT 0,
    can_download          SMALLINT NOT NULL DEFAULT 1,
    can_manage_access     SMALLINT NOT NULL DEFAULT 0,
    can_edit_settings     SMALLINT NOT NULL DEFAULT 0,
    created_at            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_sb_disk_perm_site_id ON sitebuilder_disk_permission(site_id);
CREATE INDEX idx_sb_disk_perm_block_id ON sitebuilder_disk_permission(block_id);
CREATE INDEX idx_sb_disk_perm_folder_id ON sitebuilder_disk_permission(folder_id);


---

3. Как именно привязывается блок к siteId / pageId / blockId

Ниже схема поведения.


---

3.1. При добавлении блока в редакторе

Когда редактор добавляет блок типа disk, выполняется:

1. создается запись в sitebuilder_block;


2. создается запись в sitebuilder_disk_settings;


3. в sitebuilder_disk_settings.block_id сохраняется ID блока;


4. в sitebuilder_disk_settings.site_id/page_id дублируется контекст;


5. если root не задан, блок будет использовать root сайта.




---

3.2. Пример создания блока

$blockId = BlockRepository::create([
    'site_id' => $siteId,
    'page_id' => $pageId,
    'type' => 'disk',
    'sort' => 500,
    'settings_json' => '{}',
    'created_by' => $currentUserId,
]);

DiskSettingsRepository::createDefault([
    'block_id' => $blockId,
    'site_id' => $siteId,
    'page_id' => $pageId,
    'created_by' => $currentUserId,
]);


---

4. Каркас PHP: главный класс компонента

class.php

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
            'CAN_VIEW' => !empty($permissions['canView']),
            'INITIAL_STATE' => [
                'siteId' => $context->siteId,
                'pageId' => $context->pageId,
                'blockId' => $context->blockId,
                'rootFolderId' => $rootFolderId,
                'currentFolderId' => $rootFolderId,
                'settings' => $settings,
                'permissions' => $permissions,
            ],
        ];

        include __DIR__ . '/template.php';
    }
}


---

5. Bootstrap компонента

bootstrap.php

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
    require_once __DIR__ . '/lib/DiskFolderTreeService.php';
}


---

6. Контекст компонента

lib/DiskContext.php

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

7. Репозиторий настроек компонента

lib/DiskSettingsRepository.php

<?php

class DiskSettingsRepository
{
    public static function getByBlockId(int $blockId): array
    {
        // Здесь должен быть реальный запрос к БД.
        // Временный пример.
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
        return true;
    }

    public static function save(int $blockId, array $settings): bool
    {
        return true;
    }
}


---

8. Резолвер корневой папки

lib/DiskRootResolver.php

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
            if ($siteRootFolderId) {
                return (int)$siteRootFolderId;
            }
        }

        return null;
    }

    protected static function getSiteRootFolderId(int $siteId): ?int
    {
        // Реальный запрос к site / sitebuilder_site_disk
        return null;
    }
}


---

9. Расчет прав

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
        // Здесь должна быть реальная логика определения роли пользователя в рамках сайта.
        // Пока временно считаем, что пользователь - редактор.

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

10. Валидатор контекста

lib/DiskValidator.php

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
        // Здесь обязательна реальная проверка:
        // block.site_id == context.siteId
        // block.page_id == context.pageId
        // block.type == disk

        $ok = true;

        if (!$ok) {
            throw new RuntimeException('BLOCK_CONTEXT_MISMATCH');
        }
    }

    public static function assertFolderInsideRoot(int $folderId, int $rootFolderId): void
    {
        // Здесь должна быть реальная проверка принадлежности папки дереву rootFolderId.
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
}


---

11. Адаптер хранилища

Если будете работать через Bitrix Disk или внутреннее хранилище — у вас должен быть единый интерфейс.

lib/DiskStorageAdapterInterface.php

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

lib/DiskStorageAdapter.php

<?php

class DiskStorageAdapter implements DiskStorageAdapterInterface
{
    public function listItems(DiskContext $context, int $folderId, array $options = []): array
    {
        return [
            [
                'id' => 101,
                'entityType' => 'folder',
                'name' => 'Документы',
                'extension' => null,
                'mimeType' => null,
                'size' => null,
                'updatedAt' => '2026-04-16 10:00:00',
                'downloadUrl' => null,
            ],
            [
                'id' => 102,
                'entityType' => 'file',
                'name' => 'report.pdf',
                'extension' => 'pdf',
                'mimeType' => 'application/pdf',
                'size' => 124000,
                'updatedAt' => '2026-04-16 09:55:00',
                'downloadUrl' => '/local/sitebuilder/components/disk/api.php?action=download&fileId=102',
            ],
        ];
    }

    public function createFolder(DiskContext $context, int $parentFolderId, string $name): array
    {
        return [
            'id' => 5001,
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
        return [];
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

12. Ответы API

lib/DiskResponse.php

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
        ]);
    }

    public static function error(string $errorCode, string $message = '', array $details = []): void
    {
        self::send([
            'ok' => false,
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

13. Единая точка входа API

api.php

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

14. Общий помощник чтения JSON-body

lib/helpers.php

<?php

function disk_read_json_body(): array
{
    $raw = file_get_contents('php://input');
    if (!$raw) {
        return [];
    }

    $data = json_decode($raw, true);
    return is_array($data) ? $data : [];
}


---

15. Пример API-метода resolveRoot

actions/resolve_root.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1, // заменить на реального пользователя
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::getByBlockId($context->blockId);
$rootFolderId = DiskRootResolver::resolve($context, $settings);

DiskResponse::success([
    'rootFolderId' => $rootFolderId,
    'source' => !empty($settings['rootFolderId']) ? 'block' : 'site',
]);


---

16. Пример API-метода getSettings

actions/get_settings.php

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

17. Пример API-метода getPermissions

actions/get_permissions.php

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

18. Пример API-метода list

actions/list.php

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

$currentFolderId = (int)($data['currentFolderId'] ?? $rootFolderId);

if ($rootFolderId && $currentFolderId) {
    DiskValidator::assertFolderInsideRoot($currentFolderId, $rootFolderId);
}

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
    'total' => count($items),
    'isEmpty' => empty($items),
]);


---

19. Пример API-метода createFolder

actions/create_folder.php

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

if ($currentFolderId <= 0) {
    throw new RuntimeException('INVALID_FOLDER_ID');
}

if ($name === '') {
    throw new RuntimeException('EMPTY_FOLDER_NAME');
}

DiskValidator::assertFolderInsideRoot($currentFolderId, $rootFolderId);

$adapter = new DiskStorageAdapter();
$folder = $adapter->createFolder($context, $currentFolderId, $name);

DiskResponse::success([
    'folder' => $folder,
]);


---

20. Пример API-метода upload

Здесь отдельно важно: upload обычно идет через multipart/form-data, а не JSON.

actions/upload.php

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
for ($i = 0; $i < count($files['name']); $i++) {
    $normalizedFiles[] = [
        'name' => $files['name'][$i],
        'type' => $files['type'][$i],
        'tmp_name' => $files['tmp_name'][$i],
        'error' => $files['error'][$i],
        'size' => $files['size'][$i],
    ];
}

foreach ($normalizedFiles as $file) {
    if ($file['error'] !== UPLOAD_ERR_OK) {
        throw new RuntimeException('UPLOAD_ERROR');
    }

    if (!empty($settings['maxFileSize']) && $file['size'] > (int)$settings['maxFileSize']) {
        throw new RuntimeException('FILE_TOO_LARGE');
    }

    if (!empty($settings['allowedExtensions'])) {
        $ext = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));
        if (!in_array($ext, $settings['allowedExtensions'], true)) {
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

21. HTML-шаблон компонента

template.php

<?php
$initialStateJson = htmlspecialchars(json_encode($arResult['INITIAL_STATE'], JSON_UNESCAPED_UNICODE), ENT_QUOTES, 'UTF-8');
?>
<div class="sb-disk"
     id="sb-disk-<?= (int)$arResult['BLOCK_ID'] ?>"
     data-site-id="<?= (int)$arResult['SITE_ID'] ?>"
     data-page-id="<?= (int)$arResult['PAGE_ID'] ?>"
     data-block-id="<?= (int)$arResult['BLOCK_ID'] ?>"
     data-initial-state="<?= $initialStateJson ?>">

    <div class="sb-disk__header">
        <div class="sb-disk__header-left">
            <h3 class="sb-disk__title"><?= htmlspecialchars($arResult['TITLE']) ?></h3>
            <div class="sb-disk__subtitle" data-role="subtitle"></div>
        </div>

        <div class="sb-disk__header-right">
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
                    <input type="text" class="sb-disk__search-input" data-role="search-input" placeholder="Поиск файлов и папок">
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
        <button type="button" class="sb-disk__btn" data-action="download-selected">Скачать</button>
        <button type="button" class="sb-disk__btn sb-disk__btn--danger" data-action="delete-selected">Удалить</button>
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
                        <th class="sb-disk__col sb-disk__col--checkbox"></th>
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

    <input type="file" multiple hidden data-role="upload-input">
</div>


---

22. CSS naming и стартовый CSS-каркас

styles.css

.sb-disk {
    border: 1px solid #dfe3ea;
    border-radius: 16px;
    background: #ffffff;
    padding: 20px;
    display: flex;
    flex-direction: column;
    gap: 16px;
}

.sb-disk__header,
.sb-disk__toolbar,
.sb-disk__bulkbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
}

.sb-disk__header-left,
.sb-disk__toolbar-left,
.sb-disk__toolbar-right {
    display: flex;
    align-items: center;
    gap: 12px;
}

.sb-disk__title {
    margin: 0;
    font-size: 20px;
    line-height: 1.2;
    font-weight: 600;
}

.sb-disk__subtitle {
    font-size: 13px;
    color: #6b7280;
}

.sb-disk__breadcrumbs {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    font-size: 13px;
}

.sb-disk__crumb {
    background: none;
    border: none;
    padding: 0;
    cursor: pointer;
    color: #2563eb;
}

.sb-disk__search-input,
.sb-disk__select {
    height: 40px;
    border: 1px solid #d1d5db;
    border-radius: 10px;
    padding: 0 12px;
    font-size: 14px;
}

.sb-disk__search-input {
    min-width: 260px;
}

.sb-disk__btn,
.sb-disk__view-btn {
    height: 40px;
    border: 1px solid #d1d5db;
    border-radius: 10px;
    background: #fff;
    padding: 0 14px;
    font-size: 14px;
    cursor: pointer;
}

.sb-disk__btn--danger {
    border-color: #ef4444;
}

.sb-disk__btn--ghost {
    background: #f8fafc;
}

.sb-disk__view-switch {
    display: flex;
    gap: 8px;
}

.sb-disk__view-btn.is-active {
    font-weight: 600;
}

.sb-disk__content {
    min-height: 240px;
}

.sb-disk__table {
    width: 100%;
    border-collapse: collapse;
}

.sb-disk__table th,
.sb-disk__table td {
    padding: 12px 10px;
    border-bottom: 1px solid #eef2f7;
    text-align: left;
    vertical-align: middle;
}

.sb-disk__row {
    cursor: default;
}

.sb-disk__row.is-selected {
    background: #f5f9ff;
}

.sb-disk__state {
    padding: 40px 20px;
    text-align: center;
    color: #6b7280;
    border: 1px dashed #d1d5db;
    border-radius: 12px;
}

.sb-disk__grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
    gap: 16px;
}

.sb-disk__card {
    border: 1px solid #e5e7eb;
    border-radius: 14px;
    padding: 14px;
    background: #fff;
}

.is-hidden {
    display: none !important;
}

.is-disabled {
    opacity: .5;
    pointer-events: none;
}


---

23. JS-компонент

script.js

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
      viewMode: (parsed.settings && parsed.settings.viewMode) || 'table',
      loading: false,
      error: null
    };
  };

  DiskComponent.prototype.init = async function () {
    this.bindStaticEvents();

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

  DiskComponent.prototype.api = async function (action, payload, isFormData) {
    if (isFormData) {
      var response = await fetch('/local/sitebuilder/components/disk/api.php?action=' + action, {
        method: 'POST',
        body: payload
      });
      return await response.json();
    }

    var response = await fetch('/local/sitebuilder/components/disk/api.php?action=' + action, {
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

  DiskComponent.prototype.loadFolder = async function (folderId) {
    try {
      this.setLoading(true);

      var payload = this.getBasePayload();
      payload.currentFolderId = folderId;
      payload.sortBy = this.getSortBy();
      payload.sortDir = this.getSortDir();

      var res = await this.api('list', payload);

      if (!res.ok) {
        throw new Error(res.message || res.error || 'LIST_ERROR');
      }

      this.state.currentFolderId = folderId;
      this.state.items = res.data.items || [];
      this.state.breadcrumbs = res.data.breadcrumbs || [];
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
    } finally {
      this.setLoading(false);
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

        var name = prompt('Название папки');
        if (!name) return;

        var payload = self.getBasePayload();
        payload.currentFolderId = self.state.currentFolderId;
        payload.name = name;

        var res = await self.api('createFolder', payload);
        if (!res.ok) {
          alert(res.message || res.error || 'Ошибка создания папки');
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
          alert(res.message || res.error || 'Ошибка загрузки');
          return;
        }

        uploadInput.value = '';
        await self.loadFolder(self.state.currentFolderId);
      });
    }

    var sortSelect = this.root.querySelector('[data-role="sort-select"]');
    if (sortSelect) {
      sortSelect.addEventListener('change', function () {
        self.loadFolder(self.state.currentFolderId);
      });
    }

    this.root.addEventListener('click', async function (e) {
      var crumb = e.target.closest('.sb-disk__crumb');
      if (crumb) {
        var folderId = Number(crumb.getAttribute('data-folder-id') || 0);
        if (folderId > 0) {
          await self.loadFolder(folderId);
        }
      }

      var rowAction = e.target.closest('[data-row-action="open"]');
      if (rowAction) {
        var row = e.target.closest('.sb-disk__row');
        if (!row) return;

        var entityType = row.getAttribute('data-entity-type');
        var id = Number(row.getAttribute('data-id') || 0);

        if (entityType === 'folder' && id > 0) {
          await self.loadFolder(id);
        }
      }
    });
  };

  DiskComponent.prototype.getSortBy = function () {
    var select = this.root.querySelector('[data-role="sort-select"]');
    if (!select || !select.value) return 'updatedAt';
    return String(select.value).split(':')[0] || 'updatedAt';
  };

  DiskComponent.prototype.getSortDir = function () {
    var select = this.root.querySelector('[data-role="sort-select"]');
    if (!select || !select.value) return 'desc';
    return String(select.value).split(':')[1] || 'desc';
  };

  DiskComponent.prototype.renderAll = function () {
    this.renderBreadcrumbs();
    this.renderItemsTable();
    this.renderSubtitle();
    this.renderBulkbar();
  };

  DiskComponent.prototype.renderBreadcrumbs = function () {
    var container = this.root.querySelector('[data-role="breadcrumbs"]');
    if (!container) return;

    container.innerHTML = this.state.breadcrumbs.map(function (item) {
      return '<button type="button" class="sb-disk__crumb" data-folder-id="' + item.id + '">' + escapeHtml(item.name) + '</button>';
    }).join('<span>/</span>');
  };

  DiskComponent.prototype.renderItemsTable = function () {
    var tbody = this.root.querySelector('[data-role="items-table"]');
    if (!tbody) return;

    tbody.innerHTML = this.state.items.map(function (item) {
      var typeText = item.entityType === 'folder' ? 'Папка' : (item.extension || 'Файл');
      var sizeText = item.size ? String(item.size) : '';

      return '' +
        '<tr class="sb-disk__row" data-id="' + item.id + '" data-entity-type="' + item.entityType + '">' +
          '<td><input type="checkbox" class="sb-disk__check"></td>' +
          '<td>' + escapeHtml(item.name) + '</td>' +
          '<td>' + escapeHtml(typeText) + '</td>' +
          '<td>' + escapeHtml(sizeText) + '</td>' +
          '<td>' + escapeHtml(item.updatedAt || '') + '</td>' +
          '<td><button type="button" class="sb-disk__btn sb-disk__btn--ghost" data-row-action="open">Открыть</button></td>' +
        '</tr>';
    }).join('');
  };

  DiskComponent.prototype.renderSubtitle = function () {
    var node = this.root.querySelector('[data-role="subtitle"]');
    if (!node) return;

    var count = this.state.items.length;
    node.textContent = count + ' эл.';
  };

  DiskComponent.prototype.renderBulkbar = function () {
    var bulkbar = this.root.querySelector('[data-role="bulkbar"]');
    var textNode = this.root.querySelector('[data-role="bulkbar-text"]');

    if (!bulkbar || !textNode) return;

    textNode.textContent = 'Выбрано: ' + this.state.selectedIds.length;
    bulkbar.hidden = !this.state.selectedIds.length;
  };

  DiskComponent.prototype.setLoading = function (loading) {
    this.state.loading = loading;
    this.renderState(loading ? 'loading' : null);
  };

  DiskComponent.prototype.renderState = function (stateName) {
    var nodes = this.root.querySelectorAll('[data-state]');
    nodes.forEach(function (node) {
      node.hidden = true;
    });

    if (!stateName) return;

    var node = this.root.querySelector('[data-state="' + stateName + '"]');
    if (node) node.hidden = false;
  };

  function escapeHtml(str) {
    return String(str == null ? '' : str)
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

24. Регистрация блока в конструкторе

Ниже схема, как включить его в ваш sitebuilder.

Пример регистрации типа блока

$registry['disk'] = [
    'type' => 'disk',
    'name' => 'Диск',
    'icon' => 'disk',
    'componentPath' => '/local/sitebuilder/components/disk/',
    'category' => 'content',
    'supportsSettings' => true,
    'supportsPermissions' => true,
];


---

Пример рендера страницы

foreach ($pageBlocks as $block) {
    switch ($block['type']) {
        case 'disk':
            $component = new SitebuilderDiskComponent([
                'SITE_ID' => $siteId,
                'PAGE_ID' => $pageId,
                'BLOCK_ID' => $block['id'],
                'CURRENT_USER_ID' => $currentUserId,
            ]);
            $component->execute();
            break;

        default:
            // другие блоки
            break;
    }
}


---

25. Что обязательно проверить на сервере

Это критично, иначе компонент будет дырявым.

Для любого API-запроса:

siteId существует;

pageId принадлежит siteId;

blockId принадлежит pageId и siteId;

block.type == 'disk';

пользователь имеет доступ к сайту;

currentFolderId находится внутри root-дерева блока;

действие разрешено итоговыми правами.



---

26. Практические дефолты для первого запуска

При первом создании блока:

title = 'Файлы'

viewMode = 'table'

showSearch = true

showBreadcrumbs = true

allowUpload = true

allowCreateFolder = true

allowRename = true

allowDelete = false

allowDownload = true

permissionMode = 'inherit_site'

useSiteRootFallback = true


Это даст рабочий старт без лишней настройки.


---

27. Что я бы рекомендовал делать в вашей реализации сразу

Сразу заложить:

Обязательно

DiskStorageAdapterInterface

DiskValidator::assertFolderInsideRoot()

отдельную таблицу sitebuilder_disk_settings

отдельный DiskPermissionService

таблицу и плитку

breadcrumbs

поиск

массовые действия

модалки

состояния loading / empty / error / no-access / no-root


Не обязательно в первой версии, но держать в архитектуре

copy

drag & drop move

preview

versioning

audit log

comments

favorites

quota



---

28. Минимальный план реализации по этапам

Этап 1

Собрать основу:

таблица настроек

рендер блока

resolveRoot

getSettings

getPermissions

list


Этап 2

Добавить CRUD:

upload

createFolder

rename

delete


Этап 3

Добавить UX:

search

bulk actions

grid view

modals

error states


Этап 4

Доработать безопасность и масштабирование:

folder scope validation

block-level permissions

audit log

storage adapter integration



---

29. Что у вас получится на выходе

После реализации по этому каркасу у вас будет:

реально встраиваемый блок для конструктора;

изолированный по siteId/pageId/blockId;

не завязанный на тяжелый frontend;

расширяемый;

пригодный для корпоративного портала;

совместимый с подходом вашего sitebuilder.



---

30. Что могу сделать следующим сообщением

Я могу сразу собрать следующий слой, уже совсем прикладной:

вариант 1 — полный файл template.php + styles.css + script.js для стартового рабочего UI блока
вариант 2 — полный PHP API-скелет по всем action-файлам
вариант 3 — SQL + PHP-репозитории под ваш стиль sitebuilder
вариант 4 — готовое ТЗ для разработчика/команды в структурированном виде

Я бы следующим сообщением рекомендовал сделать именно полный стартовый комплект файлов компонента.