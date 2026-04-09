Отлично. Значит menu-ветка уже живая.

Дальше логично вернуть block.*, потому что без неё не поднять редактор страниц.

Что делаем сейчас

1. подключаем block.* в роутер


2. проверяем block.php


3. потом уже собираем editor.php




---

1. Обнови /local/sitebuilder/api/index.php

Замени файл целиком на этот вариант:

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

if (
    $action === 'menu.list' ||
    $action === 'menu.create' ||
    $action === 'menu.update' ||
    $action === 'menu.delete' ||
    $action === 'menu.setTop' ||
    $action === 'menu.item.add' ||
    $action === 'menu.item.update' ||
    $action === 'menu.item.delete' ||
    $action === 'menu.item.move'
) {
    require __DIR__ . '/handlers/menu.php';
    exit;
}

if (
    $action === 'block.list' ||
    $action === 'block.create' ||
    $action === 'block.update' ||
    $action === 'block.delete' ||
    $action === 'block.duplicate' ||
    $action === 'block.move' ||
    $action === 'block.reorder'
) {
    require __DIR__ . '/handlers/block.php';
    exit;
}

sb_json_error('UNKNOWN_ACTION', 400, [
    'action' => $action,
    'file' => __FILE__,
]);


---

2. Проверь /local/sitebuilder/api/handlers/block.php

Обязательно убедись, что файл начинается с:

<?php

Если не уверен — замени целиком на этот вариант.

/local/sitebuilder/api/handlers/block.php

<?php

global $USER;

if ($action === 'block.list') {
    $pageId = (int)($_POST['pageId'] ?? 0);
    if ($pageId <= 0) {
        sb_json_error('PAGE_ID_REQUIRED', 422);
    }

    $page = sb_find_page($pageId);
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_viewer($siteId);

    $blocks = sb_blocks_for_page($pageId);
    $blocks = array_map('sb_normalize_block_record', $blocks);

    sb_json_ok([
        'blocks' => $blocks,
        'handler' => 'block',
        'file' => __FILE__,
    ]);
}

if ($action === 'block.create') {
    $pageId = (int)($_POST['pageId'] ?? 0);
    $type = trim((string)($_POST['type'] ?? 'text'));

    if ($pageId <= 0) {
        sb_json_error('PAGE_ID_REQUIRED', 422);
    }
    if ($type === '') {
        sb_json_error('TYPE_REQUIRED', 422);
    }

    $page = sb_find_page($pageId);
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $blocks = sb_read_blocks();

    $block = [
        'id' => sb_next_block_id($blocks),
        'pageId' => $pageId,
        'type' => $type,
        'sort' => sb_next_block_sort($pageId, $blocks),
        'content' => sb_default_block_content($type),
        'props' => [],
        'createdBy' => (int)$USER->GetID(),
        'createdAt' => date('c'),
        'updatedAt' => date('c'),
        'updatedBy' => (int)$USER->GetID(),
    ];

    $blocks[] = $block;
    sb_write_blocks($blocks);

    sb_json_ok([
        'block' => sb_normalize_block_record($block),
        'handler' => 'block',
        'file' => __FILE__,
    ]);
}

if ($action === 'block.update') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }

    $block = sb_find_block($id);
    if (!$block) {
        sb_json_error('BLOCK_NOT_FOUND', 404);
    }

    $page = sb_find_page((int)($block['pageId'] ?? 0));
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $contentRaw = $_POST['content'] ?? null;
    $propsRaw = $_POST['props'] ?? null;
    $typeRaw = $_POST['type'] ?? null;

    $newContent = null;
    $newProps = null;
    $newType = null;

    if ($contentRaw !== null) {
        if (is_array($contentRaw)) {
            $newContent = $contentRaw;
        } else {
            $decoded = json_decode((string)$contentRaw, true);
            if (!is_array($decoded)) {
                sb_json_error('BAD_CONTENT_JSON', 422);
            }
            $newContent = $decoded;
        }
    }

    if ($propsRaw !== null) {
        if (is_array($propsRaw)) {
            $newProps = $propsRaw;
        } else {
            $decoded = json_decode((string)$propsRaw, true);
            if (!is_array($decoded)) {
                sb_json_error('BAD_PROPS_JSON', 422);
            }
            $newProps = $decoded;
        }
    }

    if ($typeRaw !== null) {
        $newType = trim((string)$typeRaw);
        if ($newType === '') {
            sb_json_error('TYPE_REQUIRED', 422);
        }
    }

    $blocks = sb_read_blocks();
    $updated = null;

    foreach ($blocks as &$b) {
        if ((int)($b['id'] ?? 0) === $id) {
            if ($newType !== null) {
                $b['type'] = $newType;
            }
            if ($newContent !== null) {
                $b['content'] = $newContent;
            }
            if ($newProps !== null) {
                $b['props'] = $newProps;
            }

            $b['updatedAt'] = date('c');
            $b['updatedBy'] = (int)$USER->GetID();

            $updated = $b;
            break;
        }
    }
    unset($b);

    if (!$updated) {
        sb_json_error('BLOCK_NOT_FOUND', 404);
    }

    sb_write_blocks($blocks);

    sb_json_ok([
        'block' => sb_normalize_block_record($updated),
        'handler' => 'block',
        'file' => __FILE__,
    ]);
}

if ($action === 'block.delete') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }

    $block = sb_find_block($id);
    if (!$block) {
        sb_json_error('BLOCK_NOT_FOUND', 404);
    }

    $page = sb_find_page((int)($block['pageId'] ?? 0));
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $blocks = sb_read_blocks();
    $before = count($blocks);

    $blocks = array_values(array_filter($blocks, static function ($b) use ($id) {
        return (int)($b['id'] ?? 0) !== $id;
    }));

    if (count($blocks) === $before) {
        sb_json_error('BLOCK_NOT_FOUND', 404);
    }

    sb_write_blocks($blocks);

    sb_json_ok([
        'handler' => 'block',
        'file' => __FILE__,
    ]);
}

if ($action === 'block.duplicate') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }

    $src = sb_find_block($id);
    if (!$src) {
        sb_json_error('BLOCK_NOT_FOUND', 404);
    }

    $pageId = (int)($src['pageId'] ?? 0);
    $page = sb_find_page($pageId);
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $blocks = sb_read_blocks();
    $srcSort = (int)($src['sort'] ?? 500);

    foreach ($blocks as &$b) {
        if (
            (int)($b['pageId'] ?? 0) === $pageId &&
            (int)($b['sort'] ?? 0) > $srcSort
        ) {
            $b['sort'] = (int)($b['sort'] ?? 0) + 10;
            $b['updatedAt'] = date('c');
            $b['updatedBy'] = (int)$USER->GetID();
        }
    }
    unset($b);

    $copy = $src;
    $copy['id'] = sb_next_block_id($blocks);
    $copy['sort'] = $srcSort + 10;
    $copy['createdBy'] = (int)$USER->GetID();
    $copy['createdAt'] = date('c');
    $copy['updatedAt'] = date('c');
    $copy['updatedBy'] = (int)$USER->GetID();

    $blocks[] = $copy;
    sb_write_blocks($blocks);

    sb_json_ok([
        'block' => sb_normalize_block_record($copy),
        'handler' => 'block',
        'file' => __FILE__,
    ]);
}

if ($action === 'block.move') {
    $id = (int)($_POST['id'] ?? 0);
    $dir = trim((string)($_POST['dir'] ?? ''));

    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }
    if ($dir !== 'up' && $dir !== 'down') {
        sb_json_error('DIR_REQUIRED', 422);
    }

    $block = sb_find_block($id);
    if (!$block) {
        sb_json_error('BLOCK_NOT_FOUND', 404);
    }

    $pageId = (int)($block['pageId'] ?? 0);
    $page = sb_find_page($pageId);
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $blocks = sb_read_blocks();

    $siblings = array_values(array_filter($blocks, static function ($b) use ($pageId) {
        return (int)($b['pageId'] ?? 0) === $pageId;
    }));

    usort($siblings, static function ($a, $b) {
        $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);
        if ($sortCmp !== 0) {
            return $sortCmp;
        }
        return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
    });

    $pos = null;
    for ($i = 0, $cnt = count($siblings); $i < $cnt; $i++) {
        if ((int)($siblings[$i]['id'] ?? 0) === $id) {
            $pos = $i;
            break;
        }
    }

    if ($pos === null) {
        sb_json_ok([
            'handler' => 'block',
            'file' => __FILE__,
        ]);
    }

    if ($dir === 'up' && $pos === 0) {
        sb_json_ok([
            'handler' => 'block',
            'file' => __FILE__,
        ]);
    }

    if ($dir === 'down' && $pos === count($siblings) - 1) {
        sb_json_ok([
            'handler' => 'block',
            'file' => __FILE__,
        ]);
    }

    $swapPos = ($dir === 'up') ? $pos - 1 : $pos + 1;

    $idA = (int)$siblings[$pos]['id'];
    $idB = (int)$siblings[$swapPos]['id'];
    $sortA = (int)($siblings[$pos]['sort'] ?? 500);
    $sortB = (int)($siblings[$swapPos]['sort'] ?? 500);

    foreach ($blocks as &$b) {
        $bid = (int)($b['id'] ?? 0);

        if ($bid === $idA) {
            $b['sort'] = $sortB;
            $b['updatedAt'] = date('c');
            $b['updatedBy'] = (int)$USER->GetID();
        }

        if ($bid === $idB) {
            $b['sort'] = $sortA;
            $b['updatedAt'] = date('c');
            $b['updatedBy'] = (int)$USER->GetID();
        }
    }
    unset($b);

    sb_write_blocks($blocks);

    sb_json_ok([
        'handler' => 'block',
        'file' => __FILE__,
    ]);
}

if ($action === 'block.reorder') {
    $pageId = (int)($_POST['pageId'] ?? 0);
    if ($pageId <= 0) {
        sb_json_error('PAGE_ID_REQUIRED', 422);
    }

    $page = sb_find_page($pageId);
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $orderRaw = $_POST['order'] ?? null;
    if ($orderRaw === null) {
        sb_json_error('ORDER_REQUIRED', 422);
    }

    if (is_array($orderRaw)) {
        $order = $orderRaw;
    } else {
        $order = json_decode((string)$orderRaw, true);
        if (!is_array($order)) {
            sb_json_error('BAD_ORDER_JSON', 422);
        }
    }

    $orderIds = [];
    foreach ($order as $item) {
        $bid = (int)$item;
        if ($bid > 0) {
            $orderIds[] = $bid;
        }
    }

    $pageBlocks = sb_blocks_for_page($pageId);
    $pageBlockIds = [];
    foreach ($pageBlocks as $b) {
        $pageBlockIds[(int)($b['id'] ?? 0)] = true;
    }

    foreach ($orderIds as $bid) {
        if (!isset($pageBlockIds[$bid])) {
            sb_json_error('BLOCK_NOT_IN_PAGE', 422, ['blockId' => $bid]);
        }
    }

    $missing = array_diff(array_keys($pageBlockIds), $orderIds);
    if (!empty($missing)) {
        foreach ($missing as $bid) {
            $orderIds[] = (int)$bid;
        }
    }

    $sortMap = [];
    $sort = 10;
    foreach ($orderIds as $bid) {
        $sortMap[(int)$bid] = $sort;
        $sort += 10;
    }

    $blocks = sb_read_blocks();
    foreach ($blocks as &$b) {
        $bid = (int)($b['id'] ?? 0);
        if ((int)($b['pageId'] ?? 0) === $pageId && isset($sortMap[$bid])) {
            $b['sort'] = $sortMap[$bid];
            $b['updatedAt'] = date('c');
            $b['updatedBy'] = (int)$USER->GetID();
        }
    }
    unset($b);

    sb_write_blocks($blocks);

    sb_json_ok([
        'blocks' => array_map('sb_normalize_block_record', sb_blocks_for_page($pageId)),
        'handler' => 'block',
        'file' => __FILE__,
    ]);
}

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'block',
    'action' => $action,
    'file' => __FILE__,
]);

3. Сбрось opcache

Открой:

/local/sitebuilder/opcache_reset.php

4. Что проверить сейчас

Пока без editor.php можно проверить так:

в test_api.php или через временные вызовы из консоли страницы

block.list

block.create

block.update

block.delete


Но проще дальше уже не мучиться ручными вызовами, а собрать страницу редактора.

5. Следующий шаг

Теперь, когда site, page, menu, block уже подключены, следующий правильный шаг — сделать рабочий editor.php:

список страниц

создание страниц

список блоков текущей страницы

добавление/редактирование/удаление блоков


Напиши: давай editor.php