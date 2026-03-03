<?php
define('NO_KEEP_STATISTIC', true);
define('NO_AGENT_STATISTIC', true);
define('DisableEventsCheck', true);

require $_SERVER['DOCUMENT_ROOT'].'/bitrix/modules/main/include/prolog_before.php';

global $USER;

if (!$USER->IsAuthorized()) {
    LocalRedirect('/auth/');
}

header('Content-Type: text/html; charset=UTF-8');

$siteId = (int)($_GET['siteId'] ?? 0);
$pageId = (int)($_GET['pageId'] ?? 0);

function sb_data_path(string $file): string {
    return $_SERVER['DOCUMENT_ROOT'] . '/upload/sitebuilder/' . $file;
}
function sb_read_json(string $file): array {
    $path = sb_data_path($file);
    if (!file_exists($path)) return [];
    $raw = file_get_contents($path);
    $data = json_decode((string)$raw, true);
    return is_array($data) ? $data : [];
}
function h($s): string { return htmlspecialcharsbx((string)$s); }

function downloadUrl(int $siteId, int $fileId): string {
    return '/local/sitebuilder/download.php?siteId=' . $siteId . '&fileId=' . $fileId;
}
function viewUrl(int $siteId, int $pageId): string {
    return '/local/sitebuilder/view.php?siteId=' . $siteId . '&pageId=' . $pageId;
}

// load data
$sites = sb_read_json('sites.json');
$pages = sb_read_json('pages.json');
$blocksAll = sb_read_json('blocks.json');
$menusAll = sb_read_json('menus.json');

<<<<<<< HEAD
function sb_read_pages(): array { return sb_read_json_file('pages.json'); }
function sb_write_pages(array $pages): void { sb_write_json_file('pages.json', $pages, 'Cannot open pages.json'); }

function sb_read_blocks(): array { return sb_read_json_file('blocks.json'); }
function sb_write_blocks(array $blocks): void { sb_write_json_file('blocks.json', $blocks, 'Cannot open blocks.json'); }

function sb_read_access(): array { return sb_read_json_file('access.json'); }
function sb_write_access(array $access): void { sb_write_json_file('access.json', $access, 'Cannot open access.json'); }

function sb_read_menus(): array { return sb_read_json_file('menus.json'); }
function sb_write_menus(array $menus): void { sb_write_json_file('menus.json', $menus, 'Cannot open menus.json'); }

function sb_read_templates(): array { return sb_read_json_file('templates.json'); }
function sb_write_templates(array $templates): void { sb_write_json_file('templates.json', $templates, 'Cannot open templates.json'); }

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
=======
// find site/page
$site = null;
foreach ($sites as $s) {
    if ((int)($s['id'] ?? 0) === $siteId) { $site = $s; break; }
>>>>>>> f9609428e7c429199222889e01729600331f4974
}

$page = null;
foreach ($pages as $p) {
    if ((int)($p['id'] ?? 0) === $pageId && (int)($p['siteId'] ?? 0) === $siteId) { $page = $p; break; }
}

if (!$site || !$page) {
    http_response_code(404);
    ?>
    <!doctype html><html lang="ru"><head><meta charset="utf-8"><title>Не найдено</title></head>
    <body style="font-family:Arial;padding:24px;background:#f6f7f8;">
      <div style="background:#fff;border:1px solid #e5e7ea;border-radius:12px;padding:16px;">
        <h2 style="margin-top:0;">Страница не найдена</h2>
        <div>siteId=<?= (int)$siteId ?>, pageId=<?= (int)$pageId ?></div>
        <div style="margin-top:12px;"><a href="/local/sitebuilder/index.php">← Назад</a></div>
      </div>
    </body></html>
    <?php
    exit;
}

// site pages (for fallback nav / editor link)
$sitePages = array_values(array_filter($pages, fn($p) => (int)($p['siteId'] ?? 0) === $siteId));
usort($sitePages, fn($a, $b) => (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500));

// blocks for current page
$blocks = array_values(array_filter($blocksAll, fn($b) => (int)($b['pageId'] ?? 0) === $pageId));
usort($blocks, fn($a, $b) => (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500));

// menu: take first menu for this site (if exists)
$menuItems = [];
foreach ($menusAll as $rec) {
    if ((int)($rec['siteId'] ?? 0) === $siteId) {
        $menus = $rec['menus'] ?? [];
        if (is_array($menus) && count($menus) > 0) {
            $first = $menus[0];
            $menuItems = $first['items'] ?? [];
            if (!is_array($menuItems)) $menuItems = [];
            usort($menuItems, fn($a, $b) => (int)($a['sort'] ?? 0) <=> (int)($b['sort'] ?? 0));
        }
<<<<<<< HEAD
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

/** -------------------- TEMPLATE -------------------- */
if ($action === 'template.list') {
    $tpl = sb_read_templates();
    usort($tpl, fn($a,$b) => strcmp((string)($b['createdAt'] ?? ''), (string)($a['createdAt'] ?? '')));
    echo json_encode(['ok'=>true,'templates'=>$tpl], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'template.createFromPage') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $pageId = (int)($_POST['pageId'] ?? 0);
    $name = trim((string)($_POST['name'] ?? ''));

    if ($siteId<=0 || $pageId<=0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_PAGE_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($name==='') { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'NAME_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    sb_require_editor($siteId);

    $page = sb_find_page($pageId);
    if (!$page || (int)($page['siteId'] ?? 0) !== $siteId) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $blocks = sb_read_blocks();
    $pageBlocks = array_values(array_filter($blocks, fn($b)=> (int)($b['pageId'] ?? 0) === $pageId));
    usort($pageBlocks, fn($a,$b)=> (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500));

    $snapshot = [];
    foreach ($pageBlocks as $b) {
        $snapshot[] = [
            'type' => (string)($b['type'] ?? 'text'),
            'sort' => (int)($b['sort'] ?? 500),
            'content' => is_array($b['content'] ?? null) ? $b['content'] : [],
        ];
    }

    $tpl = sb_read_templates();
    $maxId = 0; foreach ($tpl as $t) $maxId = max($maxId, (int)($t['id'] ?? 0));
    $id = $maxId + 1;

    $record = [
        'id' => $id,
        'name' => $name,
        'createdAt' => date('c'),
        'createdBy' => (int)$USER->GetID(),
        'blocks' => $snapshot,
    ];

    $tpl[] = $record;
    sb_write_templates($tpl);

    echo json_encode(['ok'=>true,'template'=>$record], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'template.applyToPage') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $pageId = (int)($_POST['pageId'] ?? 0);
    $tplId  = (int)($_POST['templateId'] ?? 0);
    $mode   = (string)($_POST['mode'] ?? 'append');

    if ($siteId<=0 || $pageId<=0 || $tplId<=0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'PARAMS_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($mode !== 'append' && $mode !== 'replace') $mode = 'append';

    sb_require_editor($siteId);

    $page = sb_find_page($pageId);
    if (!$page || (int)($page['siteId'] ?? 0) !== $siteId) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $tplAll = sb_read_templates();
    $tpl = null;
    foreach ($tplAll as $t) if ((int)($t['id'] ?? 0) === $tplId) { $tpl = $t; break; }
    if (!$tpl) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'TEMPLATE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $blocks = sb_read_blocks();

    // очистка старых блоков
    if ($mode === 'replace') {
        $blocks = array_values(array_filter($blocks, fn($b)=> (int)($b['pageId'] ?? 0) !== $pageId));
    }

    // вычислим новые id и sort
    $maxId = 0; foreach ($blocks as $b) $maxId = max($maxId, (int)($b['id'] ?? 0));
    $nextId = $maxId + 1;

    $maxSort = 0;
    foreach ($blocks as $b) if ((int)($b['pageId'] ?? 0) === $pageId) $maxSort = max($maxSort, (int)($b['sort'] ?? 0));
    $baseSort = ($mode === 'append') ? ($maxSort + 10) : 10;

    $tplBlocks = is_array($tpl['blocks'] ?? null) ? $tpl['blocks'] : [];
    $i = 0;

    foreach ($tplBlocks as $tb) {
        if (!is_array($tb)) continue;
        $type = (string)($tb['type'] ?? 'text');
        $content = is_array($tb['content'] ?? null) ? $tb['content'] : [];

        $blocks[] = [
            'id' => $nextId++,
            'pageId' => $pageId,
            'type' => $type,
            'sort' => $baseSort + ($i * 10),
            'content' => $content,
            'createdBy' => (int)$USER->GetID(),
            'createdAt' => date('c'),
            'updatedAt' => date('c'),
        ];
        $i++;
    }

    sb_write_blocks($blocks);
    echo json_encode(['ok'=>true,'added'=>$i], JSON_UNESCAPED_UNICODE);
    exit;
}

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

// Домашняя страница
if ($action === 'site.setHome') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $pageId = (int)($_POST['pageId'] ?? 0);

    if ($siteId <= 0 || $pageId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_PAGE_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    // только editor+ (или owner, решишь сам)
    sb_require_editor($siteId);

    $page = sb_find_page($pageId);
    if (!$page || (int)($page['siteId'] ?? 0) !== $siteId) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_IN_SITE'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $sites = sb_read_sites();
    $found = false;

    foreach ($sites as &$s) {
        if ((int)($s['id'] ?? 0) === $siteId) {
            $s['homePageId'] = $pageId;
            $s['updatedAt'] = date('c');
            $s['updatedBy'] = (int)$USER->GetID();
            $found = true;
            break;
        }
    }
    unset($s);

    if (!$found) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'SITE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    sb_write_sites($sites);
    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
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

    if ($pageId <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'PAGE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    // ДОБАВИЛИ button
    if (!in_array($type, ['text','image','button', 'heading', 'columns2', 'gallery', 'spacer', 'card', 'cards'], true)) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'TYPE_NOT_SUPPORTED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $page = sb_find_page($pageId);
    if (!$page) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }
    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $blocks = sb_read_blocks();
    $maxId = 0;
    foreach ($blocks as $b) $maxId = max($maxId, (int)($b['id'] ?? 0));
    $id = $maxId + 1;

    $maxSort = 0;
    foreach ($blocks as $b) {
        if ((int)($b['pageId'] ?? 0) === $pageId) {
            $maxSort = max($maxSort, (int)($b['sort'] ?? 0));
        }
    }
    $sort = $maxSort + 10;

    $content = [];

    if ($type === 'text') {
        $content = ['text' => (string)($_POST['text'] ?? '')];

    } elseif ($type === 'image') {
        $fileId = (int)($_POST['fileId'] ?? 0);
        $alt = (string)($_POST['alt'] ?? '');

        if ($fileId <= 0) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'FILE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
            exit;
        }
        if (!sb_disk_file_belongs_to_site($siteId, $fileId)) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER'], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $content = ['fileId' => $fileId, 'alt' => $alt];

    } elseif ($type === 'heading') {
        $text = trim((string)($_POST['text'] ?? ''));
        $level = strtolower(trim((string)($_POST['level'] ?? 'h2')));
        $align = strtolower(trim((string)($_POST['align'] ?? 'left')));

        if ($text === '') {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'TEXT_REQUIRED'], JSON_UNESCAPED_UNICODE);
            exit;
        }
        if (!in_array($level, ['h1','h2','h3'], true)) $level = 'h2';
        if (!in_array($align, ['left','center','right'], true)) $align = 'left';

        $content = ['text' => $text, 'level' => $level, 'align' => $align];



    } elseif ($type === 'columns2') {
        $left  = (string)($_POST['left'] ?? '');
        $right = (string)($_POST['right'] ?? '');
        $ratio = (string)($_POST['ratio'] ?? '50-50');

        $ratio = trim($ratio);
        if (!in_array($ratio, ['50-50','33-67','67-33'], true)) $ratio = '50-50';

        $content = [
            'left' => $left,
            'right' => $right,
            'ratio' => $ratio,
        ];



    } elseif ($type === 'gallery') {
        $columns = (int)($_POST['columns'] ?? 3);
        if (!in_array($columns, [2,3,4], true)) $columns = 3;

        $imagesJson = (string)($_POST['images'] ?? '[]');
        $images = json_decode($imagesJson, true);
        if (!is_array($images)) $images = [];

        // нормализуем + проверяем, что файлы лежат в папке сайта
        $clean = [];
        foreach ($images as $it) {
            if (!is_array($it)) continue;
            $fid = (int)($it['fileId'] ?? 0);
            if ($fid <= 0) continue;

            if (!sb_disk_file_belongs_to_site($siteId, $fid)) {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER','fileId'=>$fid], JSON_UNESCAPED_UNICODE);
                exit;
            }

            $clean[] = [
                'fileId' => $fid,
                'alt' => (string)($it['alt'] ?? ''),
            ];
        }

        if (count($clean) === 0) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'IMAGES_REQUIRED'], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $content = [
            'columns' => $columns,
            'images' => $clean,
        ];

    } elseif ($type === 'spacer') {
        $height = (int)($_POST['height'] ?? 40);
        if ($height < 10) $height = 10;
        if ($height > 200) $height = 200;

        $line = (string)($_POST['line'] ?? '0');
        $line = ($line === '1' || $line === 'true');

        $content = ['height' => $height, 'line' => $line];



    } elseif ($type === 'card') {
        $title = trim((string)($_POST['title'] ?? ''));
        $text  = (string)($_POST['text'] ?? '');

        $imageFileId = (int)($_POST['imageFileId'] ?? 0);

        $buttonText = trim((string)($_POST['buttonText'] ?? ''));
        $buttonUrl  = trim((string)($_POST['buttonUrl'] ?? ''));

        if ($title === '') {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'TITLE_REQUIRED'], JSON_UNESCAPED_UNICODE);
            exit;
        }

        // картинка опционально, но если задана — должна быть в папке сайта
        if ($imageFileId > 0 && !sb_disk_file_belongs_to_site($siteId, $imageFileId)) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER','fileId'=>$imageFileId], JSON_UNESCAPED_UNICODE);
            exit;
        }

        // кнопка опционально, но если задан URL — проверим формат
        if ($buttonUrl !== '' && !(preg_match('~^https?://~i', $buttonUrl) || str_starts_with($buttonUrl, '/'))) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'URL_BAD_FORMAT'], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $content = [
            'title' => $title,
            'text' => $text,
            'imageFileId' => $imageFileId,
            'buttonText' => $buttonText,
            'buttonUrl' => $buttonUrl,
        ];

    } elseif ($type === 'cards') {
        $columns = (int)($_POST['columns'] ?? 3);
        if (!in_array($columns, [2,3,4], true)) $columns = 3;

        $itemsJson = (string)($_POST['items'] ?? '[]');
        $items = json_decode($itemsJson, true);
        if (!is_array($items)) $items = [];

        $clean = [];
        foreach ($items as $it) {
            if (!is_array($it)) continue;

            $title = trim((string)($it['title'] ?? ''));
            $text  = (string)($it['text'] ?? '');

            if ($title === '') continue; // минимум — заголовок

            $imageFileId = (int)($it['imageFileId'] ?? 0);
            if ($imageFileId > 0 && !sb_disk_file_belongs_to_site($siteId, $imageFileId)) {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER','fileId'=>$imageFileId], JSON_UNESCAPED_UNICODE);
                exit;
            }

            $buttonText = trim((string)($it['buttonText'] ?? ''));
            $buttonUrl  = trim((string)($it['buttonUrl'] ?? ''));

            if ($buttonUrl !== '' && !(preg_match('~^https?://~i', $buttonUrl) || str_starts_with($buttonUrl, '/'))) {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'URL_BAD_FORMAT'], JSON_UNESCAPED_UNICODE);
                exit;
            }

            $clean[] = [
                'title' => $title,
                'text' => $text,
                'imageFileId' => $imageFileId,
                'buttonText' => $buttonText,
                'buttonUrl' => $buttonUrl,
            ];
        }

        if (count($clean) === 0) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'ITEMS_REQUIRED'], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $content = [
            'columns' => $columns,
            'items' => $clean,
        ];

    } else { // button
        $text = trim((string)($_POST['text'] ?? ''));
        $url  = trim((string)($_POST['url'] ?? ''));
        $variant = strtolower(trim((string)($_POST['variant'] ?? 'primary')));

        if ($text === '') {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'TEXT_REQUIRED'], JSON_UNESCAPED_UNICODE);
            exit;
        }
        if ($url === '') {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'URL_REQUIRED'], JSON_UNESCAPED_UNICODE);
            exit;
        }
        if (!in_array($variant, ['primary','secondary'], true)) $variant = 'primary';

        // внешний http(s) или относительный /
        if (!(preg_match('~^https?://~i', $url) || str_starts_with($url, '/'))) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'URL_BAD_FORMAT'], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $content = ['text' => $text, 'url' => $url, 'variant' => $variant];
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
    if ($id <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $blk = sb_find_block($id);
    if (!$blk) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $page = sb_find_page((int)($blk['pageId'] ?? 0));
    if (!$page) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

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

            if ($fileId <= 0) {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'FILE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
                exit;
            }
            if (!sb_disk_file_belongs_to_site($siteId, $fileId)) {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER'], JSON_UNESCAPED_UNICODE);
                exit;
            }

            $b['content']['fileId'] = $fileId;
            $b['content']['alt'] = $alt;

        } elseif ($type === 'button') {
            $text = trim((string)($_POST['text'] ?? ''));
            $url  = trim((string)($_POST['url'] ?? ''));
            $variant = strtolower(trim((string)($_POST['variant'] ?? 'primary')));

            if ($text === '') {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'TEXT_REQUIRED'], JSON_UNESCAPED_UNICODE);
                exit;
            }
            if ($url === '') {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'URL_REQUIRED'], JSON_UNESCAPED_UNICODE);
                exit;
            }
            if (!in_array($variant, ['primary','secondary'], true)) $variant = 'primary';

            if (!(preg_match('~^https?://~i', $url) || str_starts_with($url, '/'))) {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'URL_BAD_FORMAT'], JSON_UNESCAPED_UNICODE);
                exit;
            }

            $b['content']['text'] = $text;
            $b['content']['url'] = $url;
            $b['content']['variant'] = $variant;



        } elseif ($type === 'heading') {
            $text = trim((string)($_POST['text'] ?? ''));
            $level = strtolower(trim((string)($_POST['level'] ?? 'h2')));
            $align = strtolower(trim((string)($_POST['align'] ?? 'left')));

            if ($text === '') {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'TEXT_REQUIRED'], JSON_UNESCAPED_UNICODE);
                exit;
            }
            if (!in_array($level, ['h1','h2','h3'], true)) $level = 'h2';
            if (!in_array($align, ['left','center','right'], true)) $align = 'left';

            $b['content']['text'] = $text;
            $b['content']['level'] = $level;
            $b['content']['align'] = $align;



        } elseif ($type === 'columns2') {
            $left  = (string)($_POST['left'] ?? '');
            $right = (string)($_POST['right'] ?? '');
            $ratio = (string)($_POST['ratio'] ?? '50-50');

            $ratio = trim($ratio);
            if (!in_array($ratio, ['50-50','33-67','67-33'], true)) $ratio = '50-50';

            $b['content']['left'] = $left;
            $b['content']['right'] = $right;
            $b['content']['ratio'] = $ratio;


        } elseif ($type === 'gallery') {
            $columns = (int)($_POST['columns'] ?? 3);
            if (!in_array($columns, [2,3,4], true)) $columns = 3;

            $imagesJson = (string)($_POST['images'] ?? '[]');
            $images = json_decode($imagesJson, true);
            if (!is_array($images)) $images = [];

            $clean = [];
            foreach ($images as $it) {
                if (!is_array($it)) continue;
                $fid = (int)($it['fileId'] ?? 0);
                if ($fid <= 0) continue;

                if (!sb_disk_file_belongs_to_site($siteId, $fid)) {
                    http_response_code(422);
                    echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER','fileId'=>$fid], JSON_UNESCAPED_UNICODE);
                    exit;
                }

                $clean[] = [
                    'fileId' => $fid,
                    'alt' => (string)($it['alt'] ?? ''),
                ];
            }

            if (count($clean) === 0) {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'IMAGES_REQUIRED'], JSON_UNESCAPED_UNICODE);
                exit;
            }

            $b['content']['columns'] = $columns;
            $b['content']['images'] = $clean;

        } elseif ($type === 'spacer') {
            $height = (int)($_POST['height'] ?? 40);
            if ($height < 10) $height = 10;
            if ($height > 200) $height = 200;

            $line = (string)($_POST['line'] ?? '0');
            $line = ($line === '1' || $line === 'true');

            $b['content']['height'] = $height;
            $b['content']['line'] = $line;

        } elseif ($type === 'card') {
            $title = trim((string)($_POST['title'] ?? ''));
            $text  = (string)($_POST['text'] ?? '');

            $imageFileId = (int)($_POST['imageFileId'] ?? 0);

            $buttonText = trim((string)($_POST['buttonText'] ?? ''));
            $buttonUrl  = trim((string)($_POST['buttonUrl'] ?? ''));

            if ($title === '') {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'TITLE_REQUIRED'], JSON_UNESCAPED_UNICODE);
                exit;
            }

            if ($imageFileId > 0 && !sb_disk_file_belongs_to_site($siteId, $imageFileId)) {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER','fileId'=>$imageFileId], JSON_UNESCAPED_UNICODE);
                exit;
            }

            if ($buttonUrl !== '' && !(preg_match('~^https?://~i', $buttonUrl) || str_starts_with($buttonUrl, '/'))) {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'URL_BAD_FORMAT'], JSON_UNESCAPED_UNICODE);
                exit;
            }

            $b['content']['title'] = $title;
            $b['content']['text'] = $text;
            $b['content']['imageFileId'] = $imageFileId;
            $b['content']['buttonText'] = $buttonText;
            $b['content']['buttonUrl'] = $buttonUrl;

        } elseif ($type === 'cards') {
            $columns = (int)($_POST['columns'] ?? 3);
            if (!in_array($columns, [2,3,4], true)) $columns = 3;

            $itemsJson = (string)($_POST['items'] ?? '[]');
            $items = json_decode($itemsJson, true);
            if (!is_array($items)) $items = [];

            $clean = [];
            foreach ($items as $it) {
                if (!is_array($it)) continue;

                $title = trim((string)($it['title'] ?? ''));
                $text  = (string)($it['text'] ?? '');

                if ($title === '') continue;

                $imageFileId = (int)($it['imageFileId'] ?? 0);
                if ($imageFileId > 0 && !sb_disk_file_belongs_to_site($siteId, $imageFileId)) {
                    http_response_code(422);
                    echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER','fileId'=>$imageFileId], JSON_UNESCAPED_UNICODE);
                    exit;
                }

                $buttonText = trim((string)($it['buttonText'] ?? ''));
                $buttonUrl  = trim((string)($it['buttonUrl'] ?? ''));

                if ($buttonUrl !== '' && !(preg_match('~^https?://~i', $buttonUrl) || str_starts_with($buttonUrl, '/'))) {
                    http_response_code(422);
                    echo json_encode(['ok'=>false,'error'=>'URL_BAD_FORMAT'], JSON_UNESCAPED_UNICODE);
                    exit;
                }

                $clean[] = [
                    'title' => $title,
                    'text' => $text,
                    'imageFileId' => $imageFileId,
                    'buttonText' => $buttonText,
                    'buttonUrl' => $buttonUrl,
                ];
            }

            if (count($clean) === 0) {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'ITEMS_REQUIRED'], JSON_UNESCAPED_UNICODE);
                exit;
            }

            $b['content']['columns'] = $columns;
            $b['content']['items'] = $clean;



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

    if (!$found) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

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
=======
        break;
    }
}
?>
<!doctype html>
<html lang="ru">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title><?=h($page['title'])?> — <?=h($site['name'])?></title>
  <style>
    body { font-family: Arial, sans-serif; margin: 0; background:#f6f7f8; color:#111; }
    .top {
      background:#fff;
      border-bottom:1px solid #e5e7ea;
      padding:12px 16px;
      display:flex;
      gap:10px;
      align-items:center;
      flex-wrap:wrap;
>>>>>>> f9609428e7c429199222889e01729600331f4974
    }
    .content { padding: 18px; }
    .card { background:#fff; border:1px solid #e5e7ea; border-radius:12px; padding:16px; }
    a { color:#0b57d0; text-decoration:none; }
    a:hover { text-decoration:underline; }
    .muted { color:#6a737f; }
    .menu { display:flex; gap:10px; flex-wrap:wrap; }
    .menu a { padding:6px 10px; border-radius:999px; }
    .menu a.active { background:#eef2ff; font-weight:bold; }
    .block-text { margin-top:12px; line-height:1.6; font-size:16px; }
    .block-img { margin-top:14px; }
    .block-img img { max-width:100%; height:auto; border-radius:12px; border:1px solid #eee; display:block; }
    code { background:#f3f4f6; padding:2px 6px; border-radius:6px; }
    .btn { display:inline-block; padding:10px 14px; border-radius:12px; border:1px solid #e5e7ea; text-decoration:none; }
    .btn-primary { background:#2563eb; color:#fff; border-color:#2563eb; }
    .btn-secondary { background:#fff; color:#111; }

        /* columns2 */
    .cols2 {
    margin-top: 14px;
    display: grid;
    gap: 14px;
    }
    .cols2 .col {
    background: #fff;
    border: 1px solid #eee;
    border-radius: 12px;
    padding: 12px;
    }
    @media (max-width: 768px) {
    .cols2 { grid-template-columns: 1fr !important; }
    }

    .gallery {
        margin-top: 14px;
        display: grid;
        gap: 12px;
    }
    .gallery img {
        width: 100%;
        height: auto;
        display: block;
        border-radius: 12px;
        border: 1px solid #eee;
        background: #fafafa;
    }
    @media (max-width: 768px) {
        .gallery { grid-template-columns: 1fr !important; }
    }

    .spacer { width:100%; }
    .spacerLine { height:1px; background:#e5e7ea; }

    .cardBlock {
        margin-top:14px;
        background:#fff;
        border:1px solid #eee;
        border-radius:16px;
        padding:14px;
    }
    .cardBlock img {
        width:100%;
        height:auto;
        display:block;
        border-radius:14px;
        border:1px solid #eee;
        margin-top:10px;
    }
    .cardTitle { font-weight:700; font-size:18px; }
    .cardText { margin-top:8px; color:#333; line-height:1.6; white-space:pre-wrap; }
    .cardBtn { display:inline-block; margin-top:10px; padding:10px 14px; border-radius:12px; border:1px solid #e5e7ea; text-decoration:none; }

    .cardsGrid {
        margin-top:14px;
        display:grid;
        gap:14px;
    }
    .cardsGrid .cardItem {
        background:#fff;
        border:1px solid #eee;
        border-radius:16px;
        padding:14px;
    }
    .cardsGrid .cardItem img{
        width:100%;
        height:auto;
        display:block;
        border-radius:14px;
        border:1px solid #eee;
        background:#fafafa;
        margin-top:10px;
    }
    .cardsGrid .t { font-weight:700; font-size:16px; }
    .cardsGrid .d { margin-top:8px; color:#333; line-height:1.6; white-space:pre-wrap; }
    .cardsGrid .a { display:inline-block; margin-top:10px; padding:10px 14px; border-radius:12px; border:1px solid #e5e7ea; text-decoration:none; }
    @media (max-width: 768px) {
        .cardsGrid { grid-template-columns: 1fr !important; }
    }
  </style>
</head>
<body>
  <div class="top">
    <a href="/local/sitebuilder/index.php">← Назад</a>
    <div class="muted">/</div>
    <div><b><?=h($site['name'])?></b></div>
    <div class="muted">/</div>
    <div><?=h($page['title'])?></div>

    <div style="flex:1;"></div>

    <a href="/local/sitebuilder/editor.php?siteId=<?= (int)$siteId ?>&pageId=<?= (int)$pageId ?>" target="_blank">Редактор</a>
    <a href="/local/sitebuilder/menu.php?siteId=<?= (int)$siteId ?>" target="_blank">Меню</a>

    <div style="flex-basis:100%; height:0;"></div>

    <?php if ($menuItems): ?>
      <div class="menu">
        <?php foreach ($menuItems as $it): ?>
          <?php
            $type = (string)($it['type'] ?? '');
            $title = (string)($it['title'] ?? '');
            $isActive = false;
            $href = '#';

            if ($type === 'page') {
              $pid = (int)($it['pageId'] ?? 0);
              $href = viewUrl($siteId, $pid);
              $isActive = ($pid === $pageId);
              if ($title === '') $title = 'page#'.$pid;
            } elseif ($type === 'url') {
              $u = (string)($it['url'] ?? '');
              $href = $u !== '' ? $u : '#';
              if ($title === '') $title = $href;
            } else {
              $href = '#';
              if ($title === '') $title = '(unknown)';
            }
          ?>
          <a class="<?= $isActive ? 'active' : '' ?>" href="<?= h($href) ?>" <?= ($type === 'url' && preg_match('~^https?://~i', $href)) ? 'target="_blank" rel="noopener noreferrer"' : '' ?>>
            <?= h($title) ?>
          </a>
        <?php endforeach; ?>
      </div>
    <?php else: ?>
      <!-- fallback: если меню ещё не создано -->
      <div class="menu">
        <?php foreach ($sitePages as $sp): ?>
          <?php $isActive = ((int)($sp['id'] ?? 0) === (int)$pageId); ?>
          <a class="<?= $isActive ? 'active' : '' ?>" href="<?= h(viewUrl($siteId, (int)($sp['id'] ?? 0))) ?>">
            <?= h($sp['title'] ?? '') ?>
          </a>
        <?php endforeach; ?>
      </div>
    <?php endif; ?>
  </div>

  <div class="content">
    <div class="card">
      <h1 style="margin-top:0;"><?=h($page['title'])?></h1>

      <?php if (!$blocks): ?>
        <div class="muted">Блоков пока нет. Добавь их в редакторе.</div>
      <?php else: ?>
        <?php foreach ($blocks as $b): ?>
          <?php $type = (string)($b['type'] ?? ''); ?>

          <?php if ($type === 'text'): ?>
            <div class="block-text">
              <?= nl2br(h((string)($b['content']['text'] ?? ''))) ?>
            </div>
          <?php endif; ?>

          <?php if ($type === 'button'): ?>
			  <?php
				$text = (string)($b['content']['text'] ?? '');
				$url = (string)($b['content']['url'] ?? '#');
				$variant = (string)($b['content']['variant'] ?? 'primary');
				$cls = ($variant === 'secondary') ? 'btn btn-secondary' : 'btn btn-primary';
			  ?>
			  <div class="block-btn" style="margin-top:14px;">
				<a class="<?=h($cls)?>" href="<?=h($url)?>" <?= preg_match('~^https?://~i', $url) ? 'target="_blank" rel="noopener noreferrer"' : '' ?>>
				  <?=h($text)?>
				</a>
			  </div>
			<?php endif; ?>

          <?php if ($type === 'image'): ?>
            <?php $fileId = (int)($b['content']['fileId'] ?? 0); ?>
            <?php $alt = (string)($b['content']['alt'] ?? ''); ?>
            <?php if ($fileId > 0): ?>
              <div class="block-img">
                <img src="<?= h(downloadUrl($siteId, $fileId)) ?>" alt="<?= h($alt) ?>">
              </div>
            <?php else: ?>
              <div class="muted" style="margin-top:12px;">(image) файл не выбран</div>
            <?php endif; ?>
          <?php endif; ?>

          <?php if ($type === 'heading'): ?>

          <?php
            $text = (string)($b['content']['text'] ?? '');
            $level = (string)($b['content']['level'] ?? 'h2');
            $align = (string)($b['content']['align'] ?? 'left');
            if (!in_array($level, ['h1','h2','h3'], true)) $level = 'h2';
            if (!in_array($align, ['left','center','right'], true)) $align = 'left';
          ?>
            <<?=h($level)?> style="margin-top:16px; text-align:<?=h($align)?>;">
                <?=h($text)?>
            </<?=h($level)?>>
            <?php endif; ?>


          <?php if ($type === 'columns2'): ?>
            <?php
                $left  = (string)($b['content']['left'] ?? '');
                $right = (string)($b['content']['right'] ?? '');
                $ratio = (string)($b['content']['ratio'] ?? '50-50');

                if (!in_array($ratio, ['50-50','33-67','67-33'], true)) $ratio = '50-50';

                $tpl = '1fr 1fr';
                if ($ratio === '33-67') $tpl = '1fr 2fr';
                if ($ratio === '67-33') $tpl = '2fr 1fr';
            ?>
            <div class="cols2" style="grid-template-columns: <?=h($tpl)?>;">
                <div class="col"><?= nl2br(h($left)) ?></div>
                <div class="col"><?= nl2br(h($right)) ?></div>
            </div>
            <?php endif; ?>

            <?php if ($type === 'gallery'): ?>
                <?php
                    $cols = (int)($b['content']['columns'] ?? 3);
                    if (!in_array($cols, [2,3,4], true)) $cols = 3;

                    $tpl = '1fr 1fr 1fr';
                    if ($cols === 2) $tpl = '1fr 1fr';
                    if ($cols === 4) $tpl = '1fr 1fr 1fr 1fr';

                    $imgs = $b['content']['images'] ?? [];
                    if (!is_array($imgs)) $imgs = [];
                ?>
                <div class="gallery" style="grid-template-columns: <?=h($tpl)?>;">
                    <?php foreach ($imgs as $it): ?>
                    <?php
                        if (!is_array($it)) continue;
                        $fid = (int)($it['fileId'] ?? 0);
                        $alt = (string)($it['alt'] ?? '');
                        if ($fid <= 0) continue;
                    ?>
                    <img src="<?= h(downloadUrl($siteId, $fid)) ?>" alt="<?= h($alt) ?>">
                    <?php endforeach; ?>
                </div>
                <?php endif; ?>


            <?php if ($type === 'spacer'): ?>
                <?php
                    $h = (int)($b['content']['height'] ?? 40);
                    if ($h < 10) $h = 10;
                    if ($h > 200) $h = 200;
                    $line = (bool)($b['content']['line'] ?? false);
                ?>
                <div class="spacer" style="height: <?= (int)$h ?>px; position:relative; margin-top:14px;">
                    <?php if ($line): ?>
                    <div class="spacerLine" style="position:absolute; left:0; right:0; top:50%;"></div>
                    <?php endif; ?>
                </div>
            <?php endif; ?>

            <?php if ($type === 'card'): ?>
                <?php
                    $title = (string)($b['content']['title'] ?? '');
                    $text  = (string)($b['content']['text'] ?? '');
                    $imgId = (int)($b['content']['imageFileId'] ?? 0);
                    $btnText = trim((string)($b['content']['buttonText'] ?? ''));
                    $btnUrl  = trim((string)($b['content']['buttonUrl'] ?? ''));
                ?>
                <div class="cardBlock">
                    <div class="cardTitle"><?=h($title)?></div>
                    <?php if ($text !== ''): ?><div class="cardText"><?=nl2br(h($text))?></div><?php endif; ?>

                    <?php if ($imgId > 0): ?>
                    <img src="<?=h(downloadUrl($siteId, $imgId))?>" alt="">
                    <?php endif; ?>

                    <?php if ($btnUrl !== ''): ?>
                    <a class="cardBtn" href="<?=h($btnUrl)?>" <?= preg_match('~^https?://~i', $btnUrl) ? 'target="_blank" rel="noopener noreferrer"' : '' ?>>
                        <?=h($btnText !== '' ? $btnText : 'Открыть')?>
                    </a>
                    <?php endif; ?>
                </div>
            <?php endif; ?>

            <?php if ($type === 'cards'): ?>
                <?php
                    $cols = (int)($b['content']['columns'] ?? 3);
                    if (!in_array($cols, [2,3,4], true)) $cols = 3;

                    $tpl = '1fr 1fr 1fr';
                    if ($cols === 2) $tpl = '1fr 1fr';
                    if ($cols === 4) $tpl = '1fr 1fr 1fr 1fr';

                    $items = $b['content']['items'] ?? [];
                    if (!is_array($items)) $items = [];
                ?>
                <div class="cardsGrid" style="grid-template-columns: <?=h($tpl)?>;">
                    <?php foreach ($items as $it): ?>
                    <?php
                        if (!is_array($it)) continue;
                        $title = (string)($it['title'] ?? '');
                        if ($title === '') continue;
                        $text = (string)($it['text'] ?? '');
                        $imgId = (int)($it['imageFileId'] ?? 0);
                        $btnText = trim((string)($it['buttonText'] ?? ''));
                        $btnUrl  = trim((string)($it['buttonUrl'] ?? ''));
                    ?>
                    <div class="cardItem">
                        <div class="t"><?=h($title)?></div>
                        <?php if ($text !== ''): ?><div class="d"><?=nl2br(h($text))?></div><?php endif; ?>
                        <?php if ($imgId > 0): ?><img src="<?=h(downloadUrl($siteId, $imgId))?>" alt=""><?php endif; ?>
                        <?php if ($btnUrl !== ''): ?>
                        <a class="a" href="<?=h($btnUrl)?>" <?= preg_match('~^https?://~i', $btnUrl) ? 'target="_blank" rel="noopener noreferrer"' : '' ?>>
                            <?=h($btnText !== '' ? $btnText : 'Открыть')?>
                        </a>
                        <?php endif; ?>
                    </div>
                    <?php endforeach; ?>
                </div>
            <?php endif; ?>

        <?php endforeach; ?>
      <?php endif; ?>

<<<<<<< HEAD
    if (!$changed) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    sb_menu_upsert_site_record($all, $siteId, $rec);
    sb_write_menus($all);

    echo json_encode(['ok' => true], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'template.rename') {
    $id = (int)($_POST['id'] ?? 0);
    $name = trim((string)($_POST['name'] ?? ''));

    if ($id <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($name === '') { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'NAME_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    $tpl = sb_read_templates();
    $found = false;

    foreach ($tpl as &$t) {
        if ((int)($t['id'] ?? 0) === $id) {
            $t['name'] = $name;
            $t['updatedAt'] = date('c');
            $t['updatedBy'] = (int)$USER->GetID();
            $found = true;
            break;
        }
    }
    unset($t);

    if (!$found) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'TEMPLATE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    sb_write_templates($tpl);
    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'template.delete') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    $tpl = sb_read_templates();
    $before = count($tpl);
    $tpl = array_values(array_filter($tpl, fn($t) => (int)($t['id'] ?? 0) !== $id));

    if (count($tpl) === $before) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'TEMPLATE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    sb_write_templates($tpl);
    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'page.updateMeta') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    $page = sb_find_page($id);
    if (!$page) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $title = trim((string)($_POST['title'] ?? ''));
    $slugIn = trim((string)($_POST['slug'] ?? ''));

    $pages = sb_read_pages();
    $found = false;

    // slug (если дали)
    $newSlug = $slugIn !== '' ? sb_slugify($slugIn) : (string)($page['slug'] ?? 'page-'.$id);
    if ($title !== '' && $slugIn === '') {
        // если slug не задан, но title изменили — можем пересчитать slug от title (аккуратно)
        $newSlug = sb_slugify($title);
    }

    // уникальность slug в рамках siteId
    $existing = array_map(fn($x) => (string)($x['slug'] ?? ''),
        array_filter($pages, fn($p) => (int)($p['siteId'] ?? 0) === $siteId && (int)($p['id'] ?? 0) !== $id)
    );
    $base = $newSlug; $i = 2;
    while (in_array($newSlug, $existing, true)) { $newSlug = $base.'-'.$i; $i++; }

    foreach ($pages as &$p) {
        if ((int)($p['id'] ?? 0) === $id) {
            if ($title !== '') $p['title'] = $title;
            $p['slug'] = $newSlug;
            $p['updatedAt'] = date('c');
            $p['updatedBy'] = (int)$USER->GetID();
            $found = true;
            break;
        }
    }
    unset($p);

    if (!$found) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    sb_write_pages($pages);
    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'page.setParent') {
    $id = (int)($_POST['id'] ?? 0);
    $parentId = (int)($_POST['parentId'] ?? 0); // 0 = корневая

    if ($id <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    $page = sb_find_page($id);
    if (!$page) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    if ($parentId > 0) {
        $parent = sb_find_page($parentId);
        if (!$parent || (int)($parent['siteId'] ?? 0) !== $siteId) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'PARENT_NOT_IN_SITE'], JSON_UNESCAPED_UNICODE);
            exit;
        }
        if ($parentId === $id) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'PARENT_SELF'], JSON_UNESCAPED_UNICODE);
            exit;
        }
    }

    $pages = sb_read_pages();
    foreach ($pages as &$p) {
        if ((int)($p['id'] ?? 0) === $id) {
            $p['parentId'] = $parentId;
            $p['updatedAt'] = date('c');
            $p['updatedBy'] = (int)$USER->GetID();
            break;
        }
    }
    unset($p);

    sb_write_pages($pages);
    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'page.move') {
    $id = (int)($_POST['id'] ?? 0);
    $dir = (string)($_POST['dir'] ?? '');

    if ($id <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($dir !== 'up' && $dir !== 'down') { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'DIR_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    $page = sb_find_page($id);
    if (!$page) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $parentId = (int)($page['parentId'] ?? 0);

    $pages = sb_read_pages();
    $siblings = array_values(array_filter($pages, fn($p) =>
        (int)($p['siteId'] ?? 0) === $siteId && (int)($p['parentId'] ?? 0) === $parentId
    ));
    usort($siblings, fn($a,$b) => (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500));

    $pos = null;
    for ($i=0; $i<count($siblings); $i++) {
        if ((int)($siblings[$i]['id'] ?? 0) === $id) { $pos = $i; break; }
    }
    if ($pos === null) { echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE); exit; }

    if ($dir === 'up' && $pos === 0) { echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE); exit; }
    if ($dir === 'down' && $pos === count($siblings)-1) { echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE); exit; }

    $swap = ($dir === 'up') ? $pos-1 : $pos+1;

    $idA = (int)$siblings[$pos]['id'];
    $idB = (int)$siblings[$swap]['id'];
    $sortA = (int)($siblings[$pos]['sort'] ?? 500);
    $sortB = (int)($siblings[$swap]['sort'] ?? 500);

    foreach ($pages as &$p) {
        $pid = (int)($p['id'] ?? 0);
        if ($pid === $idA) { $p['sort'] = $sortB; $p['updatedAt']=date('c'); $p['updatedBy']=(int)$USER->GetID(); }
        if ($pid === $idB) { $p['sort'] = $sortA; $p['updatedAt']=date('c'); $p['updatedBy']=(int)$USER->GetID(); }
    }
    unset($p);

    sb_write_pages($pages);
    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
    exit;
}

http_response_code(400);
echo json_encode(['ok' => false, 'error' => 'UNKNOWN_ACTION', 'action' => $action], JSON_UNESCAPED_UNICODE);
=======
      <div style="margin-top:16px;" class="muted">
        <div><b>slug:</b> <code><?=h($page['slug'])?></code></div>
        <div><b>pageId:</b> <?= (int)$page['id'] ?> &nbsp; <b>siteId:</b> <?= (int)$site['id'] ?></div>
      </div>
    </div>
  </div>
</body>
</html>
>>>>>>> f9609428e7c429199222889e01729600331f4974
