<?php
define('NO_KEEP_STATISTIC', true);
define('NO_AGENT_STATISTIC', true);
define('DisableEventsCheck', true);

require $_SERVER['DOCUMENT_ROOT'].'/bitrix/modules/main/include/prolog_before.php';

use Bitrix\Main\Loader;
use Bitrix\Disk\Storage;
use Bitrix\Disk\Folder;
use Bitrix\Disk\File;
use Bitrix\Disk\Driver;

global $USER;

header('Content-Type: application/json; charset=UTF-8');

if (!$USER->IsAuthorized()) {
    http_response_code(401);
    echo json_encode(['ok' => false, 'error' => 'NOT_AUTHORIZED'], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    http_response_code(405);
    echo json_encode(['ok' => false, 'error' => 'METHOD_NOT_ALLOWED'], JSON_UNESCAPED_UNICODE);
    exit;
}

if (!check_bitrix_sessid()) {
    http_response_code(403);
    echo json_encode(['ok' => false, 'error' => 'BAD_SESSID'], JSON_UNESCAPED_UNICODE);
    exit;
}

/** ====================== JSON STORAGE ====================== */

function sb_data_path(string $file): string {
    return $_SERVER['DOCUMENT_ROOT'] . '/upload/sitebuilder/' . $file;
}

function sb_read_json_file(string $file): array {
    $path = sb_data_path($file);
    if (!file_exists($path)) return [];
    $raw = file_get_contents($path);
    $data = json_decode((string)$raw, true);
    return is_array($data) ? $data : [];
}

function sb_write_json_file(string $file, array $data, string $errMsg): void {
    $dir = dirname(sb_data_path($file));
    if (!is_dir($dir)) {
        mkdir($dir, 0775, true);
    }

    $path = sb_data_path($file);
    $fp = fopen($path, 'c+');
    if (!$fp) {
        throw new \RuntimeException($errMsg);
    }

    if (!flock($fp, LOCK_EX)) {
        fclose($fp);
        throw new \RuntimeException('Cannot lock ' . $file);
    }

    ftruncate($fp, 0);
    rewind($fp);
    fwrite($fp, json_encode(array_values($data), JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT));
    fflush($fp);
    flock($fp, LOCK_UN);
    fclose($fp);
}

function sb_read_sites(): array { return sb_read_json_file('sites.json'); }
function sb_write_sites(array $sites): void { sb_write_json_file('sites.json', $sites, 'Cannot open sites.json'); }

function sb_read_pages(): array { return sb_read_json_file('pages.json'); }
function sb_write_pages(array $pages): void { sb_write_json_file('pages.json', $pages, 'Cannot open pages.json'); }

function sb_read_blocks(): array { return sb_read_json_file('blocks.json'); }
function sb_write_blocks(array $blocks): void { sb_write_json_file('blocks.json', $blocks, 'Cannot open blocks.json'); }

function sb_read_access(): array { return sb_read_json_file('access.json'); }
function sb_write_access(array $access): void { sb_write_json_file('access.json', $access, 'Cannot open access.json'); }

function sb_read_menus(): array { return sb_read_json_file('menus.json'); }
function sb_write_menus(array $menus): void { sb_write_json_file('menus.json', $menus, 'Cannot open menus.json'); }

function sb_slugify(string $name): string {
    $slug = \CUtil::translit($name, 'ru', [
        'replace_space' => '-',
        'replace_other' => '-',
        'change_case' => 'L',
        'delete_repeat_replace' => true,
        'use_google' => false,
    ]);
    $slug = trim($slug, '-');
    return $slug !== '' ? $slug : 'item';
}

/** ====================== ACCESS MODEL ====================== */

function sb_user_access_code(): string {
    return 'U' . (int)$GLOBALS['USER']->GetID();
}

function sb_get_role(int $siteId, string $accessCode): ?string {
    $access = sb_read_access();
    foreach ($access as $r) {
        if ((int)($r['siteId'] ?? 0) === $siteId && (string)($r['accessCode'] ?? '') === $accessCode) {
            $role = strtoupper((string)($r['role'] ?? ''));
            return $role !== '' ? $role : null;
        }
    }
    return null;
}

function sb_role_rank(?string $role): int {
    $role = strtoupper((string)$role);
    return match ($role) {
        'OWNER'  => 4,
        'ADMIN'  => 3,
        'EDITOR' => 2,
        'VIEWER' => 1,
        default  => 0,
    };
}

function sb_require_site_role(int $siteId, int $minRank): void {
    $role = sb_get_role($siteId, sb_user_access_code());
    if (sb_role_rank($role) < $minRank) {
        http_response_code(403);
        echo json_encode([
            'ok' => false,
            'error' => 'FORBIDDEN',
            'siteId' => $siteId,
            'role' => $role,
        ], JSON_UNESCAPED_UNICODE);
        exit;
    }
}

function sb_require_owner(int $siteId): void { sb_require_site_role($siteId, 4); }
function sb_require_editor(int $siteId): void { sb_require_site_role($siteId, 2); }
function sb_require_viewer(int $siteId): void { sb_require_site_role($siteId, 1); }

function sb_site_exists(int $siteId): bool {
    foreach (sb_read_sites() as $s) {
        if ((int)($s['id'] ?? 0) === $siteId) return true;
    }
    return false;
}

function sb_find_page(int $pageId): ?array {
    foreach (sb_read_pages() as $p) {
        if ((int)($p['id'] ?? 0) === $pageId) return $p;
    }
    return null;
}

function sb_find_block(int $blockId): ?array {
    foreach (sb_read_blocks() as $b) {
        if ((int)($b['id'] ?? 0) === $blockId) return $b;
    }
    return null;
}

/** ====================== DISK (FILES) ====================== */

function sb_disk_add_subfolder(Folder $parent, array $fields, Storage $storage): Folder {
    // NEW: addSubFolder($fields, array $rights)
    try {
        $folder = $parent->addSubFolder($fields, []);
        if ($folder instanceof Folder) return $folder;
    } catch (\Throwable $e) {}

    // OLD: addSubFolder($fields, SecurityContext)
    $ctx = $storage->getSecurityContext($GLOBALS['USER']);
    $folder = $parent->addSubFolder($fields, $ctx);
    if ($folder instanceof Folder) return $folder;

    throw new \RuntimeException('CANNOT_CREATE_FOLDER');
}

function sb_disk_upload_file(Folder $folder, array $fileArray, array $fields, Storage $storage): File {
    // NEW: uploadFile($fileArray, $fields, array $rights)
    try {
        $obj = $folder->uploadFile($fileArray, $fields, []);
        if ($obj instanceof File) return $obj;
    } catch (\Throwable $e) {}

    // OLD: uploadFile($fileArray, $fields, SecurityContext)
    $ctx = $storage->getSecurityContext($GLOBALS['USER']);
    $obj = $folder->uploadFile($fileArray, $fields, $ctx);
    if ($obj instanceof File) return $obj;

    throw new \RuntimeException('UPLOAD_FAILED');
}

function sb_disk_get_children(Folder $folder, Storage $storage): array {
    // NEW: getChildren(SecurityContext $context, array $params = [])
    try {
        $ctx = $storage->getSecurityContext($GLOBALS['USER']);
        return $folder->getChildren($ctx);
    } catch (\Throwable $e) {}
    // OLD: getChildren()
    return $folder->getChildren();
}

function sb_disk_common_storage(): Storage {
    if (!Loader::includeModule('disk')) {
        throw new \RuntimeException('DISK_NOT_INSTALLED');
    }

    // Newer Bitrix
    if (method_exists(\Bitrix\Disk\Storage::class, 'loadByEntity')) {
        $storage = \Bitrix\Disk\Storage::loadByEntity('common', 0);
        if ($storage) return $storage;
    }

    // Older/alternative
    $driver = \Bitrix\Disk\Driver::getInstance();
    if (method_exists($driver, 'getStorageByCommonId')) {
        $storage = $driver->getStorageByCommonId('shared_files_' . SITE_ID);
        if ($storage) return $storage;
    }

    throw new \RuntimeException('COMMON_STORAGE_NOT_FOUND');
}

function sb_disk_get_or_create_root(Storage $storage, string $name): Folder {
    $root = $storage->getRootObject();

    $child = $root->getChild([
        '=NAME' => $name,
        '=TYPE' => \Bitrix\Disk\Internals\ObjectTable::TYPE_FOLDER,
    ]);
    if ($child instanceof Folder) {
        return $child;
    }

    return sb_disk_add_subfolder($root, [
        'NAME' => $name,
        'CREATED_BY' => (int)$GLOBALS['USER']->GetID(),
    ], $storage);
}

function sb_disk_ensure_site_folder(int $siteId): Folder {
    $sites = sb_read_sites();
    $site = null; $siteIndex = null;

    foreach ($sites as $i => $s) {
        if ((int)($s['id'] ?? 0) === $siteId) {
            $site = $s; $siteIndex = $i; break;
        }
    }
    if (!$site) throw new \RuntimeException('SITE_NOT_FOUND');

    $storage = sb_disk_common_storage();

    $diskFolderId = (int)($site['diskFolderId'] ?? 0);
    if ($diskFolderId > 0) {
        $folder = Folder::loadById($diskFolderId);
        if ($folder) return $folder;
    }

    $root = sb_disk_get_or_create_root($storage, 'SiteBuilder');

    $slug = (string)($site['slug'] ?? ('site-' . $siteId));
    $slug = $slug !== '' ? $slug : ('site-' . $siteId);

    $existing = $root->getChild([
        '=NAME' => $slug,
        '=TYPE' => \Bitrix\Disk\Internals\ObjectTable::TYPE_FOLDER,
    ]);

    if ($existing instanceof Folder) {
        $folder = $existing;
    } else {
        $folder = sb_disk_add_subfolder($root, [
            'NAME' => $slug,
            'CREATED_BY' => (int)$GLOBALS['USER']->GetID(),
        ], $storage);
    }

    $sites[$siteIndex]['diskFolderId'] = (int)$folder->getId();
    sb_write_sites($sites);

    return $folder;
}

/**
 * Best-effort: не падаем, если в конкретной версии нет setRights()
 */
function sb_disk_sync_folder_rights(int $siteId, Folder $folder): void {
    $rm = Driver::getInstance()->getRightsManager();
    if (!method_exists($rm, 'setRights')) {
        return;
    }

    $taskRead = (method_exists($rm, 'getTaskIdByName') && defined(get_class($rm).'::TASK_READ'))
        ? $rm->getTaskIdByName($rm::TASK_READ) : null;
    $taskEdit = (method_exists($rm, 'getTaskIdByName') && defined(get_class($rm).'::TASK_EDIT'))
        ? $rm->getTaskIdByName($rm::TASK_EDIT) : null;
    $taskFull = (method_exists($rm, 'getTaskIdByName') && defined(get_class($rm).'::TASK_FULL'))
        ? $rm->getTaskIdByName($rm::TASK_FULL) : null;

    if (!$taskRead || !$taskEdit || !$taskFull) return;

    $acc = sb_read_access();
    $acc = array_values(array_filter($acc, fn($r) => (int)($r['siteId'] ?? 0) === $siteId));

    $rights = [];
    foreach ($acc as $r) {
        $code = (string)($r['accessCode'] ?? '');
        if ($code === '') continue;

        $role = strtoupper((string)($r['role'] ?? 'VIEWER'));
        $taskId = $taskRead;
        if ($role === 'EDITOR') $taskId = $taskEdit;
        if ($role === 'ADMIN' || $role === 'OWNER') $taskId = $taskFull;

        $rights[] = ['ACCESS_CODE' => $code, 'TASK_ID' => $taskId];
    }

    $rm->setRights($folder, $rights);
}

/** ===== File belongs check for image blocks ===== */

function sb_disk_file_belongs_to_site(int $siteId, int $fileId): bool {
    if ($fileId <= 0) return false;
    $folder = sb_disk_ensure_site_folder($siteId);
    $file = \Bitrix\Disk\File::loadById($fileId);
    if (!$file) return false;
    return ((int)$file->getParentId() === (int)$folder->getId());
}

/** ====================== MENU HELPERS ====================== */

function sb_menu_get_site_record(array $all, int $siteId): ?array {
    foreach ($all as $r) {
        if ((int)($r['siteId'] ?? 0) === $siteId) return $r;
    }
    return null;
}

function sb_menu_upsert_site_record(array &$all, int $siteId, array $record): void {
    $found = false;
    foreach ($all as $i => $r) {
        if ((int)($r['siteId'] ?? 0) === $siteId) {
            $all[$i] = $record;
            $found = true;
            break;
        }
    }
    if (!$found) $all[] = $record;
}

function sb_menu_next_menu_id(array $siteRecord): int {
    $max = 0;
    foreach (($siteRecord['menus'] ?? []) as $m) {
        $max = max($max, (int)($m['id'] ?? 0));
    }
    return $max + 1;
}

function sb_menu_next_item_id(array $menu): int {
    $max = 0;
    foreach (($menu['items'] ?? []) as $it) {
        $max = max($max, (int)($it['id'] ?? 0));
    }
    return $max + 1;
}

function sb_menu_next_sort(array $items): int {
    $max = 0;
    foreach ($items as $it) $max = max($max, (int)($it['sort'] ?? 0));
    return $max + 10;
}

function sb_menu_find_menu(array $siteRecord, int $menuId): ?array {
    foreach (($siteRecord['menus'] ?? []) as $m) {
        if ((int)($m['id'] ?? 0) === $menuId) return $m;
    }
    return null;
}

function sb_menu_update_menu(array &$siteRecord, int $menuId, callable $fn): bool {
    $menus = $siteRecord['menus'] ?? [];
    $changed = false;

    foreach ($menus as $i => $m) {
        if ((int)($m['id'] ?? 0) === $menuId) {
            $menus[$i] = $fn($m);
            $changed = true;
            break;
        }
    }

    if ($changed) $siteRecord['menus'] = $menus;
    return $changed;
}

/** ====================== ACTIONS ====================== */

$action = (string)($_POST['action'] ?? '');

// ping
if ($action === 'ping') {
    echo json_encode([
        'ok' => true,
        'time' => date('c'),
        'userId' => (int)$USER->GetID(),
        'login' => (string)$USER->GetLogin(),
    ], JSON_UNESCAPED_UNICODE);
    exit;
}

/** -------------------- SITES -------------------- */

if ($action === 'site.list') {
    $sites = sb_read_sites();
    $myCode = sb_user_access_code();

    $access = sb_read_access();
    $allowedSiteIds = [];
    foreach ($access as $r) {
        if ((string)($r['accessCode'] ?? '') === $myCode) {
            $sid = (int)($r['siteId'] ?? 0);
            if ($sid > 0) $allowedSiteIds[$sid] = true;
        }
    }

    $sites = array_values(array_filter($sites, fn($s) => isset($allowedSiteIds[(int)($s['id'] ?? 0)])));
    usort($sites, fn($a, $b) => (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0));

    echo json_encode(['ok' => true, 'sites' => $sites], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'site.create') {
    $name = trim((string)($_POST['name'] ?? ''));
    if ($name === '') {
        http_response_code(422);
        echo json_encode(['ok' => false, 'error' => 'NAME_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $sites = sb_read_sites();
    $maxId = 0;
    foreach ($sites as $s) $maxId = max($maxId, (int)($s['id'] ?? 0));
    $id = $maxId + 1;

    $slug = trim((string)($_POST['slug'] ?? ''));
    $slug = $slug === '' ? sb_slugify($name) : sb_slugify($slug);

    $existing = array_map(fn($x) => (string)($x['slug'] ?? ''), $sites);
    $base = $slug; $i = 2;
    while (in_array($slug, $existing, true)) { $slug = $base.'-'.$i; $i++; }

    $site = [
        'id' => $id,
        'name' => $name,
        'slug' => $slug,
        'createdBy' => (int)$USER->GetID(),
        'createdAt' => date('c'),
        'diskFolderId' => 0,
    ];

    $sites[] = $site;
    sb_write_sites($sites);

    $access = sb_read_access();
    $access[] = [
        'siteId' => $id,
        'accessCode' => 'U' . (int)$USER->GetID(),
        'role' => 'OWNER',
        'createdBy' => (int)$USER->GetID(),
        'createdAt' => date('c'),
    ];
    sb_write_access($access);

    echo json_encode(['ok' => true, 'site' => $site], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'site.delete') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) {
        http_response_code(422);
        echo json_encode(['ok' => false, 'error' => 'ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    sb_require_owner($id);

    $sites = sb_read_sites();
    $before = count($sites);
    $sites = array_values(array_filter($sites, fn($s) => (int)($s['id'] ?? 0) !== $id));
    if (count($sites) === $before) {
        http_response_code(404);
        echo json_encode(['ok' => false, 'error' => 'NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }
    sb_write_sites($sites);

    $pages = sb_read_pages();
    $pages = array_values(array_filter($pages, fn($p) => (int)($p['siteId'] ?? 0) !== $id));
    sb_write_pages($pages);

    $blocks = sb_read_blocks();
    $pagesNow = sb_read_pages();
    $pageIdsNow = [];
    foreach ($pagesNow as $p) $pageIdsNow[(int)($p['id'] ?? 0)] = true;
    $blocks = array_values(array_filter($blocks, fn($b) => isset($pageIdsNow[(int)($b['pageId'] ?? 0)])));
    sb_write_blocks($blocks);

    $acc = sb_read_access();
    $acc = array_values(array_filter($acc, fn($r) => (int)($r['siteId'] ?? 0) !== $id));
    sb_write_access($acc);

    echo json_encode(['ok' => true], JSON_UNESCAPED_UNICODE);
    exit;
}

/** -------------------- PAGES -------------------- */

if ($action === 'page.list') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    sb_require_viewer($siteId);

    $pages = sb_read_pages();
    $pages = array_values(array_filter($pages, fn($p) => (int)($p['siteId'] ?? 0) === $siteId));
    usort($pages, fn($a, $b) => (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0));

    echo json_encode(['ok' => true, 'pages' => $pages], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'page.create') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $title  = trim((string)($_POST['title'] ?? ''));
    $slugIn = trim((string)($_POST['slug'] ?? ''));

    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($title === '') { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'TITLE_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    sb_require_editor($siteId);

    $slug = $slugIn !== '' ? sb_slugify($slugIn) : sb_slugify($title);

    $pages = sb_read_pages();
    $maxId = 0; foreach ($pages as $p) $maxId = max($maxId, (int)($p['id'] ?? 0));
    $id = $maxId + 1;

    $existing = array_map(fn($x) => (string)($x['slug'] ?? ''), array_filter($pages, fn($p) => (int)($p['siteId'] ?? 0) === $siteId));
    $base = $slug; $i = 2;
    while (in_array($slug, $existing, true)) { $slug = $base.'-'.$i; $i++; }

    $page = [
        'id' => $id,
        'siteId' => $siteId,
        'title' => $title,
        'slug' => $slug,
        'parentId' => 0,
        'sort' => 500,
        'createdBy' => (int)$USER->GetID(),
        'createdAt' => date('c'),
    ];

    $pages[] = $page;
    sb_write_pages($pages);

    echo json_encode(['ok' => true, 'page' => $page], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'page.delete') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    $page = sb_find_page($id);
    if (!$page) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $pages = sb_read_pages();
    $pages = array_values(array_filter($pages, fn($p) => (int)($p['id'] ?? 0) !== $id));
    sb_write_pages($pages);

    $blocks = sb_read_blocks();
    $blocks = array_values(array_filter($blocks, fn($b) => (int)($b['pageId'] ?? 0) !== $id));
    sb_write_blocks($blocks);

    echo json_encode(['ok' => true], JSON_UNESCAPED_UNICODE);
    exit;
}

/** -------------------- BLOCKS -------------------- */

if ($action === 'block.list') {
    $pageId = (int)($_POST['pageId'] ?? 0);
    if ($pageId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'PAGE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    $page = sb_find_page($pageId);
    if (!$page) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }
    sb_require_viewer((int)($page['siteId'] ?? 0));

    $blocks = sb_read_blocks();
    $blocks = array_values(array_filter($blocks, fn($b) => (int)($b['pageId'] ?? 0) === $pageId));
    usort($blocks, fn($a,$b) => (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500));

    echo json_encode(['ok' => true, 'blocks' => $blocks], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'block.create') {
    $pageId = (int)($_POST['pageId'] ?? 0);
    $type   = trim((string)($_POST['type'] ?? 'text'));

    if ($pageId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'PAGE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if (!in_array($type, ['text','image'], true)) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'TYPE_NOT_SUPPORTED'], JSON_UNESCAPED_UNICODE); exit; }

    $page = sb_find_page($pageId);
    if (!$page) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }
    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $blocks = sb_read_blocks();
    $maxId = 0; foreach ($blocks as $b) $maxId = max($maxId, (int)($b['id'] ?? 0));
    $id = $maxId + 1;

    $maxSort = 0; foreach ($blocks as $b) if ((int)($b['pageId'] ?? 0) === $pageId) $maxSort = max($maxSort, (int)($b['sort'] ?? 0));
    $sort = $maxSort + 10;

    $content = [];
    if ($type === 'text') {
        $content = ['text' => (string)($_POST['text'] ?? '')];
    } else { // image
        $fileId = (int)($_POST['fileId'] ?? 0);
        $alt = (string)($_POST['alt'] ?? '');
        if ($fileId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'FILE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
        if (!sb_disk_file_belongs_to_site($siteId, $fileId)) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER'], JSON_UNESCAPED_UNICODE);
            exit;
        }
        $content = ['fileId' => $fileId, 'alt' => $alt];
    }

    $block = [
        'id' => $id,
        'pageId' => $pageId,
        'type' => $type,
        'sort' => $sort,
        'content' => $content,
        'createdBy' => (int)$USER->GetID(),
        'createdAt' => date('c'),
        'updatedAt' => date('c'),
    ];

    $blocks[] = $block;
    sb_write_blocks($blocks);

    echo json_encode(['ok' => true, 'block' => $block], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'block.update') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    $blk = sb_find_block($id);
    if (!$blk) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $page = sb_find_page((int)($blk['pageId'] ?? 0));
    if (!$page) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }
    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $blocks = sb_read_blocks();
    $found = false;

    foreach ($blocks as &$b) {
        if ((int)($b['id'] ?? 0) !== $id) continue;

        $type = (string)($b['type'] ?? '');
        if ($type === 'text') {
            $b['content']['text'] = (string)($_POST['text'] ?? '');
        } elseif ($type === 'image') {
            $fileId = (int)($_POST['fileId'] ?? 0);
            $alt = (string)($_POST['alt'] ?? '');
            if ($fileId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'FILE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
            if (!sb_disk_file_belongs_to_site($siteId, $fileId)) {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER'], JSON_UNESCAPED_UNICODE);
                exit;
            }
            $b['content']['fileId'] = $fileId;
            $b['content']['alt'] = $alt;
        } else {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'TYPE_NOT_SUPPORTED'], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $b['updatedAt'] = date('c');
        $found = true;
        break;
    }
    unset($b);

    if (!$found) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    sb_write_blocks($blocks);
    echo json_encode(['ok' => true], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'block.delete') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    $blk = sb_find_block($id);
    if (!$blk) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $page = sb_find_page((int)($blk['pageId'] ?? 0));
    if (!$page) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }
    sb_require_editor((int)($page['siteId'] ?? 0));

    $blocks = sb_read_blocks();
    $blocks = array_values(array_filter($blocks, fn($b) => (int)($b['id'] ?? 0) !== $id));
    sb_write_blocks($blocks);

    echo json_encode(['ok' => true], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'block.move') {
    $id  = (int)($_POST['id'] ?? 0);
    $dir = (string)($_POST['dir'] ?? '');

    if ($id <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($dir !== 'up' && $dir !== 'down') { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'DIR_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    $blk = sb_find_block($id);
    if (!$blk) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $page = sb_find_page((int)($blk['pageId'] ?? 0));
    if (!$page) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }
    sb_require_editor((int)($page['siteId'] ?? 0));

    $blocks = sb_read_blocks();
    $pageId = (int)($blk['pageId'] ?? 0);

    $pageBlocks = array_values(array_filter($blocks, fn($b) => (int)($b['pageId'] ?? 0) === $pageId));
    usort($pageBlocks, fn($a,$b) => (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500));

    $pos = null;
    for ($i=0; $i<count($pageBlocks); $i++) {
        if ((int)($pageBlocks[$i]['id'] ?? 0) === $id) { $pos = $i; break; }
    }
    if ($pos === null) { http_response_code(500); echo json_encode(['ok'=>false,'error'=>'INTERNAL'], JSON_UNESCAPED_UNICODE); exit; }

    if ($dir === 'up' && $pos === 0) { echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE); exit; }
    if ($dir === 'down' && $pos === count($pageBlocks)-1) { echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE); exit; }

    $swapWith = ($dir === 'up') ? $pos - 1 : $pos + 1;

    $idA = (int)$pageBlocks[$pos]['id'];
    $idB = (int)$pageBlocks[$swapWith]['id'];
    $sortA = (int)($pageBlocks[$pos]['sort'] ?? 500);
    $sortB = (int)($pageBlocks[$swapWith]['sort'] ?? 500);

    foreach ($blocks as &$b) {
        $bid = (int)($b['id'] ?? 0);
        if ($bid === $idA) { $b['sort'] = $sortB; $b['updatedAt'] = date('c'); }
        if ($bid === $idB) { $b['sort'] = $sortA; $b['updatedAt'] = date('c'); }
    }
    unset($b);

    sb_write_blocks($blocks);
    echo json_encode(['ok' => true], JSON_UNESCAPED_UNICODE);
    exit;
}

/** -------------------- ACCESS -------------------- */

if ($action === 'access.list') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    sb_require_owner($siteId);

    $acc = sb_read_access();
    $acc = array_values(array_filter($acc, fn($r) => (int)($r['siteId'] ?? 0) === $siteId));
    usort($acc, fn($a,$b) => sb_role_rank((string)($b['role'] ?? '')) <=> sb_role_rank((string)($a['role'] ?? '')));

    echo json_encode(['ok' => true, 'access' => $acc], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'access.set') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $userId = (int)($_POST['userId'] ?? 0);
    $role   = strtoupper(trim((string)($_POST['role'] ?? '')));

    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($userId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'USER_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if (!in_array($role, ['OWNER','ADMIN','EDITOR','VIEWER'], true)) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'ROLE_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    sb_require_owner($siteId);
    if (!sb_site_exists($siteId)) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'SITE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    if ($role === 'OWNER') {
        $acc = sb_read_access();
        foreach ($acc as $r) {
            if ((int)($r['siteId'] ?? 0) === $siteId && strtoupper((string)($r['role'] ?? '')) === 'OWNER') {
                if ((string)($r['accessCode'] ?? '') !== ('U'.$userId)) {
                    http_response_code(409);
                    echo json_encode(['ok'=>false,'error'=>'OWNER_ALREADY_EXISTS'], JSON_UNESCAPED_UNICODE);
                    exit;
                }
            }
        }
    }

    $acc = sb_read_access();
    $code = 'U' . $userId;
    $updated = false;

    foreach ($acc as &$r) {
        if ((int)($r['siteId'] ?? 0) === $siteId && (string)($r['accessCode'] ?? '') === $code) {
            $r['role'] = $role;
            $r['updatedBy'] = (int)$USER->GetID();
            $r['updatedAt'] = date('c');
            $updated = true;
            break;
        }
    }
    unset($r);

    if (!$updated) {
        $acc[] = [
            'siteId' => $siteId,
            'accessCode' => $code,
            'role' => $role,
            'createdBy' => (int)$USER->GetID(),
            'createdAt' => date('c'),
        ];
    }

    sb_write_access($acc);

    // best effort: ensure folder exists
    try { sb_disk_ensure_site_folder($siteId); } catch (\Throwable $e) {}

    echo json_encode(['ok' => true], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'access.delete') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $userId = (int)($_POST['userId'] ?? 0);

    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($userId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'USER_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    sb_require_owner($siteId);

    $code = 'U' . $userId;
    $myId = (int)$USER->GetID();

    $acc = sb_read_access();
    foreach ($acc as $r) {
        if ((int)($r['siteId'] ?? 0) === $siteId
            && (string)($r['accessCode'] ?? '') === ('U'.$myId)
            && strtoupper((string)($r['role'] ?? '')) === 'OWNER'
            && $userId === $myId
        ) {
            http_response_code(409);
            echo json_encode(['ok' => false, 'error' => 'CANNOT_REMOVE_SELF_OWNER'], JSON_UNESCAPED_UNICODE);
            exit;
        }
    }

    $before = count($acc);
    $acc = array_values(array_filter($acc, fn($r) => !((int)($r['siteId'] ?? 0) === $siteId && (string)($r['accessCode'] ?? '') === $code)));
    if (count($acc) === $before) {
        http_response_code(404);
        echo json_encode(['ok' => false, 'error' => 'NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    sb_write_access($acc);
    echo json_encode(['ok' => true], JSON_UNESCAPED_UNICODE);
    exit;
}

/** -------------------- FILES -------------------- */

if ($action === 'file.list') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    sb_require_viewer($siteId);

    try {
        $folder = sb_disk_ensure_site_folder($siteId);
        sb_disk_sync_folder_rights($siteId, $folder);

        $storage = $folder->getStorage();
        $children = sb_disk_get_children($folder, $storage);

        $files = [];
        $urlManager = Driver::getInstance()->getUrlManager();

        foreach ($children as $obj) {
            if ($obj instanceof File) {
                $downloadUrl = (method_exists($urlManager, 'getUrlForDownloadFile'))
                    ? (string)$urlManager->getUrlForDownloadFile($obj)
                    : '';

                $files[] = [
                    'id' => (int)$obj->getId(),
                    'name' => (string)$obj->getName(),
                    'size' => (int)$obj->getSize(),
                    'createdAt' => $obj->getCreateTime() ? $obj->getCreateTime()->toString() : '',
                    'downloadUrl' => $downloadUrl,
                ];
            }
        }

        $myRole = sb_get_role($siteId, sb_user_access_code());

        echo json_encode([
            'ok' => true,
            'myRole' => $myRole,
            'folder' => ['id' => (int)$folder->getId(), 'name' => (string)$folder->getName()],
            'files' => $files,
        ], JSON_UNESCAPED_UNICODE);
        exit;

    } catch (\Throwable $e) {
        http_response_code(500);
        echo json_encode(['ok' => false, 'error' => 'DISK_ERROR', 'message' => $e->getMessage()], JSON_UNESCAPED_UNICODE);
        exit;
    }
}

if ($action === 'file.upload') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    sb_require_editor($siteId);

    if (empty($_FILES['file']) || !is_array($_FILES['file'])) {
        http_response_code(422);
        echo json_encode(['ok' => false, 'error' => 'FILE_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    try {
        $folder = sb_disk_ensure_site_folder($siteId);
        sb_disk_sync_folder_rights($siteId, $folder);

        $storage = $folder->getStorage();
        $fileArray = $_FILES['file'];
        $name = (string)($fileArray['name'] ?? 'file');

        $uploaded = sb_disk_upload_file($folder, $fileArray, [
            'NAME' => $name,
            'CREATED_BY' => (int)$USER->GetID(),
        ], $storage);

        echo json_encode([
            'ok' => true,
            'file' => [
                'id' => (int)$uploaded->getId(),
                'name' => (string)$uploaded->getName(),
                'size' => (int)$uploaded->getSize(),
            ]
        ], JSON_UNESCAPED_UNICODE);
        exit;

    } catch (\Throwable $e) {
        http_response_code(500);
        echo json_encode(['ok' => false, 'error' => 'DISK_ERROR', 'message' => $e->getMessage()], JSON_UNESCAPED_UNICODE);
        exit;
    }
}

if ($action === 'file.delete') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $fileId = (int)($_POST['fileId'] ?? 0);

    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($fileId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'FILE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    sb_require_editor($siteId);

    try {
        $folder = sb_disk_ensure_site_folder($siteId);
        sb_disk_sync_folder_rights($siteId, $folder);

        $file = File::loadById($fileId);
        if (!$file) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'FILE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

        if ((int)$file->getParentId() !== (int)$folder->getId()) {
            http_response_code(403);
            echo json_encode(['ok' => false, 'error' => 'FOREIGN_FILE'], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $deleted = $file->delete((int)$USER->GetID());
        if (!$deleted) { http_response_code(500); echo json_encode(['ok'=>false,'error'=>'DELETE_FAILED'], JSON_UNESCAPED_UNICODE); exit; }

        echo json_encode(['ok' => true], JSON_UNESCAPED_UNICODE);
        exit;

    } catch (\Throwable $e) {
        http_response_code(500);
        echo json_encode(['ok' => false, 'error' => 'DISK_ERROR', 'message' => $e->getMessage()], JSON_UNESCAPED_UNICODE);
        exit;
    }
}

/** -------------------- MENU -------------------- */

// menu.list (VIEWER+)
if ($action === 'menu.list') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    sb_require_viewer($siteId);

    $all = sb_read_menus();
    $rec = sb_menu_get_site_record($all, $siteId);
    $menus = $rec ? ($rec['menus'] ?? []) : [];

    foreach ($menus as &$m) {
        $items = $m['items'] ?? [];
        usort($items, fn($a,$b) => (int)($a['sort'] ?? 0) <=> (int)($b['sort'] ?? 0));
        $m['items'] = $items;
    }
    unset($m);

    echo json_encode(['ok' => true, 'menus' => $menus], JSON_UNESCAPED_UNICODE);
    exit;
}

// menu.create (EDITOR+)
if ($action === 'menu.create') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $name = trim((string)($_POST['name'] ?? ''));
    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($name === '') { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'NAME_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    sb_require_editor($siteId);

    $all = sb_read_menus();
    $rec = sb_menu_get_site_record($all, $siteId);
    if (!$rec) $rec = ['siteId' => $siteId, 'menus' => []];

    $menuId = sb_menu_next_menu_id($rec);
    $menu = [
        'id' => $menuId,
        'name' => $name,
        'items' => [],
        'createdBy' => (int)$USER->GetID(),
        'createdAt' => date('c'),
        'updatedAt' => date('c'),
    ];

    $rec['menus'][] = $menu;
    sb_menu_upsert_site_record($all, $siteId, $rec);
    sb_write_menus($all);

    echo json_encode(['ok' => true, 'menu' => $menu], JSON_UNESCAPED_UNICODE);
    exit;
}

// menu.update (EDITOR+) rename
if ($action === 'menu.update') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $menuId = (int)($_POST['menuId'] ?? 0);
    $name = trim((string)($_POST['name'] ?? ''));
    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($menuId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'MENU_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($name === '') { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'NAME_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    sb_require_editor($siteId);

    $all = sb_read_menus();
    $rec = sb_menu_get_site_record($all, $siteId);
    if (!$rec) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $ok = sb_menu_update_menu($rec, $menuId, function($m) use ($name) {
        $m['name'] = $name;
        $m['updatedAt'] = date('c');
        return $m;
    });

    if (!$ok) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    sb_menu_upsert_site_record($all, $siteId, $rec);
    sb_write_menus($all);

    echo json_encode(['ok' => true], JSON_UNESCAPED_UNICODE);
    exit;
}

// menu.item.add (EDITOR+)
if ($action === 'menu.item.add') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $menuId = (int)($_POST['menuId'] ?? 0);
    $type = strtolower(trim((string)($_POST['type'] ?? 'page')));
    $title = trim((string)($_POST['title'] ?? ''));

    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($menuId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'MENU_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if (!in_array($type, ['page','url'], true)) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'TYPE_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    sb_require_editor($siteId);

    $all = sb_read_menus();
    $rec = sb_menu_get_site_record($all, $siteId);
    if (!$rec) $rec = ['siteId' => $siteId, 'menus' => []];

    $menu = sb_menu_find_menu($rec, $menuId);
    if (!$menu) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $items = $menu['items'] ?? [];
    $itemId = sb_menu_next_item_id($menu);
    $sort = sb_menu_next_sort($items);

    $item = [
        'id' => $itemId,
        'type' => $type,
        'title' => $title,
        'sort' => $sort,
    ];

    if ($type === 'page') {
        $pageId = (int)($_POST['pageId'] ?? 0);
        if ($pageId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'PAGE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
        $p = sb_find_page($pageId);
        if (!$p || (int)($p['siteId'] ?? 0) !== $siteId) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_IN_SITE'], JSON_UNESCAPED_UNICODE);
            exit;
        }
        $item['pageId'] = $pageId;
        if ($item['title'] === '') $item['title'] = (string)($p['title'] ?? ('page#'.$pageId));
    } else {
        $url = trim((string)($_POST['url'] ?? ''));
        if ($url === '') { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'URL_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
        if (!(preg_match('~^https?://~i', $url) || str_starts_with($url, '/'))) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'URL_BAD_FORMAT'], JSON_UNESCAPED_UNICODE);
            exit;
        }
        $item['url'] = $url;
        if ($item['title'] === '') $item['title'] = $url;
    }

    $items[] = $item;
    $menu['items'] = $items;
    $menu['updatedAt'] = date('c');

    sb_menu_update_menu($rec, $menuId, fn($_) => $menu);
    sb_menu_upsert_site_record($all, $siteId, $rec);
    sb_write_menus($all);

    echo json_encode(['ok' => true, 'item' => $item], JSON_UNESCAPED_UNICODE);
    exit;
}

// menu.item.update (EDITOR+)
if ($action === 'menu.item.update') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $menuId = (int)($_POST['menuId'] ?? 0);
    $itemId = (int)($_POST['itemId'] ?? 0);

    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($menuId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'MENU_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($itemId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'ITEM_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    sb_require_editor($siteId);

    $all = sb_read_menus();
    $rec = sb_menu_get_site_record($all, $siteId);
    if (!$rec) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $ok = sb_menu_update_menu($rec, $menuId, function($m) use ($siteId, $itemId) {
        $items = $m['items'] ?? [];
        $found = false;

        foreach ($items as &$it) {
            if ((int)($it['id'] ?? 0) !== $itemId) continue;

            $title = trim((string)($_POST['title'] ?? ''));
            if ($title !== '') $it['title'] = $title;

            if (($it['type'] ?? '') === 'page') {
                $pageId = (int)($_POST['pageId'] ?? 0);
                if ($pageId > 0) {
                    $p = sb_find_page($pageId);
                    if (!$p || (int)($p['siteId'] ?? 0) !== $siteId) {
                        http_response_code(422);
                        echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_IN_SITE'], JSON_UNESCAPED_UNICODE);
                        exit;
                    }
                    $it['pageId'] = $pageId;
                }
            } else {
                $url = trim((string)($_POST['url'] ?? ''));
                if ($url !== '') {
                    if (!(preg_match('~^https?://~i', $url) || str_starts_with($url, '/'))) {
                        http_response_code(422);
                        echo json_encode(['ok'=>false,'error'=>'URL_BAD_FORMAT'], JSON_UNESCAPED_UNICODE);
                        exit;
                    }
                    $it['url'] = $url;
                }
            }

            $found = true;
            break;
        }
        unset($it);

        if (!$found) return $m;

        $m['items'] = $items;
        $m['updatedAt'] = date('c');
        return $m;
    });

    if (!$ok) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    sb_menu_upsert_site_record($all, $siteId, $rec);
    sb_write_menus($all);

    echo json_encode(['ok' => true], JSON_UNESCAPED_UNICODE);
    exit;
}

// menu.item.delete (EDITOR+)
if ($action === 'menu.item.delete') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $menuId = (int)($_POST['menuId'] ?? 0);
    $itemId = (int)($_POST['itemId'] ?? 0);

    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($menuId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'MENU_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($itemId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'ITEM_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    sb_require_editor($siteId);

    $all = sb_read_menus();
    $rec = sb_menu_get_site_record($all, $siteId);
    if (!$rec) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $changed = sb_menu_update_menu($rec, $menuId, function($m) use ($itemId) {
        $items = $m['items'] ?? [];
        $before = count($items);
        $items = array_values(array_filter($items, fn($it) => (int)($it['id'] ?? 0) !== $itemId));
        if (count($items) === $before) return $m;
        $m['items'] = $items;
        $m['updatedAt'] = date('c');
        return $m;
    });

    if (!$changed) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    sb_menu_upsert_site_record($all, $siteId, $rec);
    sb_write_menus($all);

    echo json_encode(['ok' => true], JSON_UNESCAPED_UNICODE);
    exit;
}

// menu.item.move (EDITOR+)
if ($action === 'menu.item.move') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $menuId = (int)($_POST['menuId'] ?? 0);
    $itemId = (int)($_POST['itemId'] ?? 0);
    $dir = (string)($_POST['dir'] ?? '');

    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($menuId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'MENU_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($itemId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'ITEM_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($dir !== 'up' && $dir !== 'down') { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'DIR_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    sb_require_editor($siteId);

    $all = sb_read_menus();
    $rec = sb_menu_get_site_record($all, $siteId);
    if (!$rec) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $changed = sb_menu_update_menu($rec, $menuId, function($m) use ($itemId, $dir) {
        $items = $m['items'] ?? [];
        usort($items, fn($a,$b) => (int)($a['sort'] ?? 0) <=> (int)($b['sort'] ?? 0));

        $pos = null;
        for ($i=0; $i<count($items); $i++) {
            if ((int)($items[$i]['id'] ?? 0) === $itemId) { $pos = $i; break; }
        }
        if ($pos === null) return $m;

        if ($dir === 'up' && $pos === 0) return $m;
        if ($dir === 'down' && $pos === count($items)-1) return $m;

        $swap = ($dir === 'up') ? $pos-1 : $pos+1;

        $sortA = (int)($items[$pos]['sort'] ?? 0);
        $sortB = (int)($items[$swap]['sort'] ?? 0);

        $items[$pos]['sort'] = $sortB;
        $items[$swap]['sort'] = $sortA;

        $m['items'] = $items;
        $m['updatedAt'] = date('c');
        return $m;
    });

    if (!$changed) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    sb_menu_upsert_site_record($all, $siteId, $rec);
    sb_write_menus($all);

    echo json_encode(['ok' => true], JSON_UNESCAPED_UNICODE);
    exit;
}

http_response_code(400);
echo json_encode(['ok' => false, 'error' => 'UNKNOWN_ACTION', 'action' => $action], JSON_UNESCAPED_UNICODE);