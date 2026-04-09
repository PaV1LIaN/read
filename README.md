Отлично. Тогда добиваем page-ветку до рабочего состояния.

Ниже даю:

1. пару хелперов для страниц


2. полный актуальный api/handlers/page.php




---

1. Обнови /local/sitebuilder/lib/helpers.php

Добавь в конец файла эти функции:

<?php

if (!function_exists('sb_page_exists_in_site')) {
    function sb_page_exists_in_site(int $pageId, int $siteId): bool
    {
        $page = sb_find_page($pageId);
        return $page && (int)($page['siteId'] ?? 0) === $siteId;
    }
}

if (!function_exists('sb_page_children_ids')) {
    function sb_page_children_ids(int $siteId, int $parentId): array
    {
        $ids = [];
        foreach (sb_read_pages() as $p) {
            if (
                (int)($p['siteId'] ?? 0) === $siteId
                && (int)($p['parentId'] ?? 0) === $parentId
            ) {
                $ids[] = (int)($p['id'] ?? 0);
            }
        }
        return $ids;
    }
}

if (!function_exists('sb_page_is_descendant')) {
    function sb_page_is_descendant(int $siteId, int $candidateId, int $pageId): bool
    {
        if ($candidateId <= 0 || $pageId <= 0) {
            return false;
        }

        $pages = sb_read_pages();

        $childrenMap = [];
        foreach ($pages as $p) {
            if ((int)($p['siteId'] ?? 0) !== $siteId) {
                continue;
            }

            $pid = (int)($p['parentId'] ?? 0);
            $id  = (int)($p['id'] ?? 0);

            if (!isset($childrenMap[$pid])) {
                $childrenMap[$pid] = [];
            }
            $childrenMap[$pid][] = $id;
        }

        $stack = [$pageId];
        $seen = [];

        while ($stack) {
            $current = array_pop($stack);
            if (isset($seen[$current])) {
                continue;
            }
            $seen[$current] = true;

            foreach (($childrenMap[$current] ?? []) as $childId) {
                if ($childId === $candidateId) {
                    return true;
                }
                $stack[] = $childId;
            }
        }

        return false;
    }
}


---

2. Полностью замени /local/sitebuilder/api/handlers/page.php

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
        if (!isset($p['status']) || !in_array((string)$p['status'], ['draft', 'published'], true)) {
            $p['status'] = 'published';
        }
        if (!isset($p['publishedAt'])) {
            $p['publishedAt'] = '';
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
    ];

    $pages[] = $page;
    sb_write_pages($pages);

    sb_json_ok(['page' => $page]);
}

if ($action === 'page.delete') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }

    $page = sb_find_page($id);
    if (!$page) {
        sb_json_error('NOT_FOUND', 404);
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

    sb_json_ok();
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
    sb_json_ok();
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
    sb_json_ok();
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
        if (sb_page_is_descendant($siteId, $parentId, $id)) {
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
    sb_json_ok();
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
        sb_json_ok();
    }

    if ($dir === 'up' && $pos === 0) {
        sb_json_ok();
    }

    if ($dir === 'down' && $pos === count($siblings) - 1) {
        sb_json_ok();
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
    sb_json_ok();
}

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'page',
    'action' => $action,
]);


---

3. Что у тебя теперь готово по page

После вставки должны работать:

page.list

page.create

page.delete

page.duplicate

page.updateMeta

page.setStatus

page.setParent

page.move



---

4. Что проверить руками

Проверь по очереди:

1. создать страницу


2. сменить title/slug


3. сделать копию


4. подвигать вверх/вниз


5. поменять parent


6. сменить статус draft/published


7. удалить страницу




---

5. Что я чуть улучшил по сравнению со старым кодом

Тут есть два полезных улучшения:

запрет на установку родителя в собственного потомка: PARENT_DESCENDANT

при смене parent страница уходит в конец нового списка соседей через sort = max + 10


Это делает дерево стабильнее.


---

6. Следующий правильный шаг

Теперь логично идти в block-ветку, потому что без неё editor дальше не оживить.

Следующим сообщением я могу прислать уже первый рабочий block-пакет:

block.list

block.create

block.update

block.delete

block.duplicate

block.move

block.reorder


Напиши: давай block-пакет