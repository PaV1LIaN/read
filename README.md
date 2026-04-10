Отлично. Тогда идём дальше и подключаем layout.*, чтобы заработала страница layout.php.

Сейчас сделаем:

1. роутер для layout.*


2. проверим handler


3. соберём рабочую страницу управления зонами:

header

footer

left

right





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
    $action === 'site.update' ||
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

if (
    $action === 'file.list' ||
    $action === 'file.upload' ||
    $action === 'file.delete'
) {
    require __DIR__ . '/handlers/file.php';
    exit;
}

if (
    $action === 'layout.get' ||
    $action === 'layout.updateSettings' ||
    $action === 'layout.block.list' ||
    $action === 'layout.block.create' ||
    $action === 'layout.block.update' ||
    $action === 'layout.block.delete' ||
    $action === 'layout.block.move'
) {
    require __DIR__ . '/handlers/layout.php';
    exit;
}

sb_json_error('UNKNOWN_ACTION', 400, [
    'action' => $action,
    'file' => __FILE__,
]);


---

2. Проверь /local/sitebuilder/api/handlers/layout.php

Если не уверен, просто замени целиком на этот вариант:

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

    sb_require_editor($siteId);

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

    sb_require_editor($siteId);

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

    sb_require_editor($siteId);

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

    sb_require_editor($siteId);

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
                return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
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


---

3. Создай /local/sitebuilder/layout.php

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
        <title>SiteBuilder / Layout</title>
        <?php $APPLICATION->ShowHead(); ?>
    </head>
    <body style="font-family:Arial,sans-serif;padding:20px;">
        <h1>Не передан siteId</h1>
        <p><a href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/index.php">Вернуться к списку сайтов</a></p>
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
    <title>SiteBuilder / Layout</title>
    <?php $APPLICATION->ShowHead(); ?>
    <style>
        * { box-sizing: border-box; }
        body {
            margin: 0;
            font-family: Arial, sans-serif;
            background: #f6f8fb;
            color: #1f2937;
        }
        .page {
            max-width: 1480px;
            margin: 0 auto;
            padding: 24px;
        }
        .topbar {
            margin-bottom: 20px;
        }
        .title {
            margin: 0;
            font-size: 28px;
            font-weight: 700;
        }
        .subtitle {
            margin: 6px 0 0;
            color: #6b7280;
            font-size: 14px;
        }
        .back-link {
            display: inline-block;
            margin-bottom: 8px;
            text-decoration: none;
            color: #1d4ed8;
            font-size: 14px;
        }
        .panel {
            background: #fff;
            border: 1px solid #e5e7eb;
            border-radius: 14px;
            padding: 18px;
            margin-bottom: 20px;
            box-shadow: 0 2px 8px rgba(15, 23, 42, 0.04);
        }
        .panel-title {
            margin: 0 0 14px;
            font-size: 18px;
            font-weight: 700;
        }
        .settings-grid {
            display: grid;
            grid-template-columns: repeat(4, 1fr);
            gap: 14px;
        }
        .field {
            display: flex;
            flex-direction: column;
            gap: 6px;
        }
        .field label {
            font-size: 13px;
            color: #4b5563;
        }
        .field input,
        .field select {
            width: 100%;
            height: 40px;
            padding: 0 12px;
            border: 1px solid #d1d5db;
            border-radius: 10px;
            outline: none;
            background: #fff;
        }
        .zones-grid {
            display: grid;
            grid-template-columns: repeat(2, 1fr);
            gap: 16px;
        }
        .zone-card {
            border: 1px solid #e5e7eb;
            border-radius: 14px;
            background: #fff;
            overflow: hidden;
        }
        .zone-head {
            padding: 14px 16px;
            border-bottom: 1px solid #e5e7eb;
            display: flex;
            justify-content: space-between;
            align-items: center;
            gap: 12px;
        }
        .zone-title {
            margin: 0;
            font-size: 16px;
            font-weight: 700;
            text-transform: capitalize;
        }
        .zone-body {
            padding: 16px;
        }
        .toolbar {
            display: flex;
            gap: 8px;
            flex-wrap: wrap;
            margin-bottom: 12px;
        }
        .btn {
            height: 40px;
            border: 0;
            border-radius: 10px;
            padding: 0 16px;
            cursor: pointer;
            font-weight: 600;
        }
        .btn-small {
            height: 34px;
            padding: 0 12px;
            font-size: 13px;
        }
        .btn-primary {
            background: #2563eb;
            color: #fff;
        }
        .btn-primary:hover {
            background: #1d4ed8;
        }
        .btn-light {
            background: #eef2ff;
            color: #1e3a8a;
        }
        .btn-light:hover {
            background: #e0e7ff;
        }
        .btn-danger {
            background: #dc2626;
            color: #fff;
        }
        .btn-danger:hover {
            background: #b91c1c;
        }
        .btn-gray {
            background: #f3f4f6;
            color: #374151;
        }
        .btn-gray:hover {
            background: #e5e7eb;
        }
        .layout-blocks {
            display: flex;
            flex-direction: column;
            gap: 10px;
        }
        .layout-block {
            border: 1px solid #e5e7eb;
            border-radius: 12px;
            background: #fafafa;
            padding: 12px;
        }
        .layout-block-head {
            display: flex;
            justify-content: space-between;
            gap: 12px;
            align-items: flex-start;
        }
        .block-title {
            margin: 0 0 6px;
            font-size: 15px;
            font-weight: 700;
        }
        .meta {
            font-size: 13px;
            color: #6b7280;
            line-height: 1.5;
        }
        .actions {
            margin-top: 10px;
            display: flex;
            flex-wrap: wrap;
            gap: 8px;
        }
        .badge {
            display: inline-flex;
            align-items: center;
            background: #f3f4f6;
            color: #374151;
            border-radius: 999px;
            padding: 4px 10px;
            font-size: 12px;
            white-space: nowrap;
        }
        .editor-box {
            margin-top: 20px;
        }
        .editor-empty {
            padding: 20px;
            text-align: center;
            color: #6b7280;
            border: 1px dashed #d1d5db;
            border-radius: 12px;
            background: #fff;
        }
        textarea {
            width: 100%;
            min-height: 120px;
            padding: 10px 12px;
            border: 1px solid #d1d5db;
            border-radius: 10px;
            outline: none;
            resize: vertical;
            font: inherit;
        }
        .output {
            white-space: pre-wrap;
            background: #0f172a;
            color: #e5e7eb;
            border-radius: 12px;
            padding: 14px;
            min-height: 120px;
            font-family: Consolas, Monaco, monospace;
            font-size: 13px;
            overflow: auto;
        }
        .hidden {
            display: none !important;
        }
        @media (max-width: 1200px) {
            .settings-grid,
            .zones-grid {
                grid-template-columns: 1fr;
            }
        }
    </style>
</head>
<body>
<div class="page">
    <div class="topbar">
        <a class="back-link" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/index.php">← К списку сайтов</a>
        <h1 class="title">Layout сайта</h1>
        <p class="subtitle">siteId = <?= (int)$siteId ?></p>
    </div>

    <div class="panel">
        <h2 class="panel-title">Настройки layout</h2>

        <div class="settings-grid">
            <div class="field">
                <label for="showHeader">Show header</label>
                <select id="showHeader">
                    <option value="1">Да</option>
                    <option value="0">Нет</option>
                </select>
            </div>

            <div class="field">
                <label for="showFooter">Show footer</label>
                <select id="showFooter">
                    <option value="1">Да</option>
                    <option value="0">Нет</option>
                </select>
            </div>

            <div class="field">
                <label for="showLeft">Show left</label>
                <select id="showLeft">
                    <option value="1">Да</option>
                    <option value="0">Нет</option>
                </select>
            </div>

            <div class="field">
                <label for="showRight">Show right</label>
                <select id="showRight">
                    <option value="1">Да</option>
                    <option value="0">Нет</option>
                </select>
            </div>

            <div class="field">
                <label for="leftWidth">Left width</label>
                <input type="number" id="leftWidth" min="120" max="800">
            </div>

            <div class="field">
                <label for="rightWidth">Right width</label>
                <input type="number" id="rightWidth" min="120" max="800">
            </div>

            <div class="field">
                <label for="leftMode">Left mode</label>
                <select id="leftMode">
                    <option value="blocks">blocks</option>
                    <option value="menu">menu</option>
                </select>
            </div>
        </div>

        <div class="toolbar" style="margin-top:16px;">
            <button type="button" class="btn btn-primary" id="saveSettingsBtn">Сохранить layout settings</button>
            <button type="button" class="btn btn-light" id="reloadBtn">Обновить</button>
        </div>
    </div>

    <div class="zones-grid" id="zonesContainer"></div>

    <div class="panel editor-box">
        <h2 class="panel-title">Редактор layout-блока</h2>

        <div id="layoutBlockEditorEmpty" class="editor-empty">Выберите блок для редактирования</div>

        <div id="layoutBlockEditorForm" class="hidden">
            <div class="field" style="margin-bottom:12px;">
                <label for="editLayoutBlockZone">Zone</label>
                <input type="text" id="editLayoutBlockZone" readonly>
            </div>

            <div class="field" style="margin-bottom:12px;">
                <label for="editLayoutBlockType">Тип</label>
                <input type="text" id="editLayoutBlockType" readonly>
            </div>

            <div class="field" style="margin-bottom:12px;">
                <label for="editLayoutBlockContent">Content / JSON</label>
                <textarea id="editLayoutBlockContent"></textarea>
            </div>

            <div class="field" style="margin-bottom:12px;">
                <label for="editLayoutBlockProps">Props / JSON</label>
                <textarea id="editLayoutBlockProps">{}</textarea>
            </div>

            <div class="toolbar">
                <button type="button" class="btn btn-primary" id="saveLayoutBlockBtn">Сохранить</button>
                <button type="button" class="btn btn-danger" id="deleteLayoutBlockBtn">Удалить</button>
            </div>
        </div>
    </div>

    <div class="panel">
        <h2 class="panel-title">Отладка</h2>
        <div id="output" class="output">Здесь будут ответы API...</div>
    </div>
</div>

<script>
(function () {
    var BASE_PATH = '<?= CUtil::JSEscape($basePath) ?>';
    var API_URL = BASE_PATH + '/api.php';
    var SITE_ID = <?= (int)$siteId ?>;

    var output = document.getElementById('output');
    var zonesContainer = document.getElementById('zonesContainer');

    var state = {
        layout: null,
        currentBlockId: 0,
        currentZone: ''
    };

    var ZONES = ['header', 'footer', 'left', 'right'];

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

    function loadLayout() {
        api('layout.get', { siteId: SITE_ID }, function (res) {
            if (!res || res.ok !== true) {
                zonesContainer.innerHTML = '<div class="editor-empty">Не удалось загрузить layout</div>';
                return;
            }

            state.layout = res.layout || null;
            renderSettings();
            renderZones();
        });
    }

    function renderSettings() {
        if (!state.layout || !state.layout.settings) {
            return;
        }

        var s = state.layout.settings;
        document.getElementById('showHeader').value = s.showHeader ? '1' : '0';
        document.getElementById('showFooter').value = s.showFooter ? '1' : '0';
        document.getElementById('showLeft').value = s.showLeft ? '1' : '0';
        document.getElementById('showRight').value = s.showRight ? '1' : '0';
        document.getElementById('leftWidth').value = Number(s.leftWidth || 260);
        document.getElementById('rightWidth').value = Number(s.rightWidth || 260);
        document.getElementById('leftMode').value = s.leftMode || 'blocks';
    }

    function renderZones() {
        if (!state.layout || !state.layout.zones) {
            zonesContainer.innerHTML = '<div class="editor-empty">Нет layout-зон</div>';
            return;
        }

        var html = '';
        for (var i = 0; i < ZONES.length; i++) {
            html += renderZoneCard(ZONES[i], state.layout.zones[ZONES[i]] || []);
        }
        zonesContainer.innerHTML = html;
    }

    function renderZoneCard(zone, blocks) {
        var html = ''
            + '<div class="zone-card">'
            + '  <div class="zone-head">'
            + '    <h3 class="zone-title">' + escapeHtml(zone) + '</h3>'
            + '    <span class="badge">' + blocks.length + ' блок(ов)</span>'
            + '  </div>'
            + '  <div class="zone-body">'
            + '    <div class="toolbar">'
            + '      <button type="button" class="btn btn-light btn-small js-add-layout-block" data-zone="' + zone + '" data-type="text">+ Text</button>'
            + '      <button type="button" class="btn btn-light btn-small js-add-layout-block" data-zone="' + zone + '" data-type="heading">+ Heading</button>'
            + '      <button type="button" class="btn btn-light btn-small js-add-layout-block" data-zone="' + zone + '" data-type="button">+ Button</button>'
            + '      <button type="button" class="btn btn-light btn-small js-add-layout-block" data-zone="' + zone + '" data-type="html">+ HTML</button>'
            + '    </div>';

        if (!blocks.length) {
            html += '<div class="editor-empty">Блоков нет</div>';
        } else {
            html += '<div class="layout-blocks">';
            for (var i = 0; i < blocks.length; i++) {
                html += renderLayoutBlock(zone, blocks[i], i, blocks.length);
            }
            html += '</div>';
        }

        html += '</div></div>';
        return html;
    }

    function renderLayoutBlock(zone, block, index, total) {
        var id = Number(block.id || 0);
        var type = escapeHtml(block.type || '');
        var preview = escapeHtml(buildPreviewText(block));

        return ''
            + '<div class="layout-block">'
            + '  <div class="layout-block-head">'
            + '    <div>'
            + '      <div class="block-title">' + type + ' #' + id + '</div>'
            + '      <div class="meta">'
            + '        <div><strong>Zone:</strong> ' + escapeHtml(zone) + '</div>'
            + '        <div><strong>Sort:</strong> ' + Number(block.sort || 0) + '</div>'
            + '      </div>'
            + '    </div>'
            + '    <span class="badge">layout block</span>'
            + '  </div>'
            + '  <div class="meta" style="margin-top:8px;">' + preview + '</div>'
            + '  <div class="actions">'
            + '    <button type="button" class="btn btn-light btn-small js-edit-layout-block" data-zone="' + zone + '" data-id="' + id + '">Редактировать</button>'
            + '    <button type="button" class="btn btn-gray btn-small js-move-layout-block-up" data-id="' + id + '"' + (index === 0 ? ' disabled' : '') + '>↑</button>'
            + '    <button type="button" class="btn btn-gray btn-small js-move-layout-block-down" data-id="' + id + '"' + (index === total - 1 ? ' disabled' : '') + '>↓</button>'
            + '    <button type="button" class="btn btn-danger btn-small js-delete-layout-block" data-id="' + id + '">Удалить</button>'
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

        return JSON.stringify(content);
    }

    function saveLayoutSettings() {
        var settings = {
            showHeader: document.getElementById('showHeader').value === '1',
            showFooter: document.getElementById('showFooter').value === '1',
            showLeft: document.getElementById('showLeft').value === '1',
            showRight: document.getElementById('showRight').value === '1',
            leftWidth: parseInt(document.getElementById('leftWidth').value, 10) || 260,
            rightWidth: parseInt(document.getElementById('rightWidth').value, 10) || 260,
            leftMode: document.getElementById('leftMode').value || 'blocks'
        };

        api('layout.updateSettings', {
            siteId: SITE_ID,
            settings: JSON.stringify(settings)
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось сохранить layout settings');
                return;
            }

            state.layout = res.layout || state.layout;
            renderSettings();
            renderZones();
        });
    }

    function addLayoutBlock(zone, type) {
        api('layout.block.create', {
            siteId: SITE_ID,
            zone: zone,
            type: type
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось создать layout block');
                return;
            }

            state.layout = res.layout || state.layout;
            renderZones();
        });
    }

    function findLayoutBlock(blockId) {
        if (!state.layout || !state.layout.zones) {
            return null;
        }

        for (var i = 0; i < ZONES.length; i++) {
            var zone = ZONES[i];
            var blocks = state.layout.zones[zone] || [];
            for (var j = 0; j < blocks.length; j++) {
                if (Number(blocks[j].id || 0) === Number(blockId || 0)) {
                    return {
                        zone: zone,
                        block: blocks[j]
                    };
                }
            }
        }

        return null;
    }

    function editLayoutBlock(zone, blockId) {
        var found = findLayoutBlock(blockId);
        if (!found) {
            alert('Блок не найден');
            return;
        }

        state.currentBlockId = Number(blockId || 0);
        state.currentZone = zone;

        document.getElementById('layoutBlockEditorEmpty').classList.add('hidden');
        document.getElementById('layoutBlockEditorForm').classList.remove('hidden');
        document.getElementById('editLayoutBlockZone').value = zone;
        document.getElementById('editLayoutBlockType').value = found.block.type || '';
        document.getElementById('editLayoutBlockContent').value = JSON.stringify(found.block.content || {}, null, 2);
        document.getElementById('editLayoutBlockProps').value = JSON.stringify(found.block.props || {}, null, 2);
    }

    function clearLayoutBlockEditor() {
        state.currentBlockId = 0;
        state.currentZone = '';
        document.getElementById('layoutBlockEditorEmpty').classList.remove('hidden');
        document.getElementById('layoutBlockEditorForm').classList.add('hidden');
        document.getElementById('editLayoutBlockZone').value = '';
        document.getElementById('editLayoutBlockType').value = '';
        document.getElementById('editLayoutBlockContent').value = '';
        document.getElementById('editLayoutBlockProps').value = '{}';
    }

    function saveLayoutBlock() {
        if (!state.currentBlockId) {
            return;
        }

        var contentText = document.getElementById('editLayoutBlockContent').value || '{}';
        var propsText = document.getElementById('editLayoutBlockProps').value || '{}';

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

        api('layout.block.update', {
            siteId: SITE_ID,
            id: state.currentBlockId,
            content: contentText,
            props: propsText
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось сохранить layout block');
                return;
            }

            state.layout = res.layout || state.layout;
            renderZones();
        });
    }

    function deleteLayoutBlock(blockId) {
        if (!confirm('Удалить layout block #' + blockId + '?')) {
            return;
        }

        api('layout.block.delete', {
            siteId: SITE_ID,
            id: blockId
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось удалить layout block');
                return;
            }

            state.layout = res.layout || state.layout;

            if (Number(state.currentBlockId || 0) === Number(blockId || 0)) {
                clearLayoutBlockEditor();
            }

            renderZones();
        });
    }

    function moveLayoutBlock(blockId, dir) {
        api('layout.block.move', {
            siteId: SITE_ID,
            id: blockId,
            dir: dir
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось переместить layout block');
                return;
            }

            state.layout = res.layout || state.layout;
            renderZones();
        });
    }

    document.getElementById('saveSettingsBtn').addEventListener('click', saveLayoutSettings);
    document.getElementById('reloadBtn').addEventListener('click', loadLayout);
    document.getElementById('saveLayoutBlockBtn').addEventListener('click', saveLayoutBlock);
    document.getElementById('deleteLayoutBlockBtn').addEventListener('click', function () {
        if (state.currentBlockId) {
            deleteLayoutBlock(state.currentBlockId);
        }
    });

    document.addEventListener('click', function (e) {
        var addBtn = e.target.closest('.js-add-layout-block');
        if (addBtn) {
            addLayoutBlock(
                addBtn.getAttribute('data-zone') || '',
                addBtn.getAttribute('data-type') || 'text'
            );
            return;
        }

        var editBtn = e.target.closest('.js-edit-layout-block');
        if (editBtn) {
            editLayoutBlock(
                editBtn.getAttribute('data-zone') || '',
                parseInt(editBtn.getAttribute('data-id'), 10) || 0
            );
            return;
        }

        var delBtn = e.target.closest('.js-delete-layout-block');
        if (delBtn) {
            deleteLayoutBlock(parseInt(delBtn.getAttribute('data-id'), 10) || 0);
            return;
        }

        var upBtn = e.target.closest('.js-move-layout-block-up');
        if (upBtn) {
            moveLayoutBlock(parseInt(upBtn.getAttribute('data-id'), 10) || 0, 'up');
            return;
        }

        var downBtn = e.target.closest('.js-move-layout-block-down');
        if (downBtn) {
            moveLayoutBlock(parseInt(downBtn.getAttribute('data-id'), 10) || 0, 'down');
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

    loadLayout();
})();
</script>
</body>
</html>


---

4. Сбрось opcache

Открой:

/local/sitebuilder/opcache_reset.php


---

5. Что проверить

Открывай:

/local/sitebuilder/layout.php?siteId=1

Проверь по шагам:

1. грузятся layout settings


2. сохраняются settings


3. добавляется блок в header


4. добавляется блок в footer


5. редактируется content/props


6. блок двигается вверх/вниз


7. блок удаляется




---

Что дальше

Если layout.php оживёт, у тебя уже будет рабочий каркас почти всего проекта:

index.php

editor.php

menu.php

files.php

settings.php

layout.php


И после этого можно будет переходить к следующему этапу: приводить всё в красивую структуру и выносить CSS/JS по файлам.

Напиши, как отработал layout.php.