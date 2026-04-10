Давай. Реально внедрим дерево страниц на parentId без отдельной сущности section.

Сделаем так:

1. backend: нормализуем page-handler, чтобы можно было задавать parentId


2. frontend: в editor.php

добавим выбор родителя при создании

добавим смену родителя у существующей страницы

отрисуем дерево страниц вместо плоского списка




Ниже даю пакетом, что заменить.


---

1. Замени api/handlers/page.php

Полностью замени файл:

/local/sitebuilder/api/handlers/page.php

на этот код:

<?php

if ($action === 'page.list') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    sb_require_viewer($siteId);

    $pages = array_values(array_filter(sb_read_pages(), static function ($p) use ($siteId) {
        return (int)($p['siteId'] ?? 0) === $siteId;
    }));

    usort($pages, static function ($a, $b) {
        $parentCmp = (int)($a['parentId'] ?? 0) <=> (int)($b['parentId'] ?? 0);
        if ($parentCmp !== 0) {
            return $parentCmp;
        }

        $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);
        if ($sortCmp !== 0) {
            return $sortCmp;
        }

        return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
    });

    sb_json_ok([
        'pages' => array_values(array_map('sb_normalize_page_record', $pages)),
    ]);
}

if ($action === 'page.create') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $title = trim((string)($_POST['title'] ?? ''));
    $slug = trim((string)($_POST['slug'] ?? ''));
    $parentId = (int)($_POST['parentId'] ?? 0);

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    if ($title === '') {
        sb_json_error('TITLE_REQUIRED', 422);
    }

    sb_require_editor($siteId);

    if ($slug === '') {
        $slug = sb_slugify($title);
    }

    $pages = sb_read_pages();

    if ($parentId > 0) {
        $parent = null;
        foreach ($pages as $p) {
            if ((int)($p['id'] ?? 0) === $parentId && (int)($p['siteId'] ?? 0) === $siteId) {
                $parent = $p;
                break;
            }
        }

        if (!$parent) {
            sb_json_error('PARENT_PAGE_NOT_FOUND', 404);
        }
    }

    $maxSort = 0;
    foreach ($pages as $p) {
        if (
            (int)($p['siteId'] ?? 0) === $siteId &&
            (int)($p['parentId'] ?? 0) === $parentId
        ) {
            $maxSort = max($maxSort, (int)($p['sort'] ?? 0));
        }
    }

    $id = sb_next_id($pages, 'id');

    $page = sb_normalize_page_record([
        'id' => $id,
        'siteId' => $siteId,
        'title' => $title,
        'slug' => $slug,
        'parentId' => $parentId,
        'sort' => $maxSort > 0 ? ($maxSort + 10) : 10,
        'status' => 'draft',
        'publishedAt' => null,
        'createdAt' => date('c'),
        'updatedAt' => date('c'),
    ]);

    $pages[] = $page;
    sb_write_pages($pages);

    sb_json_ok([
        'page' => $page,
    ]);
}

if ($action === 'page.updateMeta') {
    $id = (int)($_POST['id'] ?? 0);
    $title = trim((string)($_POST['title'] ?? ''));
    $slug = trim((string)($_POST['slug'] ?? ''));
    $parentId = isset($_POST['parentId']) ? (int)$_POST['parentId'] : null;

    if ($id <= 0) {
        sb_json_error('PAGE_ID_REQUIRED', 422);
    }

    if ($title === '') {
        sb_json_error('TITLE_REQUIRED', 422);
    }

    $pages = sb_read_pages();
    $index = null;
    $page = null;

    foreach ($pages as $k => $p) {
        if ((int)($p['id'] ?? 0) === $id) {
            $index = $k;
            $page = $p;
            break;
        }
    }

    if ($index === null || !$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    if ($slug === '') {
        $slug = sb_slugify($title);
    }

    if ($parentId !== null) {
        if ($parentId === $id) {
            sb_json_error('PAGE_CANNOT_BE_OWN_PARENT', 422);
        }

        if ($parentId > 0) {
            $parent = null;
            foreach ($pages as $p) {
                if ((int)($p['id'] ?? 0) === $parentId && (int)($p['siteId'] ?? 0) === $siteId) {
                    $parent = $p;
                    break;
                }
            }

            if (!$parent) {
                sb_json_error('PARENT_PAGE_NOT_FOUND', 404);
            }

            $cursor = $parent;
            $safety = 0;
            while ($cursor && $safety < 1000) {
                $cursorParentId = (int)($cursor['parentId'] ?? 0);
                if ($cursorParentId === 0) {
                    break;
                }
                if ($cursorParentId === $id) {
                    sb_json_error('CYCLIC_PARENT_RELATION', 422);
                }

                $next = null;
                foreach ($pages as $p) {
                    if ((int)($p['id'] ?? 0) === $cursorParentId && (int)($p['siteId'] ?? 0) === $siteId) {
                        $next = $p;
                        break;
                    }
                }
                $cursor = $next;
                $safety++;
            }
        }
    }

    $page['title'] = $title;
    $page['slug'] = $slug;
    if ($parentId !== null) {
        $page['parentId'] = $parentId;
    }
    $page['updatedAt'] = date('c');

    $pages[$index] = sb_normalize_page_record($page);
    sb_write_pages($pages);

    sb_json_ok([
        'page' => $pages[$index],
    ]);
}

if ($action === 'page.setParent') {
    $id = (int)($_POST['id'] ?? 0);
    $parentId = (int)($_POST['parentId'] ?? 0);

    if ($id <= 0) {
        sb_json_error('PAGE_ID_REQUIRED', 422);
    }

    $pages = sb_read_pages();
    $index = null;
    $page = null;

    foreach ($pages as $k => $p) {
        if ((int)($p['id'] ?? 0) === $id) {
            $index = $k;
            $page = $p;
            break;
        }
    }

    if ($index === null || !$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    if ($parentId === $id) {
        sb_json_error('PAGE_CANNOT_BE_OWN_PARENT', 422);
    }

    if ($parentId > 0) {
        $parent = null;
        foreach ($pages as $p) {
            if ((int)($p['id'] ?? 0) === $parentId && (int)($p['siteId'] ?? 0) === $siteId) {
                $parent = $p;
                break;
            }
        }

        if (!$parent) {
            sb_json_error('PARENT_PAGE_NOT_FOUND', 404);
        }

        $cursor = $parent;
        $safety = 0;
        while ($cursor && $safety < 1000) {
            $cursorParentId = (int)($cursor['parentId'] ?? 0);
            if ($cursorParentId === 0) {
                break;
            }
            if ($cursorParentId === $id) {
                sb_json_error('CYCLIC_PARENT_RELATION', 422);
            }

            $next = null;
            foreach ($pages as $p) {
                if ((int)($p['id'] ?? 0) === $cursorParentId && (int)($p['siteId'] ?? 0) === $siteId) {
                    $next = $p;
                    break;
                }
            }
            $cursor = $next;
            $safety++;
        }
    }

    $page['parentId'] = $parentId;
    $page['updatedAt'] = date('c');
    $pages[$index] = sb_normalize_page_record($page);

    sb_write_pages($pages);

    sb_json_ok([
        'page' => $pages[$index],
    ]);
}

if ($action === 'page.setStatus') {
    $id = (int)($_POST['id'] ?? 0);
    $status = trim((string)($_POST['status'] ?? ''));

    if ($id <= 0) {
        sb_json_error('PAGE_ID_REQUIRED', 422);
    }

    if (!in_array($status, ['draft', 'published'], true)) {
        sb_json_error('INVALID_STATUS', 422);
    }

    $pages = sb_read_pages();
    $index = null;
    $page = null;

    foreach ($pages as $k => $p) {
        if ((int)($p['id'] ?? 0) === $id) {
            $index = $k;
            $page = $p;
            break;
        }
    }

    if ($index === null || !$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $page['status'] = $status;
    $page['publishedAt'] = $status === 'published' ? date('c') : null;
    $page['updatedAt'] = date('c');

    $pages[$index] = sb_normalize_page_record($page);
    sb_write_pages($pages);

    sb_json_ok([
        'page' => $pages[$index],
    ]);
}

if ($action === 'page.move') {
    $id = (int)($_POST['id'] ?? 0);
    $dir = trim((string)($_POST['dir'] ?? ''));

    if ($id <= 0) {
        sb_json_error('PAGE_ID_REQUIRED', 422);
    }

    if (!in_array($dir, ['up', 'down'], true)) {
        sb_json_error('INVALID_DIR', 422);
    }

    $pages = sb_read_pages();
    $page = null;

    foreach ($pages as $p) {
        if ((int)($p['id'] ?? 0) === $id) {
            $page = $p;
            break;
        }
    }

    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    $parentId = (int)($page['parentId'] ?? 0);

    sb_require_editor($siteId);

    $siblings = [];
    foreach ($pages as $k => $p) {
        if (
            (int)($p['siteId'] ?? 0) === $siteId &&
            (int)($p['parentId'] ?? 0) === $parentId
        ) {
            $siblings[] = [
                'index' => $k,
                'row' => $p,
            ];
        }
    }

    usort($siblings, static function ($a, $b) {
        $sortCmp = (int)($a['row']['sort'] ?? 500) <=> (int)($b['row']['sort'] ?? 500);
        if ($sortCmp !== 0) {
            return $sortCmp;
        }
        return (int)($a['row']['id'] ?? 0) <=> (int)($b['row']['id'] ?? 0);
    });

    $pos = null;
    for ($i = 0; $i < count($siblings); $i++) {
        if ((int)($siblings[$i]['row']['id'] ?? 0) === $id) {
            $pos = $i;
            break;
        }
    }

    if ($pos === null) {
        sb_json_error('PAGE_NOT_FOUND_IN_SIBLINGS', 404);
    }

    $swapPos = $dir === 'up' ? $pos - 1 : $pos + 1;
    if (!isset($siblings[$swapPos])) {
        sb_json_ok(['moved' => false]);
    }

    $aIndex = $siblings[$pos]['index'];
    $bIndex = $siblings[$swapPos]['index'];

    $aSort = (int)($pages[$aIndex]['sort'] ?? 500);
    $bSort = (int)($pages[$bIndex]['sort'] ?? 500);

    $pages[$aIndex]['sort'] = $bSort;
    $pages[$aIndex]['updatedAt'] = date('c');

    $pages[$bIndex]['sort'] = $aSort;
    $pages[$bIndex]['updatedAt'] = date('c');

    $pages[$aIndex] = sb_normalize_page_record($pages[$aIndex]);
    $pages[$bIndex] = sb_normalize_page_record($pages[$bIndex]);

    sb_write_pages($pages);

    sb_json_ok(['moved' => true]);
}

if ($action === 'page.delete') {
    $id = (int)($_POST['id'] ?? 0);

    if ($id <= 0) {
        sb_json_error('PAGE_ID_REQUIRED', 422);
    }

    $pages = sb_read_pages();
    $page = null;

    foreach ($pages as $p) {
        if ((int)($p['id'] ?? 0) === $id) {
            $page = $p;
            break;
        }
    }

    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $idsToDelete = [$id => true];
    $changed = true;
    $safety = 0;

    while ($changed && $safety < 1000) {
        $changed = false;
        foreach ($pages as $p) {
            $pid = (int)($p['id'] ?? 0);
            $parentId = (int)($p['parentId'] ?? 0);
            if ($pid > 0 && !isset($idsToDelete[$pid]) && isset($idsToDelete[$parentId])) {
                $idsToDelete[$pid] = true;
                $changed = true;
            }
        }
        $safety++;
    }

    $pages = array_values(array_filter($pages, static function ($p) use ($idsToDelete) {
        return !isset($idsToDelete[(int)($p['id'] ?? 0)]);
    }));
    sb_write_pages($pages);

    $blocks = sb_read_blocks();
    $blocks = array_values(array_filter($blocks, static function ($b) use ($idsToDelete) {
        return !isset($idsToDelete[(int)($b['pageId'] ?? 0)]);
    }));
    sb_write_blocks($blocks);

    sb_json_ok([
        'deleted' => true,
        'deletedPageIds' => array_map('intval', array_keys($idsToDelete)),
    ]);
}

if ($action === 'page.duplicate') {
    $id = (int)($_POST['id'] ?? 0);

    if ($id <= 0) {
        sb_json_error('PAGE_ID_REQUIRED', 422);
    }

    $pages = sb_read_pages();
    $source = null;

    foreach ($pages as $p) {
        if ((int)($p['id'] ?? 0) === $id) {
            $source = $p;
            break;
        }
    }

    if (!$source) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($source['siteId'] ?? 0);
    sb_require_editor($siteId);

    $newId = sb_next_id($pages, 'id');

    $maxSort = 0;
    foreach ($pages as $p) {
        if (
            (int)($p['siteId'] ?? 0) === $siteId &&
            (int)($p['parentId'] ?? 0) === (int)($source['parentId'] ?? 0)
        ) {
            $maxSort = max($maxSort, (int)($p['sort'] ?? 0));
        }
    }

    $copy = sb_normalize_page_record([
        'id' => $newId,
        'siteId' => $siteId,
        'title' => (string)($source['title'] ?? '') . ' (копия)',
        'slug' => sb_slugify((string)($source['slug'] ?? 'page') . '-' . $newId),
        'parentId' => (int)($source['parentId'] ?? 0),
        'sort' => $maxSort > 0 ? ($maxSort + 10) : ((int)($source['sort'] ?? 10) + 10),
        'status' => 'draft',
        'publishedAt' => null,
        'createdAt' => date('c'),
        'updatedAt' => date('c'),
    ]);

    $pages[] = $copy;
    sb_write_pages($pages);

    $blocks = sb_read_blocks();
    $sourceBlocks = array_values(array_filter($blocks, static function ($b) use ($id) {
        return (int)($b['pageId'] ?? 0) === $id;
    }));

    foreach ($sourceBlocks as $b) {
        $newBlockId = sb_next_id($blocks, 'id');
        $newBlock = sb_normalize_block_record([
            'id' => $newBlockId,
            'pageId' => $newId,
            'type' => (string)($b['type'] ?? 'text'),
            'sort' => (int)($b['sort'] ?? 500),
            'content' => is_array($b['content'] ?? null) ? $b['content'] : [],
            'props' => is_array($b['props'] ?? null) ? $b['props'] : [],
            'createdAt' => date('c'),
            'updatedAt' => date('c'),
        ]);
        $blocks[] = $newBlock;
    }

    sb_write_blocks($blocks);

    sb_json_ok([
        'page' => $copy,
    ]);
}

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'page',
    'action' => $action,
]);


---

2. Что ещё нужно проверить в api/index.php

В файле:

/local/sitebuilder/api/index.php

должен быть роут на page.php.

Если у тебя уже есть что-то вроде:

if (strpos($action, 'page.') === 0) {
    require __DIR__ . '/handlers/page.php';
    exit;
}

то ничего не делай.

Если нет — добавь этот блок.


---

3. Теперь замени editor.php

Полностью замени файл:

/local/sitebuilder/editor.php

на этот:

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

global $APPLICATION, $USER;

if (!$USER->IsAuthorized()) {
    require $_SERVER['DOCUMENT_ROOT'] . '/auth.php';
    exit;
}

CJSCore::Init(['ajax']);

header('Content-Type: text/html; charset=UTF-8');

$basePath = rtrim(str_replace($_SERVER['DOCUMENT_ROOT'], '', __DIR__), '/');
$siteId = (int)($_GET['siteId'] ?? 0);

if ($siteId <= 0) {
    ?>
    <!doctype html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <title>SiteBuilder / Editor</title>
        <?php $APPLICATION->ShowHead(); ?>
        <link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/admin.css">
    </head>
    <body class="sb-admin-body">
        <div class="sb-page">
            <h1 class="sb-title">Не передан siteId</h1>
            <p><a class="sb-back-link" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/index.php">Вернуться к списку сайтов</a></p>
        </div>
    </body>
    </html>
    <?php
    exit;
}
?>
<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>SiteBuilder / Editor</title>
    <?php $APPLICATION->ShowHead(); ?>
    <link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/admin.css">
    <style>
        .sb-editor-topline {
            display: flex;
            justify-content: space-between;
            align-items: flex-start;
            gap: 16px;
            margin-bottom: 16px;
        }

        .sb-editor-topline-note {
            margin: 0;
            color: #6b7280;
            font-size: 14px;
            max-width: 760px;
        }

        .sb-editor-topline-actions {
            display: flex;
            flex-wrap: wrap;
            gap: 8px;
            justify-content: flex-end;
        }

        .sb-editor-main {
            display: grid;
            grid-template-columns: 360px minmax(0, 1fr) 360px;
            gap: 18px;
            align-items: start;
        }

        .sb-editor-panel-sticky {
            position: sticky;
            top: 16px;
        }

        .sb-editor-subhead {
            display: flex;
            justify-content: space-between;
            align-items: center;
            gap: 12px;
            margin-bottom: 14px;
        }

        .sb-editor-subhead .sb-panel-title {
            margin: 0;
        }

        .sb-editor-create-page {
            margin-bottom: 14px;
            padding-bottom: 14px;
            border-bottom: 1px solid #eef2f7;
        }

        .sb-editor-pages-list,
        .sb-editor-blocks-list {
            display: flex;
            flex-direction: column;
            gap: 10px;
        }

        .sb-editor-page-card,
        .sb-editor-block-card {
            border: 1px solid #e5e7eb;
            border-radius: 14px;
            padding: 12px;
            background: #fafafa;
            transition: border-color .15s ease, box-shadow .15s ease, background .15s ease;
        }

        .sb-editor-page-card {
            cursor: pointer;
        }

        .sb-editor-page-card:hover {
            border-color: #c7d2fe;
            background: #fcfcff;
        }

        .sb-editor-page-card.active {
            border-color: #2563eb;
            background: #eff6ff;
            box-shadow: 0 4px 14px rgba(37, 99, 235, 0.12);
        }

        .sb-editor-page-card.has-children .sb-editor-page-title::after {
            content: 'section';
            display: inline-flex;
            align-items: center;
            margin-left: 8px;
            min-height: 20px;
            padding: 0 8px;
            border-radius: 999px;
            background: #eef2ff;
            color: #3730a3;
            font-size: 11px;
            font-weight: 700;
            vertical-align: middle;
        }

        .sb-editor-block-card.selected {
            border-color: #2563eb;
            background: #f8fbff;
            box-shadow: 0 4px 14px rgba(37, 99, 235, 0.12);
        }

        .sb-editor-page-head,
        .sb-editor-block-head {
            display: flex;
            justify-content: space-between;
            gap: 12px;
            align-items: flex-start;
        }

        .sb-editor-page-title,
        .sb-editor-block-title {
            margin: 0 0 6px;
            font-size: 16px;
            font-weight: 700;
            line-height: 1.2;
            color: #111827;
        }

        .sb-editor-page-slug {
            display: inline-flex;
            align-items: center;
            min-height: 24px;
            padding: 0 8px;
            border-radius: 999px;
            background: #f3f4f6;
            color: #4b5563;
            font-size: 12px;
            font-weight: 600;
        }

        .sb-editor-page-tree-line {
            display: flex;
            align-items: center;
            gap: 8px;
            margin-bottom: 8px;
        }

        .sb-editor-page-indent {
            width: var(--indent-width, 0px);
            flex: 0 0 var(--indent-width, 0px);
        }

        .sb-editor-page-branch {
            color: #94a3b8;
            font-size: 12px;
            font-weight: 700;
        }

        .sb-editor-page-meta {
            display: grid;
            grid-template-columns: repeat(2, 1fr);
            gap: 8px 10px;
            margin-top: 10px;
            padding-top: 10px;
            border-top: 1px dashed #e5e7eb;
        }

        .sb-editor-page-meta-item {
            min-width: 0;
        }

        .sb-editor-page-meta-label {
            font-size: 11px;
            color: #6b7280;
            margin-bottom: 2px;
        }

        .sb-editor-page-meta-value {
            font-size: 13px;
            color: #111827;
            line-height: 1.3;
            word-break: break-word;
        }

        .sb-editor-page-actions,
        .sb-editor-block-actions {
            display: flex;
            flex-wrap: wrap;
            gap: 8px;
            margin-top: 12px;
        }

        .sb-editor-page-actions .sb-btn,
        .sb-editor-block-actions .sb-btn {
            height: 30px;
            padding: 0 10px;
            font-size: 12px;
        }

        .sb-editor-current-page-card {
            margin-bottom: 14px;
            padding: 14px;
            border: 1px solid #dbe7ff;
            border-radius: 14px;
            background: linear-gradient(180deg, #ffffff 0%, #f8fbff 100%);
        }

        .sb-editor-current-page-title {
            margin: 0 0 6px;
            font-size: 18px;
            font-weight: 700;
            color: #111827;
        }

        .sb-editor-statbar {
            display: flex;
            flex-wrap: wrap;
            gap: 8px;
            margin-bottom: 14px;
        }

        .sb-editor-stat {
            display: inline-flex;
            align-items: center;
            min-height: 30px;
            padding: 0 10px;
            border-radius: 999px;
            background: #f3f4f6;
            color: #374151;
            font-size: 12px;
            font-weight: 600;
        }

        .sb-editor-block-toolbar {
            display: flex;
            flex-wrap: wrap;
            gap: 8px;
            margin-bottom: 14px;
        }

        .sb-editor-block-preview {
            margin-top: 12px;
            padding: 12px;
            border-radius: 12px;
            background: #fff;
            border: 1px solid #e5e7eb;
            color: #374151;
            line-height: 1.5;
            min-height: 48px;
        }

        .sb-editor-side-note {
            font-size: 13px;
            color: #6b7280;
            margin-top: -2px;
            margin-bottom: 12px;
        }

        .sb-editor-side-empty {
            display: flex;
            flex-direction: column;
            gap: 10px;
            align-items: flex-start;
            text-align: left;
        }

        .sb-editor-tip {
            display: inline-flex;
            align-items: center;
            min-height: 28px;
            padding: 0 10px;
            border-radius: 999px;
            background: #eef2ff;
            color: #3730a3;
            font-size: 12px;
            font-weight: 600;
        }

        .sb-editor-json-actions {
            display: flex;
            gap: 8px;
            flex-wrap: wrap;
            margin-top: 12px;
        }

        @media (max-width: 1380px) {
            .sb-editor-main {
                grid-template-columns: 340px 1fr;
            }

            .sb-editor-main > .sb-panel:last-child {
                grid-column: 1 / -1;
            }

            .sb-editor-panel-sticky {
                position: static;
            }
        }

        @media (max-width: 960px) {
            .sb-editor-topline {
                flex-direction: column;
                align-items: stretch;
            }

            .sb-editor-topline-actions {
                justify-content: flex-start;
            }

            .sb-editor-main {
                grid-template-columns: 1fr;
            }

            .sb-editor-page-meta {
                grid-template-columns: 1fr 1fr;
            }
        }

        @media (max-width: 640px) {
            .sb-editor-page-meta {
                grid-template-columns: 1fr;
            }
        }
    </style>
</head>
<body class="sb-admin-body">
<div class="sb-page">
    <div class="sb-topbar">
        <div class="sb-topbar-left">
            <a class="sb-back-link" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/index.php">← К списку сайтов</a>
            <h1 class="sb-title">Редактор сайта</h1>
            <p class="sb-subtitle">siteId = <?= (int)$siteId ?></p>
        </div>
    </div>

    <div class="sb-editor-topline">
        <p class="sb-editor-topline-note">
            Создавай страницы, выстраивай дерево разделов через parentId и редактируй блоки. Если у страницы есть дочерние страницы — она работает как section.
        </p>
        <div class="sb-editor-topline-actions">
            <a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/public.php?siteId=<?= (int)$siteId ?>" target="_blank">Открыть публичную</a>
            <a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/layout.php?siteId=<?= (int)$siteId ?>">Layout</a>
            <a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/menu.php?siteId=<?= (int)$siteId ?>">Меню</a>
        </div>
    </div>

    <div class="sb-editor-main">
        <div class="sb-panel">
            <div class="sb-editor-subhead">
                <h2 class="sb-panel-title">Страницы</h2>
                <span class="sb-badge">siteId <?= (int)$siteId ?></span>
            </div>

            <div class="sb-editor-create-page">
                <div class="sb-form-row align-end">
                    <div class="sb-field">
                        <label for="newPageTitle">Название страницы</label>
                        <input class="sb-input" type="text" id="newPageTitle" placeholder="Например: Главная">
                    </div>
                    <div class="sb-field">
                        <label for="newPageSlug">Slug</label>
                        <input class="sb-input" type="text" id="newPageSlug" placeholder="Например: home">
                    </div>
                </div>

                <div class="sb-form-row align-end" style="margin-top:12px;">
                    <div class="sb-field">
                        <label for="newPageParentId">Родительская страница</label>
                        <select class="sb-select" id="newPageParentId">
                            <option value="0">Без родителя</option>
                        </select>
                    </div>
                    <button type="button" class="sb-btn sb-btn-primary" id="createPageBtn">Создать</button>
                </div>
            </div>

            <div id="pagesContainer" class="sb-editor-pages-list">
                <div class="sb-empty">Загрузка страниц...</div>
            </div>
        </div>

        <div class="sb-panel">
            <div class="sb-editor-subhead">
                <h2 class="sb-panel-title">Блоки страницы</h2>
                <button type="button" class="sb-btn sb-btn-light sb-btn-small" id="reloadBlocksBtn">Обновить</button>
            </div>

            <div id="currentPageCard" class="sb-editor-current-page-card">
                <h3 class="sb-editor-current-page-title">Страница не выбрана</h3>
                <div id="currentPageInfo" class="sb-meta">Выберите страницу слева, чтобы увидеть её блоки.</div>
            </div>

            <div class="sb-editor-statbar">
                <div class="sb-editor-stat">Страниц: <span id="pagesCount" style="margin-left:6px;">0</span></div>
                <div class="sb-editor-stat">Блоков: <span id="blocksCount" style="margin-left:6px;">0</span></div>
            </div>

            <div class="sb-editor-block-toolbar">
                <button type="button" class="sb-btn sb-btn-light sb-btn-small js-add-block" data-type="text">+ Text</button>
                <button type="button" class="sb-btn sb-btn-light sb-btn-small js-add-block" data-type="heading">+ Heading</button>
                <button type="button" class="sb-btn sb-btn-light sb-btn-small js-add-block" data-type="button">+ Button</button>
                <button type="button" class="sb-btn sb-btn-light sb-btn-small js-add-block" data-type="html">+ HTML</button>
                <button type="button" class="sb-btn sb-btn-light sb-btn-small js-add-block" data-type="spacer">+ Spacer</button>
            </div>

            <div id="blocksContainer" class="sb-editor-blocks-list">
                <div class="sb-empty">Выберите страницу</div>
            </div>
        </div>

        <div class="sb-panel sb-editor-panel-sticky">
            <h2 class="sb-panel-title">Редактор страницы / блока</h2>
            <div class="sb-editor-side-note">Можно менять родителя страницы и редактировать JSON выбранного блока.</div>

            <div class="sb-field" style="margin-bottom:12px;">
                <label for="editPageParentId">Родитель текущей страницы</label>
                <select class="sb-select" id="editPageParentId">
                    <option value="0">Без родителя</option>
                </select>
            </div>

            <div class="sb-form-row" style="margin-bottom:16px;">
                <button type="button" class="sb-btn sb-btn-light" id="savePageParentBtn">Сохранить parent</button>
            </div>

            <div id="blockEditorEmpty" class="sb-empty sb-editor-side-empty">
                <span class="sb-editor-tip">Подсказка</span>
                <div>Выберите блок в центральной колонке. После этого справа откроется его JSON-редактор.</div>
            </div>

            <div id="blockEditorForm" class="sb-hidden">
                <div class="sb-field" style="margin-bottom:12px;">
                    <label for="editBlockType">Тип</label>
                    <input class="sb-input" type="text" id="editBlockType" readonly>
                </div>

                <div class="sb-field" style="margin-bottom:12px;">
                    <label for="editBlockContentText">Content / JSON</label>
                    <textarea class="sb-textarea" id="editBlockContentText"></textarea>
                </div>

                <div class="sb-field" style="margin-bottom:12px;">
                    <label for="editBlockPropsText">Props / JSON</label>
                    <textarea class="sb-textarea" id="editBlockPropsText">{}</textarea>
                </div>

                <div class="sb-form-row">
                    <button type="button" class="sb-btn sb-btn-primary" id="saveBlockBtn">Сохранить блок</button>
                    <button type="button" class="sb-btn sb-btn-danger" id="deleteBlockBtn">Удалить блок</button>
                </div>

                <div class="sb-editor-json-actions">
                    <button type="button" class="sb-btn sb-btn-gray sb-btn-small" id="formatJsonBtn">Форматировать JSON</button>
                    <button type="button" class="sb-btn sb-btn-gray sb-btn-small" id="resetPropsBtn">Сбросить props</button>
                </div>
            </div>
        </div>
    </div>

    <div class="sb-panel" style="margin-top:20px;">
        <h2 class="sb-panel-title">Отладка</h2>
        <div id="output" class="sb-output">Здесь будут ответы API...</div>
    </div>
</div>

<script>
(function () {
    var BASE_PATH = '<?= CUtil::JSEscape($basePath) ?>';
    var API_URL = BASE_PATH + '/api.php';
    var SITE_ID = <?= (int)$siteId ?>;

    var output = document.getElementById('output');
    var pagesContainer = document.getElementById('pagesContainer');
    var blocksContainer = document.getElementById('blocksContainer');
    var currentPageInfo = document.getElementById('currentPageInfo');
    var currentPageCardTitle = document.querySelector('#currentPageCard .sb-editor-current-page-title');
    var pagesCount = document.getElementById('pagesCount');
    var blocksCount = document.getElementById('blocksCount');
    var newPageParentId = document.getElementById('newPageParentId');
    var editPageParentId = document.getElementById('editPageParentId');

    var state = {
        site: null,
        pages: [],
        blocks: [],
        currentPageId: 0,
        currentBlockId: 0
    };

    function print(data) {
        if (typeof data === 'string') {
            output.textContent = data;
            return;
        }
        try {
            output.textContent = JSON.stringify(data, null, 2);
        } catch (e) {
            output.textContent = String(data);
        }
    }

    function escapeHtml(value) {
        return String(value)
            .replace(/&/g, '&amp;')
            .replace(/</g, '&lt;')
            .replace(/>/g, '&gt;')
            .replace(/"/g, '&quot;')
            .replace(/'/g, '&#039;');
    }

    function getSessid() {
        if (typeof window.BX !== 'undefined' && typeof BX.bitrix_sessid === 'function') {
            return BX.bitrix_sessid();
        }
        return '<?= CUtil::JSEscape(bitrix_sessid()) ?>';
    }

    function api(action, data, onSuccess, onFailure) {
        if (typeof window.BX === 'undefined' || typeof BX.ajax !== 'function') {
            print('BX.ajax не загружен');
            return;
        }

        BX.ajax({
            url: API_URL,
            method: 'POST',
            dataType: 'json',
            timeout: 60,
            data: Object.assign({
                action: action,
                sessid: getSessid()
            }, data || {}),
            onsuccess: function (res) {
                print(res);
                if (typeof onSuccess === 'function') {
                    onSuccess(res);
                }
            },
            onfailure: function (err) {
                print({
                    ok: false,
                    error: 'AJAX_ERROR',
                    detail: err
                });
                if (typeof onFailure === 'function') {
                    onFailure(err);
                }
            }
        });
    }

    function updateStats() {
        pagesCount.textContent = String(Array.isArray(state.pages) ? state.pages.length : 0);
        blocksCount.textContent = String(Array.isArray(state.blocks) ? state.blocks.length : 0);
    }

    function findPage(pageId) {
        for (var i = 0; i < state.pages.length; i++) {
            if (Number(state.pages[i].id || 0) === Number(pageId || 0)) {
                return state.pages[i];
            }
        }
        return null;
    }

    function findBlock(blockId) {
        for (var i = 0; i < state.blocks.length; i++) {
            if (Number(state.blocks[i].id || 0) === Number(blockId || 0)) {
                return state.blocks[i];
            }
        }
        return null;
    }

    function buildPageTree(pages) {
        var map = {};
        var roots = [];

        for (var i = 0; i < pages.length; i++) {
            var page = Object.assign({}, pages[i]);
            page.children = [];
            map[Number(page.id || 0)] = page;
        }

        for (var j = 0; j < pages.length; j++) {
            var current = map[Number(pages[j].id || 0)];
            var parentId = Number(current.parentId || 0);

            if (parentId > 0 && map[parentId]) {
                map[parentId].children.push(current);
            } else {
                roots.push(current);
            }
        }

        function sortNodes(nodes) {
            nodes.sort(function (a, b) {
                var sortCmp = Number(a.sort || 0) - Number(b.sort || 0);
                if (sortCmp !== 0) return sortCmp;
                return Number(a.id || 0) - Number(b.id || 0);
            });

            for (var k = 0; k < nodes.length; k++) {
                sortNodes(nodes[k].children);
            }
        }

        sortNodes(roots);
        return roots;
    }

    function flattenTree(nodes, depth, out) {
        depth = depth || 0;
        out = out || [];

        for (var i = 0; i < nodes.length; i++) {
            nodes[i]._depth = depth;
            nodes[i]._hasChildren = Array.isArray(nodes[i].children) && nodes[i].children.length > 0;
            out.push(nodes[i]);
            if (nodes[i].children && nodes[i].children.length) {
                flattenTree(nodes[i].children, depth + 1, out);
            }
        }

        return out;
    }

    function fillParentSelect(selectEl, excludePageId, selectedParentId) {
        var tree = buildPageTree(state.pages);
        var flat = flattenTree(tree, 0, []);
        var html = '<option value="0">Без родителя</option>';

        for (var i = 0; i < flat.length; i++) {
            var page = flat[i];
            var pid = Number(page.id || 0);

            if (excludePageId > 0 && pid === excludePageId) {
                continue;
            }

            var prefix = '';
            for (var d = 0; d < Number(page._depth || 0); d++) {
                prefix += '— ';
            }

            var selected = pid === Number(selectedParentId || 0) ? ' selected' : '';
            html += '<option value="' + pid + '"' + selected + '>'
                + escapeHtml(prefix + (page.title || ('Страница #' + pid)))
                + '</option>';
        }

        selectEl.innerHTML = html;
    }

    function loadSite(next) {
        api('site.get', { siteId: SITE_ID }, function (res) {
            if (res && res.ok === true) {
                state.site = res.site || null;
            }
            if (typeof next === 'function') {
                next();
            }
        });
    }

    function loadPages(next) {
        pagesContainer.innerHTML = '<div class="sb-empty">Загрузка страниц...</div>';

        api('page.list', { siteId: SITE_ID }, function (res) {
            if (!res || res.ok !== true) {
                pagesContainer.innerHTML = '<div class="sb-empty">Не удалось загрузить страницы</div>';
                updateStats();
                return;
            }

            state.pages = Array.isArray(res.pages) ? res.pages : [];

            if (!state.currentPageId && state.pages.length) {
                state.currentPageId = Number(state.pages[0].id || 0);
            }

            var hasCurrent = false;
            for (var i = 0; i < state.pages.length; i++) {
                if (Number(state.pages[i].id || 0) === Number(state.currentPageId || 0)) {
                    hasCurrent = true;
                    break;
                }
            }

            if (!hasCurrent) {
                state.currentPageId = state.pages.length ? Number(state.pages[0].id || 0) : 0;
            }

            fillParentSelect(newPageParentId, 0, 0);

            var currentPage = findPage(state.currentPageId);
            fillParentSelect(editPageParentId, state.currentPageId, currentPage ? Number(currentPage.parentId || 0) : 0);

            renderPages();
            updateStats();

            if (typeof next === 'function') {
                next();
            }
        });
    }

    function renderPages() {
        if (!state.pages.length) {
            pagesContainer.innerHTML = '<div class="sb-empty">Страниц пока нет</div>';
            currentPageCardTitle.textContent = 'Страница не выбрана';
            currentPageInfo.textContent = 'Создайте первую страницу, чтобы начать редактирование.';
            blocksContainer.innerHTML = '<div class="sb-empty">Выберите страницу</div>';
            return;
        }

        var tree = buildPageTree(state.pages);
        var flat = flattenTree(tree, 0, []);

        var html = '';
        for (var i = 0; i < flat.length; i++) {
            html += renderPageCard(flat[i]);
        }
        pagesContainer.innerHTML = html;
    }

    function renderPageCard(page) {
        var id = Number(page.id || 0);
        var active = id === Number(state.currentPageId || 0) ? ' active' : '';
        var hasChildren = page._hasChildren ? ' has-children' : '';
        var isPublished = String(page.status || '') === 'published';
        var badge = isPublished
            ? '<span class="sb-badge sb-badge-green">published</span>'
            : '<span class="sb-badge sb-badge-yellow">draft</span>';

        var depth = Number(page._depth || 0);
        var indentWidth = depth * 18;

        return ''
            + '<div class="sb-editor-page-card' + active + hasChildren + ' js-page-card" data-id="' + id + '">'
            + '  <div class="sb-editor-page-tree-line">'
            + '    <span class="sb-editor-page-indent" style="--indent-width:' + indentWidth + 'px;"></span>'
            + '    <span class="sb-editor-page-branch">' + (depth > 0 ? '└' : '') + '</span>'
            + '    <div style="min-width:0; width:100%;">'
            + '      <div class="sb-editor-page-head">'
            + '        <div>'
            + '          <div class="sb-editor-page-title">' + escapeHtml(page.title || '') + '</div>'
            + '          <div class="sb-editor-page-slug">' + escapeHtml(page.slug || '') + '</div>'
            + '        </div>'
            + '        ' + badge
            + '      </div>'
            + '    </div>'
            + '  </div>'
            + '  <div class="sb-editor-page-meta">'
            + '    <div class="sb-editor-page-meta-item">'
            + '      <div class="sb-editor-page-meta-label">ID</div>'
            + '      <div class="sb-editor-page-meta-value">' + id + '</div>'
            + '    </div>'
            + '    <div class="sb-editor-page-meta-item">'
            + '      <div class="sb-editor-page-meta-label">Parent ID</div>'
            + '      <div class="sb-editor-page-meta-value">' + Number(page.parentId || 0) + '</div>'
            + '    </div>'
            + '    <div class="sb-editor-page-meta-item">'
            + '      <div class="sb-editor-page-meta-label">Sort</div>'
            + '      <div class="sb-editor-page-meta-value">' + Number(page.sort || 0) + '</div>'
            + '    </div>'
            + '    <div class="sb-editor-page-meta-item">'
            + '      <div class="sb-editor-page-meta-label">Published</div>'
            + '      <div class="sb-editor-page-meta-value">' + escapeHtml(page.publishedAt || '—') + '</div>'
            + '    </div>'
            + '  </div>'
            + '  <div class="sb-editor-page-actions">'
            + '    <button type="button" class="sb-btn sb-btn-primary js-open-page" data-id="' + id + '">Открыть</button>'
            + '    <button type="button" class="sb-btn sb-btn-light js-rename-page" data-id="' + id + '">Meta</button>'
            + '    <button type="button" class="sb-btn sb-btn-light js-duplicate-page" data-id="' + id + '">Дублировать</button>'
            + '    <button type="button" class="sb-btn sb-btn-light js-toggle-status" data-id="' + id + '" data-status="' + escapeHtml(page.status || 'draft') + '">'
            +         (isPublished ? 'В draft' : 'Опубликовать')
            + '    </button>'
            + '    <button type="button" class="sb-btn sb-btn-gray js-move-page-up" data-id="' + id + '">↑</button>'
            + '    <button type="button" class="sb-btn sb-btn-gray js-move-page-down" data-id="' + id + '">↓</button>'
            + '    <button type="button" class="sb-btn sb-btn-danger js-delete-page" data-id="' + id + '">Удалить</button>'
            + '  </div>'
            + '</div>';
    }

    function createPage() {
        var titleInput = document.getElementById('newPageTitle');
        var slugInput = document.getElementById('newPageSlug');
        var parentId = parseInt(newPageParentId.value, 10) || 0;

        var title = (titleInput.value || '').trim();
        var slug = (slugInput.value || '').trim();

        if (!title) {
            alert('Введите название страницы');
            titleInput.focus();
            return;
        }

        api('page.create', {
            siteId: SITE_ID,
            title: title,
            slug: slug,
            parentId: parentId
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось создать страницу');
                return;
            }

            titleInput.value = '';
            slugInput.value = '';
            newPageParentId.value = '0';

            state.currentPageId = Number((res.page && res.page.id) || 0);
            loadPages(loadBlocks);
        });
    }

    function saveCurrentPageParent() {
        if (!state.currentPageId) {
            alert('Сначала выберите страницу');
            return;
        }

        var parentId = parseInt(editPageParentId.value, 10) || 0;

        api('page.setParent', {
            id: state.currentPageId,
            parentId: parentId
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось изменить parent');
                return;
            }

            loadPages(function () {
                loadBlocks();
            });
        });
    }

    function openPage(pageId) {
        state.currentPageId = Number(pageId || 0);
        state.currentBlockId = 0;

        var currentPage = findPage(state.currentPageId);
        fillParentSelect(editPageParentId, state.currentPageId, currentPage ? Number(currentPage.parentId || 0) : 0);

        renderPages();
        loadBlocks();
        clearBlockEditor();
    }

    function renamePage(pageId) {
        var page = findPage(pageId);
        if (!page) {
            alert('Страница не найдена');
            return;
        }

        var title = prompt('Название страницы:', page.title || '');
        if (title === null) {
            return;
        }
        title = title.trim();
        if (!title) {
            alert('Название не может быть пустым');
            return;
        }

        var slug = prompt('Slug страницы:', page.slug || '');
        if (slug === null) {
            return;
        }
        slug = slug.trim();

        api('page.updateMeta', {
            id: pageId,
            title: title,
            slug: slug,
            parentId: Number(page.parentId || 0)
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось обновить страницу');
                return;
            }

            loadPages();
        });
    }

    function duplicatePage(pageId) {
        api('page.duplicate', { id: pageId }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось дублировать страницу');
                return;
            }

            state.currentPageId = Number((res.page && res.page.id) || 0);
            loadPages(loadBlocks);
        });
    }

    function togglePageStatus(pageId, currentStatus) {
        var nextStatus = currentStatus === 'published' ? 'draft' : 'published';

        api('page.setStatus', {
            id: pageId,
            status: nextStatus
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось изменить статус');
                return;
            }

            loadPages();
        });
    }

    function movePage(pageId, dir) {
        api('page.move', {
            id: pageId,
            dir: dir
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось переместить страницу');
                return;
            }

            loadPages();
        });
    }

    function deletePage(pageId) {
        if (!confirm('Удалить страницу #' + pageId + ' и все её дочерние страницы?')) {
            return;
        }

        api('page.delete', { id: pageId }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось удалить страницу');
                return;
            }

            if (Number(state.currentPageId || 0) === Number(pageId || 0)) {
                state.currentPageId = 0;
                state.currentBlockId = 0;
                clearBlockEditor();
            }

            loadPages(loadBlocks);
        });
    }

    function loadBlocks() {
        if (!state.currentPageId) {
            currentPageCardTitle.textContent = 'Страница не выбрана';
            currentPageInfo.textContent = 'Выберите страницу слева.';
            blocksContainer.innerHTML = '<div class="sb-empty">Выберите страницу</div>';
            state.blocks = [];
            updateStats();
            return;
        }

        var page = findPage(state.currentPageId);
        if (page) {
            currentPageCardTitle.textContent = page.title || 'Страница';
            currentPageInfo.innerHTML =
                '<strong>ID:</strong> ' + Number(page.id || 0)
                + ' &nbsp;·&nbsp; <strong>Slug:</strong> ' + escapeHtml(page.slug || '')
                + ' &nbsp;·&nbsp; <strong>Status:</strong> ' + escapeHtml(page.status || 'draft')
                + ' &nbsp;·&nbsp; <strong>Parent:</strong> ' + Number(page.parentId || 0);

            fillParentSelect(editPageParentId, state.currentPageId, Number(page.parentId || 0));
        }

        blocksContainer.innerHTML = '<div class="sb-empty">Загрузка блоков...</div>';

        api('block.list', { pageId: state.currentPageId }, function (res) {
            if (!res || res.ok !== true) {
                blocksContainer.innerHTML = '<div class="sb-empty">Не удалось загрузить блоки</div>';
                state.blocks = [];
                updateStats();
                return;
            }

            state.blocks = Array.isArray(res.blocks) ? res.blocks : [];
            renderBlocks();
            updateStats();
        });
    }

    function renderBlocks() {
        if (!state.blocks.length) {
            blocksContainer.innerHTML = '<div class="sb-empty">Блоков пока нет</div>';
            return;
        }

        var html = '';
        for (var i = 0; i < state.blocks.length; i++) {
            html += renderBlockCard(state.blocks[i], i);
        }
        blocksContainer.innerHTML = html;
    }

    function renderBlockCard(block, index) {
        var id = Number(block.id || 0);
        var type = escapeHtml(block.type || '');
        var preview = escapeHtml(buildPreviewText(block));
        var selected = id === Number(state.currentBlockId || 0) ? ' selected' : '';

        return ''
            + '<div class="sb-editor-block-card' + selected + '">'
            + '  <div class="sb-editor-block-head">'
            + '    <div>'
            + '      <div class="sb-editor-block-title">' + type + ' #' + id + '</div>'
            + '      <div class="sb-meta"><strong>Sort:</strong> ' + Number(block.sort || 0) + '</div>'
            + '    </div>'
            + '    <span class="sb-badge">block</span>'
            + '  </div>'
            + '  <div class="sb-editor-block-preview">' + preview + '</div>'
            + '  <div class="sb-editor-block-actions">'
            + '    <button type="button" class="sb-btn sb-btn-primary js-edit-block" data-id="' + id + '">Редактировать</button>'
            + '    <button type="button" class="sb-btn sb-btn-light js-duplicate-block" data-id="' + id + '">Дублировать</button>'
            + '    <button type="button" class="sb-btn sb-btn-gray js-move-block-up" data-id="' + id + '"' + (index === 0 ? ' disabled' : '') + '>↑</button>'
            + '    <button type="button" class="sb-btn sb-btn-gray js-move-block-down" data-id="' + id + '"' + (index === state.blocks.length - 1 ? ' disabled' : '') + '>↓</button>'
            + '    <button type="button" class="sb-btn sb-btn-danger js-delete-block" data-id="' + id + '">Удалить</button>'
            + '  </div>'
            + '</div>';
    }

    function buildPreviewText(block) {
        var type = String(block.type || '');
        var content = block.content || {};

        if (type === 'text') {
            return String(content.html || '').replace(/<[^>]*>/g, ' ').trim() || '[text]';
        }
        if (type === 'heading') {
            return String(content.text || '') || '[heading]';
        }
        if (type === 'button') {
            return (content.text || '[button]') + ' → ' + (content.href || '');
        }
        if (type === 'html') {
            return String(content.html || '').replace(/<[^>]*>/g, ' ').trim() || '[html]';
        }
        if (type === 'spacer') {
            return 'height: ' + String(content.height || 0);
        }

        return JSON.stringify(content);
    }

    function addBlock(type) {
        if (!state.currentPageId) {
            alert('Сначала выберите страницу');
            return;
        }

        api('block.create', {
            pageId: state.currentPageId,
            type: type
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось создать блок');
                return;
            }

            loadBlocks();
        });
    }

    function editBlock(blockId) {
        var block = findBlock(blockId);
        if (!block) {
            alert('Блок не найден');
            return;
        }

        state.currentBlockId = Number(blockId || 0);
        renderBlocks();

        document.getElementById('blockEditorEmpty').classList.add('sb-hidden');
        document.getElementById('blockEditorForm').classList.remove('sb-hidden');
        document.getElementById('editBlockType').value = block.type || '';
        document.getElementById('editBlockContentText').value = JSON.stringify(block.content || {}, null, 2);
        document.getElementById('editBlockPropsText').value = JSON.stringify(block.props || {}, null, 2);
    }

    function clearBlockEditor() {
        state.currentBlockId = 0;
        renderBlocks();
        document.getElementById('blockEditorEmpty').classList.remove('sb-hidden');
        document.getElementById('blockEditorForm').classList.add('sb-hidden');
        document.getElementById('editBlockType').value = '';
        document.getElementById('editBlockContentText').value = '';
        document.getElementById('editBlockPropsText').value = '{}';
    }

    function saveBlock() {
        if (!state.currentBlockId) {
            return;
        }

        var contentText = document.getElementById('editBlockContentText').value || '{}';
        var propsText = document.getElementById('editBlockPropsText').value || '{}';

        try {
            JSON.parse(contentText);
        } catch (e) {
            alert('Content должен быть валидным JSON');
            return;
        }

        try {
            JSON.parse(propsText);
        } catch (e) {
            alert('Props должен быть валидным JSON');
            return;
        }

        api('block.update', {
            id: state.currentBlockId,
            content: contentText,
            props: propsText
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось сохранить блок');
                return;
            }

            loadBlocks();
        });
    }

    function deleteBlock(blockId) {
        if (!confirm('Удалить блок #' + blockId + '?')) {
            return;
        }

        api('block.delete', { id: blockId }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось удалить блок');
                return;
            }

            if (Number(state.currentBlockId || 0) === Number(blockId || 0)) {
                clearBlockEditor();
            }

            loadBlocks();
        });
    }

    function duplicateBlock(blockId) {
        api('block.duplicate', { id: blockId }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось дублировать блок');
                return;
            }

            loadBlocks();
        });
    }

    function moveBlock(blockId, dir) {
        api('block.move', {
            id: blockId,
            dir: dir
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось переместить блок');
                return;
            }

            loadBlocks();
        });
    }

    function formatCurrentJson() {
        var contentEl = document.getElementById('editBlockContentText');
        var propsEl = document.getElementById('editBlockPropsText');

        try {
            contentEl.value = JSON.stringify(JSON.parse(contentEl.value || '{}'), null, 2);
        } catch (e) {}

        try {
            propsEl.value = JSON.stringify(JSON.parse(propsEl.value || '{}'), null, 2);
        } catch (e) {}
    }

    function resetProps() {
        document.getElementById('editBlockPropsText').value = '{}';
    }

    document.getElementById('createPageBtn').addEventListener('click', createPage);
    document.getElementById('savePageParentBtn').addEventListener('click', saveCurrentPageParent);
    document.getElementById('reloadBlocksBtn').addEventListener('click', loadBlocks);
    document.getElementById('saveBlockBtn').addEventListener('click', saveBlock);
    document.getElementById('deleteBlockBtn').addEventListener('click', function () {
        if (state.currentBlockId) {
            deleteBlock(state.currentBlockId);
        }
    });
    document.getElementById('formatJsonBtn').addEventListener('click', formatCurrentJson);
    document.getElementById('resetPropsBtn').addEventListener('click', resetProps);

    document.addEventListener('click', function (e) {
        var pageCard = e.target.closest('.js-page-card');
        if (pageCard && !e.target.closest('button, a')) {
            openPage(parseInt(pageCard.getAttribute('data-id'), 10) || 0);
            return;
        }

        var addBlockBtn = e.target.closest('.js-add-block');
        if (addBlockBtn) {
            addBlock(addBlockBtn.getAttribute('data-type') || 'text');
            return;
        }

        var openPageBtn = e.target.closest('.js-open-page');
        if (openPageBtn) {
            openPage(parseInt(openPageBtn.getAttribute('data-id'), 10) || 0);
            return;
        }

        var renamePageBtn = e.target.closest('.js-rename-page');
        if (renamePageBtn) {
            renamePage(parseInt(renamePageBtn.getAttribute('data-id'), 10) || 0);
            return;
        }

        var duplicatePageBtn = e.target.closest('.js-duplicate-page');
        if (duplicatePageBtn) {
            duplicatePage(parseInt(duplicatePageBtn.getAttribute('data-id'), 10) || 0);
            return;
        }

        var toggleStatusBtn = e.target.closest('.js-toggle-status');
        if (toggleStatusBtn) {
            togglePageStatus(
                parseInt(toggleStatusBtn.getAttribute('data-id'), 10) || 0,
                toggleStatusBtn.getAttribute('data-status') || 'draft'
            );
            return;
        }

        var movePageUpBtn = e.target.closest('.js-move-page-up');
        if (movePageUpBtn) {
            movePage(parseInt(movePageUpBtn.getAttribute('data-id'), 10) || 0, 'up');
            return;
        }

        var movePageDownBtn = e.target.closest('.js-move-page-down');
        if (movePageDownBtn) {
            movePage(parseInt(movePageDownBtn.getAttribute('data-id'), 10) || 0, 'down');
            return;
        }

        var deletePageBtn = e.target.closest('.js-delete-page');
        if (deletePageBtn) {
            deletePage(parseInt(deletePageBtn.getAttribute('data-id'), 10) || 0);
            return;
        }

        var editBlockBtn = e.target.closest('.js-edit-block');
        if (editBlockBtn) {
            editBlock(parseInt(editBlockBtn.getAttribute('data-id'), 10) || 0);
            return;
        }

        var deleteBlockBtn = e.target.closest('.js-delete-block');
        if (deleteBlockBtn) {
            deleteBlock(parseInt(deleteBlockBtn.getAttribute('data-id'), 10) || 0);
            return;
        }

        var duplicateBlockBtn = e.target.closest('.js-duplicate-block');
        if (duplicateBlockBtn) {
            duplicateBlock(parseInt(duplicateBlockBtn.getAttribute('data-id'), 10) || 0);
            return;
        }

        var moveBlockUpBtn = e.target.closest('.js-move-block-up');
        if (moveBlockUpBtn) {
            moveBlock(parseInt(moveBlockUpBtn.getAttribute('data-id'), 10) || 0, 'up');
            return;
        }

        var moveBlockDownBtn = e.target.closest('.js-move-block-down');
        if (moveBlockDownBtn) {
            moveBlock(parseInt(moveBlockDownBtn.getAttribute('data-id'), 10) || 0, 'down');
            return;
        }
    });

    window.onerror = function (message, source, lineno, colno, error) {
        print({
            jsError: true,
            message: message,
            source: source,
            line: lineno,
            column: colno,
            stack: error && error.stack ? error.stack : null
        });
    };

    loadSite(function () {
        loadPages(function () {
            if (state.currentPageId) {
                loadBlocks();
            } else {
                updateStats();
            }
        });
    });
})();
</script>
</body>
</html>


---

4. Что теперь появится

После этого у тебя будет:

выбор родителя при создании страницы

редактирование parentId у текущей страницы

дерево страниц по parentId

вложенность через отступы

страница с дочерними страницами будет визуально помечаться как section

удаление страницы будет удалять и дочерние страницы



---

5. Что проверить после замены

Проверь по шагам:

Создание

1. создай страницу без родителя


2. создай вторую страницу с родителем = первая


3. создай третью страницу с родителем = вторая



Ожидание:

слева появится дерево с вложенностью


Переназначение

1. выбери страницу


2. справа поменяй Родитель текущей страницы


3. нажми Сохранить parent



Ожидание:

страница переедет в другое место дерева


Section

Если у страницы появились дочерние страницы:

возле заголовка карточки появится метка section


Удаление

Если удалить родительскую страницу:

удалятся и её дочерние страницы

удалятся и блоки этих страниц



---

6. Что дальше

После этого следующим шагом уже логично делать:

breadcrumbs в public.php

левую навигацию по разделу

иерархию страниц в menu.php


Если хочешь, следующим сообщением я дам пакет на breadcrumbs + section navigation в public.php.