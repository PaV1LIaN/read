Ниже все actions/*.php, которые тебе нужны.


---

/local/sitebuilder/components/disk/actions/resolve_root.php

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

$result = DiskRootResolver::resolveWithSource($context, $settings);

DiskResponse::success([
    'rootFolderId' => $result['rootFolderId'],
    'source' => $result['source'],
]);


---

/local/sitebuilder/components/disk/actions/get_settings.php

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

DiskResponse::success([
    'settings' => $settings,
]);


---

/local/sitebuilder/components/disk/actions/save_settings.php

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

$currentSettings = DiskSettingsRepository::ensureExistsForBlock(
    $context->blockId,
    $context->siteId,
    $context->pageId,
    $context->currentUserId
);

$rootFolderId = DiskRootResolver::resolve($context, $currentSettings);
$permissions = DiskPermissionService::resolve($context, $currentSettings, $rootFolderId);

DiskValidator::assertCan($permissions, 'canEditSettings');

$settings = $data['settings'] ?? [];
if (!is_array($settings)) {
    throw new RuntimeException('INVALID_SETTINGS_PAYLOAD');
}

$allowedExtensions = [];
if (isset($settings['allowedExtensions'])) {
    if (is_array($settings['allowedExtensions'])) {
        $allowedExtensions = $settings['allowedExtensions'];
    } else {
        $raw = trim((string)$settings['allowedExtensions']);
        if ($raw !== '') {
            $allowedExtensions = preg_split('/[\s,;]+/u', $raw);
        }
    }
}

$rootFolderIdValue = null;
if (array_key_exists('rootFolderId', $settings)) {
    $tmp = trim((string)$settings['rootFolderId']);
    $rootFolderIdValue = ($tmp !== '' && (int)$tmp > 0) ? (int)$tmp : null;
}

$normalized = [
    'title' => trim((string)($settings['title'] ?? 'Файлы')),
    'rootFolderId' => $rootFolderIdValue,
    'viewMode' => in_array((string)($settings['viewMode'] ?? 'table'), ['table', 'grid'], true)
        ? (string)$settings['viewMode']
        : 'table',
    'allowUpload' => disk_normalize_bool($settings['allowUpload'] ?? true),
    'allowCreateFolder' => disk_normalize_bool($settings['allowCreateFolder'] ?? true),
    'allowRename' => disk_normalize_bool($settings['allowRename'] ?? true),
    'allowDelete' => disk_normalize_bool($settings['allowDelete'] ?? false),
    'allowDownload' => disk_normalize_bool($settings['allowDownload'] ?? true),
    'showSearch' => disk_normalize_bool($settings['showSearch'] ?? true),
    'showBreadcrumbs' => disk_normalize_bool($settings['showBreadcrumbs'] ?? true),
    'defaultSort' => trim((string)($settings['defaultSort'] ?? 'updatedAt')),
    'defaultSortDirection' => strtolower((string)($settings['defaultSortDirection'] ?? 'desc')) === 'asc' ? 'asc' : 'desc',
    'allowedExtensions' => array_values(array_filter(array_map(static function ($value) {
        return strtolower(trim((string)$value));
    }, $allowedExtensions))),
    'maxFileSize' => max(0, (int)($settings['maxFileSize'] ?? 52428800)),
    'permissionMode' => in_array((string)($settings['permissionMode'] ?? 'inherit_site'), ['inherit_site', 'custom'], true)
        ? (string)$settings['permissionMode']
        : 'inherit_site',
    'useSiteRootFallback' => disk_normalize_bool($settings['useSiteRootFallback'] ?? true),
];

DiskSettingsRepository::save($context->blockId, $normalized);

$updatedSettings = DiskSettingsRepository::getByBlockId($context->blockId);

DiskResponse::success([
    'settings' => $updatedSettings,
]);


---

/local/sitebuilder/components/disk/actions/get_permissions.php

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

DiskResponse::success([
    'permissions' => $permissions,
]);


---

/local/sitebuilder/components/disk/actions/get_root_options.php

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

$site = SiteRepository::getById($context->siteId);
$siteRootFolderId = SiteRepository::getRootDiskFolderId($context->siteId);

$options = [];

if ($siteRootFolderId) {
    $options[] = [
        'value' => '',
        'label' => 'Использовать корень сайта',
        'type' => 'site_root',
        'folderId' => (int)$siteRootFolderId,
    ];
}

if (!empty($settings['rootFolderId'])) {
    $options[] = [
        'value' => (int)$settings['rootFolderId'],
        'label' => 'Собственная папка блока',
        'type' => 'block_root',
        'folderId' => (int)$settings['rootFolderId'],
    ];
}

DiskResponse::success([
    'options' => $options,
    'siteRootFolderId' => $siteRootFolderId ? (int)$siteRootFolderId : null,
    'blockRootFolderId' => !empty($settings['rootFolderId']) ? (int)$settings['rootFolderId'] : null,
    'siteName' => $site ? (string)$site['name'] : '',
]);


---

/local/sitebuilder/components/disk/actions/list.php

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

/local/sitebuilder/components/disk/actions/upload.php

<?php

DiskCsrf::validateFromRequest();

$currentUserId = DiskCurrentUser::requireId();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($_POST['siteId'] ?? 0),
    'pageId' => (int)($_POST['pageId'] ?? 0),
    'blockId' => (int)($_POST['blockId'] ?? 0),
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

/local/sitebuilder/components/disk/actions/create_folder.php

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

/local/sitebuilder/components/disk/actions/rename.php

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

/local/sitebuilder/components/disk/actions/delete.php

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

/local/sitebuilder/components/disk/actions/move.php

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

/local/sitebuilder/components/disk/actions/copy.php

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

/local/sitebuilder/components/disk/actions/search.php

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

/local/sitebuilder/components/disk/actions/download.php

<?php

$currentUserId = DiskCurrentUser::requireId();

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? ($_GET['siteId'] ?? 0)),
    'pageId' => (int)($data['pageId'] ?? ($_GET['pageId'] ?? 0)),
    'blockId' => (int)($data['blockId'] ?? ($_GET['blockId'] ?? 0)),
    'currentUserId' => $currentUserId,
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

// Чтобы не уйти в цикл на тот же action, берем прямой URL Bitrix Disk:
$file = \Bitrix\Disk\File::loadById($fileId);
if (!$file instanceof \Bitrix\Disk\File) {
    throw new RuntimeException('DISK_FILE_NOT_FOUND');
}

LocalRedirect((string)$file->getDownloadUrl());
exit;


---

/local/sitebuilder/components/disk/actions/init_site_root.php

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

/local/sitebuilder/components/disk/actions/init_block_root.php

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

Следом пришлю оставшиеся три больших файла:

template.php

styles.css

script.js