Супер. Тогда следующий шаг — нормальный полный site.php.

Сейчас вернём:

site.list

site.get

site.create

site.delete

site.setHome


И сразу после этого расширим роутер под page.*.


---

1. Полностью замени /local/sitebuilder/api/handlers/site.php

<?php

global $USER;

if ($action === 'site.list') {
    $sites = sb_read_sites();
    $myCode = sb_user_access_code();

    $access = sb_read_access();
    $allowedSiteIds = [];

    foreach ($access as $r) {
        if ((string)($r['accessCode'] ?? '') === $myCode) {
            $sid = (int)($r['siteId'] ?? 0);
            if ($sid > 0) {
                $allowedSiteIds[$sid] = true;
            }
        }
    }

    $sites = array_values(array_filter($sites, static function ($s) use ($allowedSiteIds) {
        return isset($allowedSiteIds[(int)($s['id'] ?? 0)]);
    }));

    usort($sites, static function ($a, $b) {
        return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
    });

    sb_json_ok([
        'sites' => $sites,
        'handler' => 'site',
        'file' => __FILE__,
    ]);
}

if ($action === 'site.get') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    sb_require_viewer($siteId);

    $site = sb_find_site($siteId);
    if (!$site) {
        sb_json_error('SITE_NOT_FOUND', 404);
    }

    sb_json_ok([
        'site' => $site,
        'handler' => 'site',
        'file' => __FILE__,
    ]);
}

if ($action === 'site.create') {
    $name = trim((string)($_POST['name'] ?? ''));
    if ($name === '') {
        sb_json_error('NAME_REQUIRED', 422);
    }

    $sites = sb_read_sites();

    $maxId = 0;
    foreach ($sites as $s) {
        $maxId = max($maxId, (int)($s['id'] ?? 0));
    }
    $id = $maxId + 1;

    $slug = trim((string)($_POST['slug'] ?? ''));
    $slug = $slug === '' ? sb_slugify($name) : sb_slugify($slug);

    $existing = array_map(static function ($x) {
        return (string)($x['slug'] ?? '');
    }, $sites);

    $base = $slug !== '' ? $slug : 'site';
    $slug = $base;
    $i = 2;
    while (in_array($slug, $existing, true)) {
        $slug = $base . '-' . $i;
        $i++;
    }

    $site = [
        'id' => $id,
        'name' => $name,
        'slug' => $slug,
        'createdBy' => (int)$USER->GetID(),
        'createdAt' => date('c'),
        'updatedAt' => date('c'),
        'updatedBy' => (int)$USER->GetID(),
        'homePageId' => 0,
        'diskFolderId' => 0,
        'topMenuId' => 0,
        'settings' => [
            'containerWidth' => 1100,
            'accent' => '#2563eb',
            'logoFileId' => 0,
        ],
        'layout' => [
            'showHeader' => true,
            'showFooter' => true,
            'showLeft' => false,
            'showRight' => false,
            'leftWidth' => 260,
            'rightWidth' => 260,
            'leftMode' => 'blocks',
        ],
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
        'updatedAt' => date('c'),
        'updatedBy' => (int)$USER->GetID(),
    ];
    sb_write_access($access);

    sb_json_ok([
        'site' => $site,
        'handler' => 'site',
        'file' => __FILE__,
    ]);
}

if ($action === 'site.delete') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }

    sb_require_owner($id);

    $sites = sb_read_sites();
    $before = count($sites);

    $sites = array_values(array_filter($sites, static function ($s) use ($id) {
        return (int)($s['id'] ?? 0) !== $id;
    }));

    if (count($sites) === $before) {
        sb_json_error('NOT_FOUND', 404);
    }

    sb_write_sites($sites);

    $pages = sb_read_pages();
    $deletedPageIds = [];
    foreach ($pages as $p) {
        if ((int)($p['siteId'] ?? 0) === $id) {
            $deletedPageIds[(int)($p['id'] ?? 0)] = true;
        }
    }

    $pages = array_values(array_filter($pages, static function ($p) use ($id) {
        return (int)($p['siteId'] ?? 0) !== $id;
    }));
    sb_write_pages($pages);

    $blocks = sb_read_blocks();
    $blocks = array_values(array_filter($blocks, static function ($b) use ($deletedPageIds) {
        return !isset($deletedPageIds[(int)($b['pageId'] ?? 0)]);
    }));
    sb_write_blocks($blocks);

    $access = sb_read_access();
    $access = array_values(array_filter($access, static function ($r) use ($id) {
        return (int)($r['siteId'] ?? 0) !== $id;
    }));
    sb_write_access($access);

    $menus = sb_read_menus();
    $menus = array_values(array_filter($menus, static function ($m) use ($id) {
        return (int)($m['siteId'] ?? 0) !== $id;
    }));
    sb_write_menus($menus);

    $templates = sb_read_templates();
    $templates = array_values(array_filter($templates, static function ($tpl) use ($id) {
        return (int)($tpl['siteId'] ?? 0) !== $id;
    }));
    sb_write_templates($templates);

    $layouts = sb_read_layouts();
    $layouts = array_values(array_filter($layouts, static function ($layout) use ($id) {
        return (int)($layout['siteId'] ?? 0) !== $id;
    }));
    sb_write_layouts($layouts);

    sb_json_ok([
        'handler' => 'site',
        'file' => __FILE__,
    ]);
}

if ($action === 'site.setHome') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $pageId = (int)($_POST['pageId'] ?? 0);

    if ($siteId <= 0 || $pageId <= 0) {
        sb_json_error('SITE_PAGE_REQUIRED', 422);
    }

    sb_require_editor($siteId);

    $page = sb_find_page($pageId);
    if (!$page || (int)($page['siteId'] ?? 0) !== $siteId) {
        sb_json_error('PAGE_NOT_IN_SITE', 422);
    }

    $sites = sb_read_sites();
    $found = false;

    foreach ($sites as $i => $s) {
        if ((int)($s['id'] ?? 0) === $siteId) {
            $sites[$i]['homePageId'] = $pageId;
            $sites[$i]['updatedAt'] = date('c');
            $sites[$i]['updatedBy'] = (int)$USER->GetID();
            $found = true;
            break;
        }
    }

    if (!$found) {
        sb_json_error('SITE_NOT_FOUND', 404);
    }

    sb_write_sites($sites);

    sb_json_ok([
        'handler' => 'site',
        'file' => __FILE__,
    ]);
}

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'site',
    'action' => $action,
    'file' => __FILE__,
]);


---

2. Расширь /local/sitebuilder/api/index.php

Сейчас добавим page.*, но пока без остальных веток, чтобы не вносить лишнюю сложность.

<?php

require_once __DIR__ . '/bootstrap.php';

$action = (string)($_POST['action'] ?? '');

if ($action === 'ping') {
    require __DIR__ . '/handlers/common.php';
    exit;
}

if (
    $action === 'site.list' ||
    $action === 'site.get' ||
    $action === 'site.create' ||
    $action === 'site.delete' ||
    $action === 'site.setHome'
) {
    require __DIR__ . '/handlers/site.php';
    exit;
}

if (
    $action === 'page.list' ||
    $action === 'page.create' ||
    $action === 'page.delete' ||
    $action === 'page.duplicate' ||
    $action === 'page.updateMeta' ||
    $action === 'page.setStatus' ||
    $action === 'page.setParent' ||
    $action === 'page.move'
) {
    require __DIR__ . '/handlers/page.php';
    exit;
}

sb_json_error('UNKNOWN_ACTION', 400, [
    'action' => $action,
    'file' => __FILE__,
]);


---

3. Проверь, что /local/sitebuilder/api/handlers/page.php начинается с <?php

Это важно.
После прошлой ошибки обязательно проверь, что самая первая строка файла:

<?php

Если хочешь, просто замени файл целиком на этот рабочий вариант.

/local/sitebuilder/api/handlers/page.php

<?php

global $USER;

if ($action === 'page.list') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    sb_require_viewer($siteId);

    $pages = sb_read_pages();
    $pages = array_values(array_filter($pages, static function ($p) use ($siteId) {
        return (int)($p['siteId'] ?? 0) === $siteId;
    }));

    foreach ($pages as &$p) {
        if (!isset($p['status']) || !in_array((string)($p['status'] ?? ''), ['draft', 'published'], true)) {
            $p['status'] = 'published';
        }
        if (!isset($p['publishedAt'])) {
            $p['publishedAt'] = '';
        }
        if (!isset($p['parentId'])) {
            $p['parentId'] = 0;
        }
        if (!isset($p['sort'])) {
            $p['sort'] = 500;
        }
    }
    unset($p);

    usort($pages, static function ($a, $b) {
        $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);
        if ($sortCmp !== 0) {
            return $sortCmp;
        }
        return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
    });

    $homePageId = 0;
    $site = sb_find_site($siteId);
    if ($site) {
        $homePageId = (int)($site['homePageId'] ?? 0);
    }

    sb_json_ok([
        'pages' => $pages,
        'homePageId' => $homePageId,
        'handler' => 'page',
        'file' => __FILE__,
    ]);
}

if ($action === 'page.create') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $title  = trim((string)($_POST['title'] ?? ''));
    $slugIn = trim((string)($_POST['slug'] ?? ''));

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }
    if ($title === '') {
        sb_json_error('TITLE_REQUIRED', 422);
    }

    sb_require_editor($siteId);

    $slug = $slugIn !== '' ? sb_slugify($slugIn) : sb_slugify($title);

    $pages = sb_read_pages();

    $maxId = 0;
    $maxSort = 0;
    foreach ($pages as $p) {
        $maxId = max($maxId, (int)($p['id'] ?? 0));
        if ((int)($p['siteId'] ?? 0) === $siteId && (int)($p['parentId'] ?? 0) === 0) {
            $maxSort = max($maxSort, (int)($p['sort'] ?? 0));
        }
    }
    $id = $maxId + 1;

    $existing = array_map(
        static function ($x) {
            return (string)($x['slug'] ?? '');
        },
        array_filter($pages, static function ($p) use ($siteId) {
            return (int)($p['siteId'] ?? 0) === $siteId;
        })
    );

    $base = $slug !== '' ? $slug : 'page';
    $slug = $base;
    $i = 2;
    while (in_array($slug, $existing, true)) {
        $slug = $base . '-' . $i;
        $i++;
    }

    $page = [
        'id' => $id,
        'siteId' => $siteId,
        'title' => $title,
        'slug' => $slug,
        'parentId' => 0,
        'sort' => $maxSort + 10,
        'status' => 'draft',
        'publishedAt' => '',
        'createdBy' => (int)$USER->GetID(),
        'createdAt' => date('c'),
        'updatedAt' => date('c'),
        'updatedBy' => (int)$USER->GetID(),
    ];

    $pages[] = $page;
    sb_write_pages($pages);

    sb_json_ok([
        'page' => $page,
        'handler' => 'page',
        'file' => __FILE__,
    ]);
}

if ($action === 'page.delete') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }

    $page = sb_find_page($id);
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $pages = sb_read_pages();
    $pages = array_values(array_filter($pages, static function ($p) use ($id) {
        return (int)($p['id'] ?? 0) !== $id;
    }));
    sb_write_pages($pages);

    $blocks = sb_read_blocks();
    $blocks = array_values(array_filter($blocks, static function ($b) use ($id) {
        return (int)($b['pageId'] ?? 0) !== $id;
    }));
    sb_write_blocks($blocks);

    $sites = sb_read_sites();
    $sitesChanged = false;
    foreach ($sites as &$s) {
        if ((int)($s['id'] ?? 0) === $siteId && (int)($s['homePageId'] ?? 0) === $id) {
            $s['homePageId'] = 0;
            $s['updatedAt'] = date('c');
            $s['updatedBy'] = (int)$USER->GetID();
            $sitesChanged = true;
        }
    }
    unset($s);

    if ($sitesChanged) {
        sb_write_sites($sites);
    }

    sb_json_ok([
        'handler' => 'page',
        'file' => __FILE__,
    ]);
}

if ($action === 'page.duplicate') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }

    $srcPage = sb_find_page($id);
    if (!$srcPage) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($srcPage['siteId'] ?? 0);
    sb_require_editor($siteId);

    $pages = sb_read_pages();
    $blocks = sb_read_blocks();

    $maxPageId = 0;
    foreach ($pages as $p) {
        $maxPageId = max($maxPageId, (int)($p['id'] ?? 0));
    }
    $newPageId = $maxPageId + 1;

    $srcTitle = (string)($srcPage['title'] ?? 'Страница');
    $newTitle = $srcTitle . ' (копия)';

    $baseSlug = sb_slugify((string)($srcPage['slug'] ?? ($srcPage['title'] ?? 'page')));
    if ($baseSlug === '') {
        $baseSlug = 'page';
    }

    $newSlug = $baseSlug . '-copy';

    $existing = array_map(
        static function ($x) {
            return (string)($x['slug'] ?? '');
        },
        array_filter($pages, static function ($p) use ($siteId) {
            return (int)($p['siteId'] ?? 0) === $siteId;
        })
    );

    $base = $newSlug;
    $i = 2;
    while (in_array($newSlug, $existing, true)) {
        $newSlug = $base . '-' . $i;
        $i++;
    }

    $srcSort = (int)($srcPage['sort'] ?? 500);
    $srcParentId = (int)($srcPage['parentId'] ?? 0);

    foreach ($pages as &$p) {
        if (
            (int)($p['siteId'] ?? 0) === $siteId &&
            (int)($p['parentId'] ?? 0) === $srcParentId &&
            (int)($p['sort'] ?? 0) > $srcSort
        ) {
            $p['sort'] = (int)($p['sort'] ?? 0) + 10;
            $p['updatedAt'] = date('c');
            $p['updatedBy'] = (int)$USER->GetID();
        }
    }
    unset($p);

    $newPage = $srcPage;
    $newPage['id'] = $newPageId;
    $newPage['title'] = $newTitle;
    $newPage['slug'] = $newSlug;
    $newPage['sort'] = $srcSort + 10;
    $newPage['createdBy'] = (int)$USER->GetID();
    $newPage['createdAt'] = date('c');
    $newPage['updatedAt'] = date('c');
    $newPage['updatedBy'] = (int)$USER->GetID();
    $newPage['status'] = 'draft';
    $newPage['publishedAt'] = '';

    $pages[] = $newPage;

    $maxBlockId = 0;
    foreach ($blocks as $b) {
        $maxBlockId = max($maxBlockId, (int)($b['id'] ?? 0));
    }
    $nextBlockId = $maxBlockId + 1;

    $srcBlocks = array_values(array_filter($blocks, static function ($b) use ($id) {
        return (int)($b['pageId'] ?? 0) === $id;
    }));

    usort($srcBlocks, static function ($a, $b) {
        return (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);
    });

    foreach ($srcBlocks as $b) {
        $copy = $b;
        $copy['id'] = $nextBlockId++;
        $copy['pageId'] = $newPageId;
        $copy['createdBy'] = (int)$USER->GetID();
        $copy['createdAt'] = date('c');
        $copy['updatedAt'] = date('c');
        $copy['updatedBy'] = (int)$USER->GetID();
        $blocks[] = $copy;
    }

    sb_write_pages($pages);
    sb_write_blocks($blocks);

    sb_json_ok([
        'page' => $newPage,
        'handler' => 'page',
        'file' => __FILE__,
    ]);
}

if ($action === 'page.updateMeta') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }

    $page = sb_find_page($id);
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $title = trim((string)($_POST['title'] ?? ''));
    $slugIn = trim((string)($_POST['slug'] ?? ''));

    $pages = sb_read_pages();
    $found = false;

    $newSlug = $slugIn !== ''
        ? sb_slugify($slugIn)
        : (string)($page['slug'] ?? ('page-' . $id));

    if ($title !== '' && $slugIn === '') {
        $newSlug = sb_slugify($title);
    }

    if ($newSlug === '') {
        $newSlug = 'page-' . $id;
    }

    $existing = array_map(
        static function ($x) {
            return (string)($x['slug'] ?? '');
        },
        array_filter($pages, static function ($p) use ($siteId, $id) {
            return (int)($p['siteId'] ?? 0) === $siteId && (int)($p['id'] ?? 0) !== $id;
        })
    );

    $base = $newSlug;
    $i = 2;
    while (in_array($newSlug, $existing, true)) {
        $newSlug = $base . '-' . $i;
        $i++;
    }

    foreach ($pages as &$p) {
        if ((int)($p['id'] ?? 0) === $id) {
            if ($title !== '') {
                $p['title'] = $title;
            }
            $p['slug'] = $newSlug;
            $p['updatedAt'] = date('c');
            $p['updatedBy'] = (int)$USER->GetID();
            $found = true;
            break;
        }
    }
    unset($p);

    if (!$found) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    sb_write_pages($pages);

    sb_json_ok([
        'handler' => 'page',
        'file' => __FILE__,
    ]);
}

if ($action === 'page.setStatus') {
    $id = (int)($_POST['id'] ?? 0);
    $status = strtolower(trim((string)($_POST['status'] ?? 'draft')));

    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }

    if (!in_array($status, ['draft', 'published'], true)) {
        sb_json_error('BAD_STATUS', 422);
    }

    $page = sb_find_page($id);
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $pages = sb_read_pages();
    $found = false;

    foreach ($pages as &$p) {
        if ((int)($p['id'] ?? 0) === $id) {
            $p['status'] = $status;
            $p['publishedAt'] = ($status === 'published') ? date('c') : '';
            $p['updatedAt'] = date('c');
            $p['updatedBy'] = (int)$USER->GetID();
            $found = true;
            break;
        }
    }
    unset($p);

    if (!$found) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    sb_write_pages($pages);

    sb_json_ok([
        'handler' => 'page',
        'file' => __FILE__,
    ]);
}

if ($action === 'page.setParent') {
    $id = (int)($_POST['id'] ?? 0);
    $parentId = (int)($_POST['parentId'] ?? 0);

    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }

    $page = sb_find_page($id);
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    if ($parentId > 0) {
        $parent = sb_find_page($parentId);
        if (!$parent || (int)($parent['siteId'] ?? 0) !== $siteId) {
            sb_json_error('PARENT_NOT_IN_SITE', 422);
        }
        if ($parentId === $id) {
            sb_json_error('PARENT_SELF', 422);
        }
        if (function_exists('sb_page_is_descendant') && sb_page_is_descendant($siteId, $parentId, $id)) {
            sb_json_error('PARENT_DESCENDANT', 422);
        }
    }

    $pages = sb_read_pages();

    $maxSort = 0;
    foreach ($pages as $p) {
        if (
            (int)($p['siteId'] ?? 0) === $siteId &&
            (int)($p['parentId'] ?? 0) === $parentId
        ) {
            $maxSort = max($maxSort, (int)($p['sort'] ?? 0));
        }
    }

    foreach ($pages as &$p) {
        if ((int)($p['id'] ?? 0) === $id) {
            $p['parentId'] = $parentId;
            $p['sort'] = $maxSort + 10;
            $p['updatedAt'] = date('c');
            $p['updatedBy'] = (int)$USER->GetID();
            break;
        }
    }
    unset($p);

    sb_write_pages($pages);

    sb_json_ok([
        'handler' => 'page',
        'file' => __FILE__,
    ]);
}

if ($action === 'page.move') {
    $id = (int)($_POST['id'] ?? 0);
    $dir = (string)($_POST['dir'] ?? '');

    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }
    if ($dir !== 'up' && $dir !== 'down') {
        sb_json_error('DIR_REQUIRED', 422);
    }

    $page = sb_find_page($id);
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $parentId = (int)($page['parentId'] ?? 0);

    $pages = sb_read_pages();
    $siblings = array_values(array_filter($pages, static function ($p) use ($siteId, $parentId) {
        return
            (int)($p['siteId'] ?? 0) === $siteId &&
            (int)($p['parentId'] ?? 0) === $parentId;
    }));

    usort($siblings, static function ($a, $b) {
        $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);
        if ($sortCmp !== 0) {
            return $sortCmp;
        }
        return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
    });

    $pos = null;
    for ($i = 0; $i < count($siblings); $i++) {
        if ((int)($siblings[$i]['id'] ?? 0) === $id) {
            $pos = $i;
            break;
        }
    }

    if ($pos === null) {
        sb_json_ok([
            'handler' => 'page',
            'file' => __FILE__,
        ]);
    }

    if ($dir === 'up' && $pos === 0) {
        sb_json_ok([
            'handler' => 'page',
            'file' => __FILE__,
        ]);
    }

    if ($dir === 'down' && $pos === count($siblings) - 1) {
        sb_json_ok([
            'handler' => 'page',
            'file' => __FILE__,
        ]);
    }

    $swap = ($dir === 'up') ? $pos - 1 : $pos + 1;

    $idA = (int)$siblings[$pos]['id'];
    $idB = (int)$siblings[$swap]['id'];
    $sortA = (int)($siblings[$pos]['sort'] ?? 500);
    $sortB = (int)($siblings[$swap]['sort'] ?? 500);

    foreach ($pages as &$p) {
        $pid = (int)($p['id'] ?? 0);
        if ($pid === $idA) {
            $p['sort'] = $sortB;
            $p['updatedAt'] = date('c');
            $p['updatedBy'] = (int)$USER->GetID();
        }
        if ($pid === $idB) {
            $p['sort'] = $sortA;
            $p['updatedAt'] = date('c');
            $p['updatedBy'] = (int)$USER->GetID();
        }
    }
    unset($p);

    sb_write_pages($pages);

    sb_json_ok([
        'handler' => 'page',
        'file' => __FILE__,
    ]);
}

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'page',
    'action' => $action,
    'file' => __FILE__,
]);


---

4. Сбрось opcache

Открой один раз:

/local/sitebuilder/opcache_reset.php


---

5. Что проверить сейчас

В test_api.php можно быстро проверить:

Ping

Site list


А в index.php:

загрузку списка сайтов

создание сайта

удаление сайта


Если хочешь быстро проверить page без отдельной кнопки, можно временно сделать POST руками в браузерной консоли через BX.ajax, но лучше я следующим сообщением сразу дам тебе готовый pages.php или editor.php-каркас, чтобы не тестировать в слепую.


---

Куда идём дальше

Сейчас логичный порядок такой:

1. убедиться, что site.* снова полные и живые


2. подключить page.*


3. потом вернуть в роутер menu.*


4. потом block.*



Чтобы не развалить снова всё разом, я предлагаю после этой вставки сделать следующий шаг: подключить в роутер menu.* и дать рабочую menu.php уже на новом API.

Напиши, что:

index.php снова работает

или какая именно ошибка сейчас осталась


И я дам следующий точный пакет.