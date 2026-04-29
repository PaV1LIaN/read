Даю следующие 3 файла.

## `/local/sitebuilder/api/handlers/menu.php`

```php
<?php

global $USER;

if ($action === 'menu.list') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    sb_require_viewer($siteId);

    $menus = array_values(array_filter(sb_read_menus(), static function ($m) use ($siteId) {
        return (int)($m['siteId'] ?? 0) === $siteId;
    }));

    $menus = array_map('sb_normalize_menu_record', $menus);

    usort($menus, static function ($a, $b) {
        return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
    });

    $site = sb_find_site($siteId);
    $topMenuId = $site ? (int)($site['topMenuId'] ?? 0) : 0;

    sb_json_ok([
        'menus' => $menus,
        'topMenuId' => $topMenuId,
    ]);
}

if ($action === 'menu.create') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $name = trim((string)($_POST['name'] ?? ''));

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }
    if ($name === '') {
        sb_json_error('NAME_REQUIRED', 422);
    }

    sb_require_content_manager($siteId);

    $menus = sb_read_menus();

    $menu = [
        'id' => sb_next_menu_id($menus),
        'siteId' => $siteId,
        'name' => $name,
        'items' => [],
        'createdBy' => (int)$USER->GetID(),
        'createdAt' => date('c'),
        'updatedAt' => date('c'),
        'updatedBy' => (int)$USER->GetID(),
    ];

    $menus[] = $menu;
    sb_write_menus($menus);

    sb_json_ok([
        'menu' => sb_normalize_menu_record($menu),
    ]);
}

if ($action === 'menu.update') {
    $id = (int)($_POST['id'] ?? 0);
    $name = trim((string)($_POST['name'] ?? ''));

    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }
    if ($name === '') {
        sb_json_error('NAME_REQUIRED', 422);
    }

    $menu = sb_find_menu($id);
    if (!$menu) {
        sb_json_error('MENU_NOT_FOUND', 404);
    }

    $siteId = (int)($menu['siteId'] ?? 0);
    sb_require_content_manager($siteId);

    $menus = sb_read_menus();
    $updated = null;

    foreach ($menus as &$m) {
        if ((int)($m['id'] ?? 0) === $id) {
            $m['name'] = $name;
            $m['updatedAt'] = date('c');
            $m['updatedBy'] = (int)$USER->GetID();
            $updated = $m;
            break;
        }
    }
    unset($m);

    if (!$updated) {
        sb_json_error('MENU_NOT_FOUND', 404);
    }

    sb_write_menus($menus);

    sb_json_ok([
        'menu' => sb_normalize_menu_record($updated),
    ]);
}

if ($action === 'menu.delete') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }

    $menu = sb_find_menu($id);
    if (!$menu) {
        sb_json_error('MENU_NOT_FOUND', 404);
    }

    $siteId = (int)($menu['siteId'] ?? 0);
    sb_require_content_manager($siteId);

    $menus = sb_read_menus();
    $before = count($menus);

    $menus = array_values(array_filter($menus, static function ($m) use ($id) {
        return (int)($m['id'] ?? 0) !== $id;
    }));

    if (count($menus) === $before) {
        sb_json_error('MENU_NOT_FOUND', 404);
    }

    sb_write_menus($menus);

    $sites = sb_read_sites();
    $sitesChanged = false;

    foreach ($sites as &$s) {
        if ((int)($s['id'] ?? 0) === $siteId && (int)($s['topMenuId'] ?? 0) === $id) {
            $s['topMenuId'] = 0;
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

if ($action === 'menu.setTop') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $menuId = (int)($_POST['menuId'] ?? 0);

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    sb_require_content_manager($siteId);

    if ($menuId > 0) {
        $menu = sb_find_menu($menuId);
        if (!$menu || (int)($menu['siteId'] ?? 0) !== $siteId) {
            sb_json_error('MENU_NOT_IN_SITE', 422);
        }
    }

    $sites = sb_read_sites();
    $found = false;

    foreach ($sites as &$s) {
        if ((int)($s['id'] ?? 0) === $siteId) {
            $s['topMenuId'] = $menuId;
            $s['updatedAt'] = date('c');
            $s['updatedBy'] = (int)$USER->GetID();
            $found = true;
            break;
        }
    }
    unset($s);

    if (!$found) {
        sb_json_error('SITE_NOT_FOUND', 404);
    }

    sb_write_sites($sites);
    sb_json_ok();
}

if ($action === 'menu.item.add') {
    $menuId = (int)($_POST['menuId'] ?? 0);
    $title = trim((string)($_POST['title'] ?? ''));
    $type = trim((string)($_POST['type'] ?? 'page'));
    $pageId = (int)($_POST['pageId'] ?? 0);
    $url = trim((string)($_POST['url'] ?? ''));
    $target = trim((string)($_POST['target'] ?? '_self'));

    if ($menuId <= 0) {
        sb_json_error('MENU_ID_REQUIRED', 422);
    }
    if ($title === '') {
        sb_json_error('TITLE_REQUIRED', 422);
    }
    if (!in_array($type, ['page', 'url'], true)) {
        sb_json_error('BAD_ITEM_TYPE', 422);
    }
    if (!in_array($target, ['_self', '_blank'], true)) {
        $target = '_self';
    }

    $menu = sb_find_menu($menuId);
    if (!$menu) {
        sb_json_error('MENU_NOT_FOUND', 404);
    }

    $siteId = (int)($menu['siteId'] ?? 0);
    sb_require_content_manager($siteId);

    if ($type === 'page') {
        if ($pageId <= 0) {
            sb_json_error('PAGE_ID_REQUIRED', 422);
        }

        $page = sb_find_page($pageId);
        if (!$page || (int)($page['siteId'] ?? 0) !== $siteId) {
            sb_json_error('PAGE_NOT_IN_SITE', 422);
        }

        $url = '';
    } else {
        $pageId = 0;
    }

    $menus = sb_read_menus();
    $updatedMenu = null;

    foreach ($menus as &$m) {
        if ((int)($m['id'] ?? 0) !== $menuId) {
            continue;
        }

        if (!isset($m['items']) || !is_array($m['items'])) {
            $m['items'] = [];
        }

        $item = [
            'id' => sb_next_menu_item_id($m['items']),
            'title' => $title,
            'type' => $type,
            'pageId' => $pageId,
            'url' => $url,
            'target' => $target,
            'sort' => sb_menu_next_item_sort($m['items']),
        ];

        $m['items'][] = $item;
        $m['updatedAt'] = date('c');
        $m['updatedBy'] = (int)$USER->GetID();
        $updatedMenu = $m;
        break;
    }
    unset($m);

    if (!$updatedMenu) {
        sb_json_error('MENU_NOT_FOUND', 404);
    }

    sb_write_menus($menus);

    sb_json_ok([
        'menu' => sb_normalize_menu_record($updatedMenu),
    ]);
}

if ($action === 'menu.item.update') {
    $menuId = (int)($_POST['menuId'] ?? 0);
    $itemId = (int)($_POST['itemId'] ?? 0);
    $title = trim((string)($_POST['title'] ?? ''));
    $type = trim((string)($_POST['type'] ?? 'page'));
    $pageId = (int)($_POST['pageId'] ?? 0);
    $url = trim((string)($_POST['url'] ?? ''));
    $target = trim((string)($_POST['target'] ?? '_self'));

    if ($menuId <= 0) {
        sb_json_error('MENU_ID_REQUIRED', 422);
    }
    if ($itemId <= 0) {
        sb_json_error('ITEM_ID_REQUIRED', 422);
    }
    if ($title === '') {
        sb_json_error('TITLE_REQUIRED', 422);
    }
    if (!in_array($type, ['page', 'url'], true)) {
        sb_json_error('BAD_ITEM_TYPE', 422);
    }
    if (!in_array($target, ['_self', '_blank'], true)) {
        $target = '_self';
    }

    $menu = sb_find_menu($menuId);
    if (!$menu) {
        sb_json_error('MENU_NOT_FOUND', 404);
    }

    $siteId = (int)($menu['siteId'] ?? 0);
    sb_require_content_manager($siteId);

    if ($type === 'page') {
        if ($pageId <= 0) {
            sb_json_error('PAGE_ID_REQUIRED', 422);
        }

        $page = sb_find_page($pageId);
        if (!$page || (int)($page['siteId'] ?? 0) !== $siteId) {
            sb_json_error('PAGE_NOT_IN_SITE', 422);
        }

        $url = '';
    } else {
        $pageId = 0;
    }

    $menus = sb_read_menus();
    $updatedMenu = null;
    $foundItem = false;

    foreach ($menus as &$m) {
        if ((int)($m['id'] ?? 0) !== $menuId) {
            continue;
        }

        if (!isset($m['items']) || !is_array($m['items'])) {
            $m['items'] = [];
        }

        foreach ($m['items'] as &$item) {
            if ((int)($item['id'] ?? 0) !== $itemId) {
                continue;
            }

            $item['title'] = $title;
            $item['type'] = $type;
            $item['pageId'] = $pageId;
            $item['url'] = $url;
            $item['target'] = $target;

            $foundItem = true;
            break;
        }
        unset($item);

        if ($foundItem) {
            $m['updatedAt'] = date('c');
            $m['updatedBy'] = (int)$USER->GetID();
            $updatedMenu = $m;
            break;
        }
    }
    unset($m);

    if (!$foundItem || !$updatedMenu) {
        sb_json_error('ITEM_NOT_FOUND', 404);
    }

    sb_write_menus($menus);

    sb_json_ok([
        'menu' => sb_normalize_menu_record($updatedMenu),
    ]);
}

if ($action === 'menu.item.delete') {
    $menuId = (int)($_POST['menuId'] ?? 0);
    $itemId = (int)($_POST['itemId'] ?? 0);

    if ($menuId <= 0) {
        sb_json_error('MENU_ID_REQUIRED', 422);
    }
    if ($itemId <= 0) {
        sb_json_error('ITEM_ID_REQUIRED', 422);
    }

    $menu = sb_find_menu($menuId);
    if (!$menu) {
        sb_json_error('MENU_NOT_FOUND', 404);
    }

    $siteId = (int)($menu['siteId'] ?? 0);
    sb_require_content_manager($siteId);

    $menus = sb_read_menus();
    $updatedMenu = null;
    $foundItem = false;

    foreach ($menus as &$m) {
        if ((int)($m['id'] ?? 0) !== $menuId) {
            continue;
        }

        if (!isset($m['items']) || !is_array($m['items'])) {
            $m['items'] = [];
        }

        $before = count($m['items']);
        $m['items'] = array_values(array_filter($m['items'], static function ($item) use ($itemId) {
            return (int)($item['id'] ?? 0) !== $itemId;
        }));

        if (count($m['items']) !== $before) {
            $foundItem = true;
            $m['updatedAt'] = date('c');
            $m['updatedBy'] = (int)$USER->GetID();
            $updatedMenu = $m;
        }

        break;
    }
    unset($m);

    if (!$foundItem) {
        sb_json_error('ITEM_NOT_FOUND', 404);
    }

    sb_write_menus($menus);

    sb_json_ok([
        'menu' => sb_normalize_menu_record($updatedMenu),
    ]);
}

if ($action === 'menu.item.move') {
    $menuId = (int)($_POST['menuId'] ?? 0);
    $itemId = (int)($_POST['itemId'] ?? 0);
    $dir = trim((string)($_POST['dir'] ?? ''));

    if ($menuId <= 0) {
        sb_json_error('MENU_ID_REQUIRED', 422);
    }
    if ($itemId <= 0) {
        sb_json_error('ITEM_ID_REQUIRED', 422);
    }
    if ($dir !== 'up' && $dir !== 'down') {
        sb_json_error('DIR_REQUIRED', 422);
    }

    $menu = sb_find_menu($menuId);
    if (!$menu) {
        sb_json_error('MENU_NOT_FOUND', 404);
    }

    $siteId = (int)($menu['siteId'] ?? 0);
    sb_require_content_manager($siteId);

    $menus = sb_read_menus();
    $updatedMenu = null;
    $foundMenu = false;

    foreach ($menus as &$m) {
        if ((int)($m['id'] ?? 0) !== $menuId) {
            continue;
        }

        $foundMenu = true;

        if (!isset($m['items']) || !is_array($m['items'])) {
            $m['items'] = [];
        }

        usort($m['items'], static function ($a, $b) {
            $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);
            if ($sortCmp !== 0) {
                return $sortCmp;
            }
            return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
        });

        $pos = null;
        for ($i = 0, $cnt = count($m['items']); $i < $cnt; $i++) {
            if ((int)($m['items'][$i]['id'] ?? 0) === $itemId) {
                $pos = $i;
                break;
            }
        }

        if ($pos === null) {
            sb_json_error('ITEM_NOT_FOUND', 404);
        }

        if ($dir === 'up' && $pos === 0) {
            $updatedMenu = $m;
            break;
        }

        if ($dir === 'down' && $pos === count($m['items']) - 1) {
            $updatedMenu = $m;
            break;
        }

        $swapPos = ($dir === 'up') ? $pos - 1 : $pos + 1;

        $sortA = (int)($m['items'][$pos]['sort'] ?? 500);
        $sortB = (int)($m['items'][$swapPos]['sort'] ?? 500);

        $m['items'][$pos]['sort'] = $sortB;
        $m['items'][$swapPos]['sort'] = $sortA;
        $m['updatedAt'] = date('c');
        $m['updatedBy'] = (int)$USER->GetID();

        $updatedMenu = $m;
        break;
    }
    unset($m);

    if (!$foundMenu) {
        sb_json_error('MENU_NOT_FOUND', 404);
    }

    sb_write_menus($menus);

    sb_json_ok([
        'menu' => sb_normalize_menu_record($updatedMenu),
    ]);
}

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'menu',
    'action' => $action,
]);
```

## `/local/sitebuilder/api/handlers/layout.php`

```php
<?php

global $USER;

if ($action === 'layout.get') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    sb_require_viewer($siteId);

    $layout = sb_layout_ensure_record($siteId);

    sb_json_ok([
        'layout' => $layout,
        'handler' => 'layout',
        'file' => __FILE__,
    ]);
}

if ($action === 'layout.updateSettings') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    sb_require_content_manager($siteId);

    $settingsRaw = $_POST['settings'] ?? null;
    if ($settingsRaw === null) {
        sb_json_error('SETTINGS_REQUIRED', 422);
    }

    if (is_array($settingsRaw)) {
        $settings = $settingsRaw;
    } else {
        $settings = json_decode((string)$settingsRaw, true);
        if (!is_array($settings)) {
            sb_json_error('BAD_SETTINGS_JSON', 422);
        }
    }

    $allowedKeys = [
        'showHeader',
        'showFooter',
        'showLeft',
        'showRight',
        'leftWidth',
        'rightWidth',
        'leftMode',
    ];

    $filtered = [];
    foreach ($allowedKeys as $key) {
        if (array_key_exists($key, $settings)) {
            $filtered[$key] = $settings[$key];
        }
    }

    if (isset($filtered['showHeader'])) {
        $filtered['showHeader'] = (bool)$filtered['showHeader'];
    }
    if (isset($filtered['showFooter'])) {
        $filtered['showFooter'] = (bool)$filtered['showFooter'];
    }
    if (isset($filtered['showLeft'])) {
        $filtered['showLeft'] = (bool)$filtered['showLeft'];
    }
    if (isset($filtered['showRight'])) {
        $filtered['showRight'] = (bool)$filtered['showRight'];
    }
    if (isset($filtered['leftWidth'])) {
        $filtered['leftWidth'] = max(120, min(800, (int)$filtered['leftWidth']));
    }
    if (isset($filtered['rightWidth'])) {
        $filtered['rightWidth'] = max(120, min(800, (int)$filtered['rightWidth']));
    }
    if (isset($filtered['leftMode'])) {
        $filtered['leftMode'] = in_array((string)$filtered['leftMode'], ['blocks', 'menu'], true)
            ? (string)$filtered['leftMode']
            : 'blocks';
    }

    $layouts = sb_read_layouts();
    $updated = null;
    $found = false;

    foreach ($layouts as &$layout) {
        if ((int)($layout['siteId'] ?? 0) !== $siteId) {
            continue;
        }

        $layout = sb_normalize_layout_record($layout);
        $layout['settings'] = array_merge($layout['settings'], $filtered);
        $layout['updatedAt'] = date('c');
        $layout['updatedBy'] = (int)$USER->GetID();

        $updated = $layout;
        $found = true;
        break;
    }
    unset($layout);

    if (!$found) {
        $updated = sb_layout_default_record($siteId);
        $updated['settings'] = array_merge($updated['settings'], $filtered);
        $updated['createdAt'] = date('c');
        $updated['createdBy'] = (int)$USER->GetID();
        $updated['updatedAt'] = date('c');
        $updated['updatedBy'] = (int)$USER->GetID();
        $layouts[] = $updated;
    }

    sb_write_layouts($layouts);

    sb_json_ok([
        'layout' => sb_normalize_layout_record($updated),
        'handler' => 'layout',
        'file' => __FILE__,
    ]);
}

if ($action === 'layout.block.list') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $zone = trim((string)($_POST['zone'] ?? ''));

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }
    if (!sb_layout_valid_zone($zone)) {
        sb_json_error('BAD_ZONE', 422);
    }

    sb_require_viewer($siteId);

    $layout = sb_layout_ensure_record($siteId);

    sb_json_ok([
        'blocks' => array_values($layout['zones'][$zone] ?? []),
        'zone' => $zone,
        'handler' => 'layout',
        'file' => __FILE__,
    ]);
}

if ($action === 'layout.block.create') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $zone = trim((string)($_POST['zone'] ?? ''));
    $type = trim((string)($_POST['type'] ?? 'text'));

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }
    if (!sb_layout_valid_zone($zone)) {
        sb_json_error('BAD_ZONE', 422);
    }
    if ($type === '') {
        sb_json_error('TYPE_REQUIRED', 422);
    }

    sb_require_content_manager($siteId);

    $layouts = sb_read_layouts();
    $updatedLayout = null;
    $newBlock = null;
    $found = false;

    foreach ($layouts as &$layout) {
        if ((int)($layout['siteId'] ?? 0) !== $siteId) {
            continue;
        }

        $layout = sb_normalize_layout_record($layout);

        $newBlock = [
            'id' => sb_layout_next_block_id($layout),
            'type' => $type,
            'sort' => sb_layout_next_block_sort($layout, $zone),
            'content' => sb_default_block_content($type),
            'props' => [],
            'createdBy' => (int)$USER->GetID(),
            'createdAt' => date('c'),
            'updatedAt' => date('c'),
            'updatedBy' => (int)$USER->GetID(),
        ];

        $layout['zones'][$zone][] = $newBlock;
        $layout['updatedAt'] = date('c');
        $layout['updatedBy'] = (int)$USER->GetID();

        $updatedLayout = $layout;
        $found = true;
        break;
    }
    unset($layout);

    if (!$found) {
        $updatedLayout = sb_layout_default_record($siteId);

        $newBlock = [
            'id' => 1,
            'type' => $type,
            'sort' => 10,
            'content' => sb_default_block_content($type),
            'props' => [],
            'createdBy' => (int)$USER->GetID(),
            'createdAt' => date('c'),
            'updatedAt' => date('c'),
            'updatedBy' => (int)$USER->GetID(),
        ];

        $updatedLayout['zones'][$zone][] = $newBlock;
        $updatedLayout['createdAt'] = date('c');
        $updatedLayout['createdBy'] = (int)$USER->GetID();
        $updatedLayout['updatedAt'] = date('c');
        $updatedLayout['updatedBy'] = (int)$USER->GetID();

        $layouts[] = $updatedLayout;
    }

    sb_write_layouts($layouts);

    sb_json_ok([
        'block' => sb_normalize_block_record($newBlock),
        'zone' => $zone,
        'layout' => sb_normalize_layout_record($updatedLayout),
        'handler' => 'layout',
        'file' => __FILE__,
    ]);
}

if ($action === 'layout.block.update') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $id = (int)($_POST['id'] ?? 0);

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }
    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }

    sb_require_content_manager($siteId);

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

    $layouts = sb_read_layouts();
    $updatedBlock = null;
    $updatedLayout = null;
    $foundLayout = false;
    $foundBlock = false;

    foreach ($layouts as &$layout) {
        if ((int)($layout['siteId'] ?? 0) !== $siteId) {
            continue;
        }

        $foundLayout = true;
        $layout = sb_normalize_layout_record($layout);

        foreach (['header', 'footer', 'left', 'right'] as $zone) {
            foreach ($layout['zones'][$zone] as &$block) {
                if ((int)($block['id'] ?? 0) !== $id) {
                    continue;
                }

                if ($newType !== null) {
                    $block['type'] = $newType;
                }
                if ($newContent !== null) {
                    $block['content'] = $newContent;
                }
                if ($newProps !== null) {
                    $block['props'] = $newProps;
                }

                $block['updatedAt'] = date('c');
                $block['updatedBy'] = (int)$USER->GetID();
                $block['_zone'] = $zone;

                $updatedBlock = $block;
                $foundBlock = true;
                break 2;
            }
            unset($block);
        }

        if ($foundBlock) {
            $layout['updatedAt'] = date('c');
            $layout['updatedBy'] = (int)$USER->GetID();
            $updatedLayout = $layout;
            break;
        }
    }
    unset($layout);

    if (!$foundLayout) {
        sb_json_error('LAYOUT_NOT_FOUND', 404);
    }

    if (!$foundBlock || !$updatedBlock) {
        sb_json_error('BLOCK_NOT_FOUND', 404);
    }

    sb_write_layouts($layouts);

    sb_json_ok([
        'block' => sb_normalize_block_record($updatedBlock),
        'layout' => sb_normalize_layout_record($updatedLayout),
        'handler' => 'layout',
        'file' => __FILE__,
    ]);
}

if ($action === 'layout.block.delete') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $id = (int)($_POST['id'] ?? 0);

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }
    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }

    sb_require_content_manager($siteId);

    $layouts = sb_read_layouts();
    $updatedLayout = null;
    $foundLayout = false;
    $foundBlock = false;

    foreach ($layouts as &$layout) {
        if ((int)($layout['siteId'] ?? 0) !== $siteId) {
            continue;
        }

        $foundLayout = true;
        $layout = sb_normalize_layout_record($layout);

        foreach (['header', 'footer', 'left', 'right'] as $zone) {
            $before = count($layout['zones'][$zone]);
            $layout['zones'][$zone] = array_values(array_filter(
                $layout['zones'][$zone],
                static function ($block) use ($id) {
                    return (int)($block['id'] ?? 0) !== $id;
                }
            ));

            if (count($layout['zones'][$zone]) !== $before) {
                $foundBlock = true;
                $layout['updatedAt'] = date('c');
                $layout['updatedBy'] = (int)$USER->GetID();
                $updatedLayout = $layout;
                break;
            }
        }

        if ($foundBlock) {
            break;
        }
    }
    unset($layout);

    if (!$foundLayout) {
        sb_json_error('LAYOUT_NOT_FOUND', 404);
    }

    if (!$foundBlock) {
        sb_json_error('BLOCK_NOT_FOUND', 404);
    }

    sb_write_layouts($layouts);

    sb_json_ok([
        'layout' => sb_normalize_layout_record($updatedLayout),
        'handler' => 'layout',
        'file' => __FILE__,
    ]);
}

if ($action === 'layout.block.move') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $id = (int)($_POST['id'] ?? 0);
    $dir = trim((string)($_POST['dir'] ?? ''));

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }
    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }
    if ($dir !== 'up' && $dir !== 'down') {
        sb_json_error('DIR_REQUIRED', 422);
    }

    sb_require_content_manager($siteId);

    $layouts = sb_read_layouts();
    $updatedLayout = null;
    $foundLayout = false;
    $foundBlock = false;

    foreach ($layouts as &$layout) {
        if ((int)($layout['siteId'] ?? 0) !== $siteId) {
            continue;
        }

        $foundLayout = true;
        $layout = sb_normalize_layout_record($layout);

        foreach (['header', 'footer', 'left', 'right'] as $zone) {
            $siblings = $layout['zones'][$zone];

            $pos = null;
            for ($i = 0, $cnt = count($siblings); $i < $cnt; $i++) {
                if ((int)($siblings[$i]['id'] ?? 0) === $id) {
                    $pos = $i;
                    break;
                }
            }

            if ($pos === null) {
                continue;
            }

            $foundBlock = true;

            if ($dir === 'up' && $pos === 0) {
                $updatedLayout = $layout;
                break 2;
            }

            if ($dir === 'down' && $pos === count($siblings) - 1) {
                $updatedLayout = $layout;
                break 2;
            }

            $swapPos = ($dir === 'up') ? $pos - 1 : $pos + 1;

            $sortA = (int)($siblings[$pos]['sort'] ?? 500);
            $sortB = (int)($siblings[$swapPos]['sort'] ?? 500);

            $layout['zones'][$zone][$pos]['sort'] = $sortB;
            $layout['zones'][$zone][$pos]['updatedAt'] = date('c');
            $layout['zones'][$zone][$pos]['updatedBy'] = (int)$USER->GetID();

            $layout['zones'][$zone][$swapPos]['sort'] = $sortA;
            $layout['zones'][$zone][$swapPos]['updatedAt'] = date('c');
            $layout['zones'][$zone][$swapPos]['updatedBy'] = (int)$USER->GetID();

            usort($layout['zones'][$zone], static function ($a, $b) {
                $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);
                if ($sortCmp !== 0) {
                    return $sortCmp;
                }
                return (int)($a['id'] ?? 0) <=> (int)$b['id'];
            });

            $layout['updatedAt'] = date('c');
            $layout['updatedBy'] = (int)$USER->GetID();
            $updatedLayout = $layout;
            break 2;
        }
    }
    unset($layout);

    if (!$foundLayout) {
        sb_json_error('LAYOUT_NOT_FOUND', 404);
    }

    if (!$foundBlock) {
        sb_json_error('BLOCK_NOT_FOUND', 404);
    }

    sb_write_layouts($layouts);

    sb_json_ok([
        'layout' => sb_normalize_layout_record($updatedLayout),
        'handler' => 'layout',
        'file' => __FILE__,
    ]);
}

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'layout',
    'action' => $action,
    'file' => __FILE__,
]);
```

## `/local/sitebuilder/api/handlers/template.php`

```php
<?php

global $USER;

if ($action === 'template.list') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    sb_require_viewer($siteId);

    $templates = array_map('sb_normalize_template_record', sb_templates_for_site($siteId));

    sb_json_ok([
        'templates' => $templates,
    ]);
}

if ($action === 'template.createFromPage') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $pageId = (int)($_POST['pageId'] ?? 0);
    $name = trim((string)($_POST['name'] ?? ''));

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }
    if ($pageId <= 0) {
        sb_json_error('PAGE_ID_REQUIRED', 422);
    }
    if ($name === '') {
        sb_json_error('NAME_REQUIRED', 422);
    }

    sb_require_content_manager($siteId);

    $page = sb_find_page($pageId);
    if (!$page || (int)($page['siteId'] ?? 0) !== $siteId) {
        sb_json_error('PAGE_NOT_IN_SITE', 422);
    }

    $pageBlocks = sb_blocks_for_page($pageId);

    $storedBlocks = [];
    foreach ($pageBlocks as $block) {
        $copy = sb_normalize_block_record($block);
        unset($copy['pageId']);
        $storedBlocks[] = $copy;
    }

    $templates = sb_read_templates();

    $template = [
        'id' => sb_next_template_id($templates),
        'siteId' => $siteId,
        'name' => $name,
        'sourcePageId' => $pageId,
        'blocks' => $storedBlocks,
        'createdBy' => (int)$USER->GetID(),
        'createdAt' => date('c'),
        'updatedAt' => date('c'),
        'updatedBy' => (int)$USER->GetID(),
    ];

    $templates[] = $template;
    sb_write_templates($templates);

    sb_json_ok([
        'template' => sb_normalize_template_record($template),
    ]);
}

if ($action === 'template.applyToPage') {
    $templateId = (int)($_POST['templateId'] ?? 0);
    $pageId = (int)($_POST['pageId'] ?? 0);

    if ($templateId <= 0) {
        sb_json_error('TEMPLATE_ID_REQUIRED', 422);
    }
    if ($pageId <= 0) {
        sb_json_error('PAGE_ID_REQUIRED', 422);
    }

    $template = sb_find_template($templateId);
    if (!$template) {
        sb_json_error('TEMPLATE_NOT_FOUND', 404);
    }

    $page = sb_find_page($pageId);
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    if ((int)($template['siteId'] ?? 0) !== $siteId) {
        sb_json_error('TEMPLATE_NOT_IN_SITE', 422);
    }

    sb_require_content_manager($siteId);

    $blocks = sb_read_blocks();

    $blocks = array_values(array_filter($blocks, static function ($b) use ($pageId) {
        return (int)($b['pageId'] ?? 0) !== $pageId;
    }));

    $nextBlockId = sb_next_block_id($blocks);

    $templateBlocks = $template['blocks'] ?? [];
    usort($templateBlocks, static function ($a, $b) {
        $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);
        if ($sortCmp !== 0) {
            return $sortCmp;
        }
        return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
    });

    $sort = 10;
    foreach ($templateBlocks as $tplBlock) {
        $newBlock = sb_normalize_block_record($tplBlock);
        $newBlock['id'] = $nextBlockId++;
        $newBlock['pageId'] = $pageId;
        $newBlock['sort'] = $sort;
        $newBlock['createdBy'] = (int)$USER->GetID();
        $newBlock['createdAt'] = date('c');
        $newBlock['updatedAt'] = date('c');
        $newBlock['updatedBy'] = (int)$USER->GetID();
        $blocks[] = $newBlock;
        $sort += 10;
    }

    sb_write_blocks($blocks);

    sb_json_ok([
        'blocks' => array_map('sb_normalize_block_record', sb_blocks_for_page($pageId)),
    ]);
}

if ($action === 'template.rename') {
    $id = (int)($_POST['id'] ?? 0);
    $name = trim((string)($_POST['name'] ?? ''));

    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }
    if ($name === '') {
        sb_json_error('NAME_REQUIRED', 422);
    }

    $template = sb_find_template($id);
    if (!$template) {
        sb_json_error('TEMPLATE_NOT_FOUND', 404);
    }

    $siteId = (int)($template['siteId'] ?? 0);
    sb_require_content_manager($siteId);

    $templates = sb_read_templates();
    $updated = null;

    foreach ($templates as &$tpl) {
        if ((int)($tpl['id'] ?? 0) === $id) {
            $tpl['name'] = $name;
            $tpl['updatedAt'] = date('c');
            $tpl['updatedBy'] = (int)$USER->GetID();
            $updated = $tpl;
            break;
        }
    }
    unset($tpl);

    if (!$updated) {
        sb_json_error('TEMPLATE_NOT_FOUND', 404);
    }

    sb_write_templates($templates);

    sb_json_ok([
        'template' => sb_normalize_template_record($updated),
    ]);
}

if ($action === 'template.delete') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }

    $template = sb_find_template($id);
    if (!$template) {
        sb_json_error('TEMPLATE_NOT_FOUND', 404);
    }

    $siteId = (int)($template['siteId'] ?? 0);
    sb_require_content_manager($siteId);

    $templates = sb_read_templates();
    $before = count($templates);

    $templates = array_values(array_filter($templates, static function ($tpl) use ($id) {
        return (int)($tpl['id'] ?? 0) !== $id;
    }));

    if (count($templates) === $before) {
        sb_json_error('TEMPLATE_NOT_FOUND', 404);
    }

    sb_write_templates($templates);

    sb_json_ok();
}

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'template',
    'action' => $action,
]);
```

В `layout.php` обрати внимание: я оставил `layout.get` и `layout.block.list` через `sb_require_viewer()`, потому что это чтение. Всё изменение layout — через `sb_require_content_manager()`.
