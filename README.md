Отлично. Тогда делаем интеграционный пакет disk под текущий sitebuilder без отдельной таблицы настроек блока.

Цель:

disk берет siteId/pageId/blockId из блока страницы

настройки читает из sitebuilder.block.props_json

права берет из sitebuilder.access

root папки создает автоматически

инициализируется одним вызовом disk.bootstrap


Ниже даю что создать и что заменить.


---

1. Новый bridge между disk и sitebuilder

/local/sitebuilder/components/disk/lib/DiskSitebuilderBridge.php

<?php

class DiskSitebuilderBridge
{
    public static function getSiteById(int $siteId): ?array
    {
        $row = DiskDb::fetchOne("
            SELECT
                id,
                name,
                slug,
                home_page_id,
                disk_folder_id,
                top_menu_id,
                settings_json,
                layout_json,
                created_by,
                created_at,
                updated_by,
                updated_at
            FROM sitebuilder.site
            WHERE id = :id
            LIMIT 1
        ", [
            ':id' => $siteId,
        ]);

        if (!$row) {
            return null;
        }

        return [
            'id' => (int)$row['id'],
            'name' => (string)$row['name'],
            'slug' => (string)$row['slug'],
            'homePageId' => !empty($row['home_page_id']) ? (int)$row['home_page_id'] : 0,
            'diskFolderId' => !empty($row['disk_folder_id']) ? (int)$row['disk_folder_id'] : 0,
            'topMenuId' => !empty($row['top_menu_id']) ? (int)$row['top_menu_id'] : 0,
            'settings' => sb_json_decode_assoc($row['settings_json'] ?? '{}'),
            'layout' => sb_json_decode_assoc($row['layout_json'] ?? '{}'),
            'createdBy' => isset($row['created_by']) ? (int)$row['created_by'] : 0,
            'createdAt' => (string)($row['created_at'] ?? ''),
            'updatedBy' => isset($row['updated_by']) ? (int)$row['updated_by'] : 0,
            'updatedAt' => (string)($row['updated_at'] ?? ''),
        ];
    }

    public static function getPageById(int $pageId): ?array
    {
        $row = DiskDb::fetchOne("
            SELECT
                id,
                site_id,
                title,
                slug,
                parent_id,
                sort,
                status,
                published_at,
                created_by,
                created_at,
                updated_by,
                updated_at
            FROM sitebuilder.page
            WHERE id = :id
            LIMIT 1
        ", [
            ':id' => $pageId,
        ]);

        if (!$row) {
            return null;
        }

        return [
            'id' => (int)$row['id'],
            'siteId' => (int)$row['site_id'],
            'title' => (string)$row['title'],
            'slug' => (string)$row['slug'],
            'parentId' => !empty($row['parent_id']) ? (int)$row['parent_id'] : 0,
            'sort' => (int)($row['sort'] ?? 500),
            'status' => (string)($row['status'] ?? 'draft'),
            'publishedAt' => !empty($row['published_at']) ? (string)$row['published_at'] : null,
            'createdBy' => isset($row['created_by']) ? (int)$row['created_by'] : 0,
            'createdAt' => (string)($row['created_at'] ?? ''),
            'updatedBy' => isset($row['updated_by']) ? (int)$row['updated_by'] : 0,
            'updatedAt' => (string)($row['updated_at'] ?? ''),
        ];
    }

    public static function getBlockById(int $blockId): ?array
    {
        $row = DiskDb::fetchOne("
            SELECT
                id,
                page_id,
                type,
                sort,
                content_json,
                props_json,
                created_by,
                created_at,
                updated_by,
                updated_at
            FROM sitebuilder.block
            WHERE id = :id
            LIMIT 1
        ", [
            ':id' => $blockId,
        ]);

        if (!$row) {
            return null;
        }

        return [
            'id' => (int)$row['id'],
            'pageId' => (int)$row['page_id'],
            'type' => (string)$row['type'],
            'sort' => (int)($row['sort'] ?? 500),
            'content' => sb_json_decode_assoc($row['content_json'] ?? '{}'),
            'props' => sb_json_decode_assoc($row['props_json'] ?? '{}'),
            'createdBy' => isset($row['created_by']) ? (int)$row['created_by'] : 0,
            'createdAt' => (string)($row['created_at'] ?? ''),
            'updatedBy' => isset($row['updated_by']) ? (int)$row['updated_by'] : 0,
            'updatedAt' => (string)($row['updated_at'] ?? ''),
        ];
    }

    public static function saveBlockProps(int $blockId, array $props): bool
    {
        return DiskDb::execute("
            UPDATE sitebuilder.block
            SET
                props_json = :props_json::jsonb,
                updated_at = CURRENT_TIMESTAMP
            WHERE id = :id
        ", [
            ':id' => $blockId,
            ':props_json' => json_encode($props, JSON_UNESCAPED_UNICODE),
        ]);
    }

    public static function updateSiteDiskFolderId(int $siteId, int $folderId): bool
    {
        return DiskDb::execute("
            UPDATE sitebuilder.site
            SET
                disk_folder_id = :disk_folder_id,
                updated_at = CURRENT_TIMESTAMP
            WHERE id = :id
        ", [
            ':id' => $siteId,
            ':disk_folder_id' => $folderId,
        ]);
    }

    public static function normalizeDiskProps(array $props): array
    {
        return [
            'title' => trim((string)($props['title'] ?? 'Файлы')),
            'rootMode' => in_array((string)($props['rootMode'] ?? 'site'), ['site', 'block'], true)
                ? (string)$props['rootMode']
                : 'site',
            'rootFolderId' => !empty($props['rootFolderId']) ? (int)$props['rootFolderId'] : null,
            'viewMode' => in_array((string)($props['viewMode'] ?? 'table'), ['table', 'grid'], true)
                ? (string)$props['viewMode']
                : 'table',
            'allowUpload' => !array_key_exists('allowUpload', $props) || !empty($props['allowUpload']),
            'allowCreateFolder' => !array_key_exists('allowCreateFolder', $props) || !empty($props['allowCreateFolder']),
            'allowRename' => !array_key_exists('allowRename', $props) || !empty($props['allowRename']),
            'allowDelete' => !empty($props['allowDelete']),
            'allowDownload' => !array_key_exists('allowDownload', $props) || !empty($props['allowDownload']),
            'showSearch' => !array_key_exists('showSearch', $props) || !empty($props['showSearch']),
            'showBreadcrumbs' => !array_key_exists('showBreadcrumbs', $props) || !empty($props['showBreadcrumbs']),
            'defaultSort' => trim((string)($props['defaultSort'] ?? 'updatedAt')),
            'defaultSortDirection' => strtolower((string)($props['defaultSortDirection'] ?? 'desc')) === 'asc' ? 'asc' : 'desc',
            'allowedExtensions' => is_array($props['allowedExtensions'] ?? null) ? array_values($props['allowedExtensions']) : [],
            'maxFileSize' => max(0, (int)($props['maxFileSize'] ?? 52428800)),
            'permissionMode' => in_array((string)($props['permissionMode'] ?? 'inherit_site'), ['inherit_site', 'custom'], true)
                ? (string)$props['permissionMode']
                : 'inherit_site',
            'useSiteRootFallback' => !array_key_exists('useSiteRootFallback', $props) || !empty($props['useSiteRootFallback']),
        ];
    }
}


---

2. Подключить bridge в bootstrap.php

/local/sitebuilder/components/disk/bootstrap.php

Добавь:

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/db.php';

и ниже среди require_once:

require_once __DIR__ . '/lib/DiskSitebuilderBridge.php';

Если sb_json_decode_assoc() уже в db.php, этого достаточно.


---

3. Заменить SiteRepository.php

Теперь disk не должен жить своей отдельной моделью сайта.

/local/sitebuilder/components/disk/lib/SiteRepository.php

<?php

class SiteRepository
{
    public static function getById(int $siteId): ?array
    {
        return DiskSitebuilderBridge::getSiteById($siteId);
    }

    public static function getRootDiskFolderId(int $siteId): ?int
    {
        $site = self::getById($siteId);
        if (!$site) {
            return null;
        }

        return !empty($site['diskFolderId']) ? (int)$site['diskFolderId'] : null;
    }

    public static function updateRootDiskFolderId(int $siteId, ?int $folderId): bool
    {
        if (!$folderId || $folderId <= 0) {
            throw new RuntimeException('INVALID_FOLDER_ID');
        }

        return DiskSitebuilderBridge::updateSiteDiskFolderId($siteId, (int)$folderId);
    }
}


---

4. Заменить BlockRepository.php

Теперь блок тоже берем из sitebuilder.block.

/local/sitebuilder/components/disk/lib/BlockRepository.php

<?php

class BlockRepository
{
    public static function getById(int $blockId): ?array
    {
        return DiskSitebuilderBridge::getBlockById($blockId);
    }

    public static function getDiskBlockByContext(int $siteId, int $pageId, int $blockId): ?array
    {
        $block = self::getById($blockId);
        if (!$block) {
            return null;
        }

        if ((string)($block['type'] ?? '') !== 'disk') {
            return null;
        }

        if ((int)($block['pageId'] ?? 0) !== $pageId) {
            return null;
        }

        $page = DiskSitebuilderBridge::getPageById($pageId);
        if (!$page) {
            return null;
        }

        if ((int)($page['siteId'] ?? 0) !== $siteId) {
            return null;
        }

        return $block;
    }

    public static function updateSettingsJson(int $blockId, array $settings): bool
    {
        return DiskSitebuilderBridge::saveBlockProps($blockId, $settings);
    }
}


---

5. Больше не использовать DiskSettingsRepository как отдельную таблицу

Теперь настройки disk — это props_json блока.

/local/sitebuilder/components/disk/lib/DiskSettingsRepository.php

Полностью замени на:

<?php

class DiskSettingsRepository
{
    public static function getByBlockId(int $blockId): array
    {
        $block = DiskSitebuilderBridge::getBlockById($blockId);
        if (!$block) {
            throw new RuntimeException('BLOCK_NOT_FOUND');
        }

        return DiskSitebuilderBridge::normalizeDiskProps($block['props'] ?? []);
    }

    public static function createDefault(array $data): bool
    {
        $blockId = (int)($data['block_id'] ?? 0);
        if ($blockId <= 0) {
            throw new RuntimeException('INVALID_BLOCK_ID');
        }

        $default = DiskSitebuilderBridge::normalizeDiskProps([]);
        return DiskSitebuilderBridge::saveBlockProps($blockId, $default);
    }

    public static function save(int $blockId, array $settings): bool
    {
        $current = self::getByBlockId($blockId);
        $merged = array_merge($current, $settings);
        $normalized = DiskSitebuilderBridge::normalizeDiskProps($merged);

        return DiskSitebuilderBridge::saveBlockProps($blockId, $normalized);
    }

    public static function ensureExistsForBlock(int $blockId, int $siteId, int $pageId, ?int $createdBy = null): array
    {
        $block = DiskSitebuilderBridge::getBlockById($blockId);
        if (!$block) {
            throw new RuntimeException('BLOCK_NOT_FOUND');
        }

        $props = $block['props'] ?? [];
        $normalized = DiskSitebuilderBridge::normalizeDiskProps($props);

        if (($block['props'] ?? []) !== $normalized) {
            DiskSitebuilderBridge::saveBlockProps($blockId, $normalized);
        }

        return $normalized;
    }
}


---

6. Новый единый disk.bootstrap

Новый файл /local/sitebuilder/components/disk/actions/bootstrap.php

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

$rootInfo = DiskRootResolver::resolveWithSource($context, $settings, true);
$permissions = DiskPermissionService::resolve($context, $settings, $rootInfo['rootFolderId']);

DiskResponse::success([
    'siteId' => $context->siteId,
    'pageId' => $context->pageId,
    'blockId' => $context->blockId,
    'settings' => $settings,
    'permissions' => $permissions,
    'rootFolderId' => $rootInfo['rootFolderId'],
    'currentFolderId' => $rootInfo['rootFolderId'],
    'rootSource' => $rootInfo['source'],
]);


---

7. Обновить DiskRootResolver.php

Теперь он должен уметь:

брать disk_folder_id сайта

создавать корень сайта

создавать папку блока

сохранять rootFolderId обратно в props_json


/local/sitebuilder/components/disk/lib/DiskRootResolver.php

<?php

class DiskRootResolver
{
    public static function resolve(DiskContext $context, array $settings, bool $autoCreate = false): ?int
    {
        $result = self::resolveWithSource($context, $settings, $autoCreate);
        return $result['rootFolderId'];
    }

    public static function resolveWithSource(DiskContext $context, array $settings, bool $autoCreate = false): array
    {
        $rootMode = (string)($settings['rootMode'] ?? 'site');

        if ($rootMode === 'block') {
            if (!empty($settings['rootFolderId'])) {
                return [
                    'rootFolderId' => (int)$settings['rootFolderId'],
                    'source' => 'block',
                ];
            }

            if ($autoCreate) {
                $folderId = BlockDiskInitializer::ensureBlockRootFolder(
                    $context->siteId,
                    $context->pageId,
                    $context->blockId,
                    $context->currentUserId,
                    (string)($settings['title'] ?? '')
                );

                return [
                    'rootFolderId' => $folderId,
                    'source' => 'block',
                ];
            }
        }

        $siteRootFolderId = SiteRepository::getRootDiskFolderId($context->siteId);
        if ($siteRootFolderId !== null && $siteRootFolderId > 0) {
            return [
                'rootFolderId' => $siteRootFolderId,
                'source' => 'site',
            ];
        }

        if ($autoCreate && !empty($settings['useSiteRootFallback'])) {
            $site = SiteRepository::getById($context->siteId);
            if (!$site) {
                throw new RuntimeException('SITE_NOT_FOUND');
            }

            $folderId = SiteDiskInitializer::ensureSiteRootFolder(
                $context->siteId,
                $context->currentUserId,
                (string)$site['name']
            );

            return [
                'rootFolderId' => $folderId,
                'source' => 'site',
            ];
        }

        return [
            'rootFolderId' => null,
            'source' => 'none',
        ];
    }
}


---

8. Обновить BlockDiskInitializer.php

Важно: теперь он должен писать rootFolderId в sitebuilder.block.props_json.

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
            'rootMode' => 'block',
            'rootFolderId' => (int)$created->getId(),
            'useSiteRootFallback' => true,
        ]);

        return (int)$created->getId();
    }
}


---

9. Добавить bootstrap action в api.php

/local/sitebuilder/components/disk/api.php

В switch добавь:

case 'bootstrap':
    require __DIR__ . '/actions/bootstrap.php';
    break;


---

10. Как встраивать блок в сайт

Теперь на странице sitebuilder рендер блока disk должен быть таким:

<?php
$blockId = (int)$block['id'];
$pageId = (int)$page['id'];
$siteId = (int)$site['id'];
$props = is_array($block['props'] ?? null) ? $block['props'] : [];
$title = (string)($props['title'] ?? 'Файлы');
?>
<div class="sb-disk"
     data-site-id="<?= $siteId ?>"
     data-page-id="<?= $pageId ?>"
     data-block-id="<?= $blockId ?>"
     data-initial-state="<?= htmlspecialchars(json_encode([
         'siteId' => $siteId,
         'pageId' => $pageId,
         'blockId' => $blockId,
         'settings' => $props,
     ], JSON_UNESCAPED_UNICODE), ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>">
</div>

И на фронте script.js при инициализации должен первым делом вызывать:

api('bootstrap', {
  siteId: this.state.siteId,
  pageId: this.state.pageId,
  blockId: this.state.blockId,
  sessid: this.getSessid()
})


---

11. Что это даст

После этого disk будет:

сам получать siteId/pageId/blockId из текущего блока

сам тянуть настройки из sitebuilder.block.props_json

сам подтягивать права пользователя из sitebuilder.access

сам создавать корень сайта и папку блока при первом запуске

быть полностью встроенным в sitebuilder, а не жить отдельно



---

12. Что делать следующим шагом

Теперь уже логично сделать реальный case 'disk' в renderer страницы и editor renderer, чтобы:

блок можно было добавлять как type = disk

у него открывались настройки

и он жил как полноценный блок конструктора


Следующим сообщением я могу дать тебе готовый renderer + editor integration для блока disk.