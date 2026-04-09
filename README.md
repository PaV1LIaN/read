Отлично. Тогда следующий шаг — settings.php.

Сделаем рабочую страницу настроек сайта, без красоты “финальной версии”, но уже полезную:

загрузка сайта

редактирование name

редактирование slug

containerWidth

accent

logoFileId

homePageId

сохранение через API


Но сейчас есть один важный момент: у нас ещё нет action site.update в живом роутере и handler-е.
Поэтому сначала добавим его, а потом саму страницу.


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

sb_json_error('UNKNOWN_ACTION', 400, [
    'action' => $action,
    'file' => __FILE__,
]);


---

2. Обнови /local/sitebuilder/api/handlers/site.php

Замени файл целиком на этот вариант:

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

if ($action === 'site.update') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    sb_require_editor($siteId);

    $site = sb_find_site($siteId);
    if (!$site) {
        sb_json_error('SITE_NOT_FOUND', 404);
    }

    $name = trim((string)($_POST['name'] ?? ''));
    $slug = trim((string)($_POST['slug'] ?? ''));
    $containerWidth = (int)($_POST['containerWidth'] ?? 0);
    $accent = trim((string)($_POST['accent'] ?? ''));
    $logoFileId = (int)($_POST['logoFileId'] ?? 0);

    if ($name === '') {
        $name = (string)($site['name'] ?? '');
    }

    if ($slug === '') {
        $slug = (string)($site['slug'] ?? '');
    }
    $slug = sb_slugify($slug);
    if ($slug === '') {
        $slug = 'site-' . $siteId;
    }

    $sites = sb_read_sites();

    $existing = array_map(
        static function ($x) {
            return (string)($x['slug'] ?? '');
        },
        array_filter($sites, static function ($s) use ($siteId) {
            return (int)($s['id'] ?? 0) !== $siteId;
        })
    );

    $base = $slug;
    $i = 2;
    while (in_array($slug, $existing, true)) {
        $slug = $base . '-' . $i;
        $i++;
    }

    if ($containerWidth <= 0) {
        $containerWidth = (int)($site['settings']['containerWidth'] ?? 1100);
    }
    $containerWidth = max(320, min(1920, $containerWidth));

    if ($accent === '') {
        $accent = (string)($site['settings']['accent'] ?? '#2563eb');
    }

    if (!preg_match('/^#[0-9a-fA-F]{6}$/', $accent)) {
        $accent = '#2563eb';
    }

    $updated = null;

    foreach ($sites as &$s) {
        if ((int)($s['id'] ?? 0) !== $siteId) {
            continue;
        }

        $settings = isset($s['settings']) && is_array($s['settings']) ? $s['settings'] : [];
        $settings['containerWidth'] = $containerWidth;
        $settings['accent'] = $accent;
        $settings['logoFileId'] = $logoFileId;

        $s['name'] = $name;
        $s['slug'] = $slug;
        $s['settings'] = $settings;
        $s['updatedAt'] = date('c');
        $s['updatedBy'] = (int)$USER->GetID();

        $updated = $s;
        break;
    }
    unset($s);

    if (!$updated) {
        sb_json_error('SITE_NOT_FOUND', 404);
    }

    sb_write_sites($sites);

    sb_json_ok([
        'site' => $updated,
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

3. Создай /local/sitebuilder/settings.php

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
        <title>SiteBuilder / Settings</title>
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
    <title>SiteBuilder / Settings</title>
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
            max-width: 980px;
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
        .form-grid {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 14px;
        }
        .field {
            display: flex;
            flex-direction: column;
            gap: 6px;
        }
        .field.full {
            grid-column: 1 / -1;
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
        .field input:focus,
        .field select:focus {
            border-color: #2563eb;
        }
        .toolbar {
            display: flex;
            gap: 10px;
            flex-wrap: wrap;
            margin-top: 16px;
        }
        .btn {
            height: 40px;
            border: 0;
            border-radius: 10px;
            padding: 0 16px;
            cursor: pointer;
            font-weight: 600;
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
        .meta {
            font-size: 13px;
            color: #6b7280;
            line-height: 1.6;
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
        .empty {
            padding: 20px;
            text-align: center;
            color: #6b7280;
            border: 1px dashed #d1d5db;
            border-radius: 12px;
            background: #fff;
        }
        @media (max-width: 800px) {
            .form-grid {
                grid-template-columns: 1fr;
            }
        }
    </style>
</head>
<body>
<div class="page">
    <div class="topbar">
        <a class="back-link" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/index.php">← К списку сайтов</a>
        <h1 class="title">Настройки сайта</h1>
        <p class="subtitle">siteId = <?= (int)$siteId ?></p>
    </div>

    <div class="panel">
        <h2 class="panel-title">Основные настройки</h2>

        <div id="settingsEmpty" class="empty">Загрузка...</div>

        <div id="settingsForm" style="display:none;">
            <div class="form-grid">
                <div class="field">
                    <label for="siteName">Название сайта</label>
                    <input type="text" id="siteName">
                </div>

                <div class="field">
                    <label for="siteSlug">Slug</label>
                    <input type="text" id="siteSlug">
                </div>

                <div class="field">
                    <label for="containerWidth">Ширина контейнера</label>
                    <input type="number" id="containerWidth" min="320" max="1920">
                </div>

                <div class="field">
                    <label for="accent">Accent color</label>
                    <input type="text" id="accent" placeholder="#2563eb">
                </div>

                <div class="field">
                    <label for="logoFileId">Logo file ID</label>
                    <input type="number" id="logoFileId" min="0">
                </div>

                <div class="field">
                    <label for="homePageId">Домашняя страница</label>
                    <select id="homePageId"></select>
                </div>

                <div class="field full">
                    <label>Служебная информация</label>
                    <div class="meta" id="siteMeta"></div>
                </div>
            </div>

            <div class="toolbar">
                <button type="button" class="btn btn-primary" id="saveSettingsBtn">Сохранить</button>
                <button type="button" class="btn btn-light" id="reloadBtn">Обновить</button>
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
    var settingsEmpty = document.getElementById('settingsEmpty');
    var settingsForm = document.getElementById('settingsForm');

    var state = {
        site: null,
        pages: []
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

    function loadSite(next) {
        api('site.get', { siteId: SITE_ID }, function (res) {
            if (!res || res.ok !== true) {
                settingsEmpty.textContent = 'Не удалось загрузить сайт';
                return;
            }

            state.site = res.site || null;

            if (typeof next === 'function') {
                next();
            }
        });
    }

    function loadPages(next) {
        api('page.list', { siteId: SITE_ID }, function (res) {
            if (res && res.ok === true) {
                state.pages = Array.isArray(res.pages) ? res.pages : [];
            } else {
                state.pages = [];
            }

            if (typeof next === 'function') {
                next();
            }
        });
    }

    function renderHomePageOptions() {
        var select = document.getElementById('homePageId');
        var html = '<option value="0">Не выбрана</option>';

        for (var i = 0; i < state.pages.length; i++) {
            var page = state.pages[i];
            var id = Number(page.id || 0);
            var selected = id === Number((state.site && state.site.homePageId) || 0) ? ' selected' : '';
            html += '<option value="' + id + '"' + selected + '>'
                + escapeHtml(page.title || ('Страница #' + id))
                + ' (#' + id + ')</option>';
        }

        select.innerHTML = html;
    }

    function renderSite() {
        if (!state.site) {
            settingsEmpty.textContent = 'Сайт не найден';
            return;
        }

        document.getElementById('siteName').value = state.site.name || '';
        document.getElementById('siteSlug').value = state.site.slug || '';
        document.getElementById('containerWidth').value = Number((state.site.settings && state.site.settings.containerWidth) || 1100);
        document.getElementById('accent').value = (state.site.settings && state.site.settings.accent) || '#2563eb';
        document.getElementById('logoFileId').value = Number((state.site.settings && state.site.settings.logoFileId) || 0);

        renderHomePageOptions();

        document.getElementById('siteMeta').innerHTML =
            '<div><strong>ID:</strong> ' + Number(state.site.id || 0) + '</div>'
            + '<div><strong>Disk folder ID:</strong> ' + Number(state.site.diskFolderId || 0) + '</div>'
            + '<div><strong>Top menu ID:</strong> ' + Number(state.site.topMenuId || 0) + '</div>'
            + '<div><strong>Created at:</strong> ' + escapeHtml(state.site.createdAt || '') + '</div>'
            + '<div><strong>Updated at:</strong> ' + escapeHtml(state.site.updatedAt || '') + '</div>';

        settingsEmpty.style.display = 'none';
        settingsForm.style.display = '';
    }

    function saveSettings() {
        if (!state.site) {
            return;
        }

        var siteName = (document.getElementById('siteName').value || '').trim();
        var siteSlug = (document.getElementById('siteSlug').value || '').trim();
        var containerWidth = parseInt(document.getElementById('containerWidth').value, 10) || 1100;
        var accent = (document.getElementById('accent').value || '').trim();
        var logoFileId = parseInt(document.getElementById('logoFileId').value, 10) || 0;
        var homePageId = parseInt(document.getElementById('homePageId').value, 10) || 0;

        if (!siteName) {
            alert('Название сайта не может быть пустым');
            return;
        }

        api('site.update', {
            siteId: SITE_ID,
            name: siteName,
            slug: siteSlug,
            containerWidth: containerWidth,
            accent: accent,
            logoFileId: logoFileId
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось сохранить настройки сайта');
                return;
            }

            state.site = res.site || state.site;

            if (homePageId > 0) {
                api('site.setHome', {
                    siteId: SITE_ID,
                    pageId: homePageId
                }, function (res2) {
                    if (!res2 || res2.ok !== true) {
                        alert('Основные настройки сохранены, но не удалось установить домашнюю страницу');
                        loadSite(function () {
                            loadPages(renderSite);
                        });
                        return;
                    }

                    loadSite(function () {
                        loadPages(renderSite);
                    });
                });
            } else {
                var oldSite = state.site || {};
                oldSite.homePageId = 0;
                state.site = oldSite;
                renderSite();
                alert('Настройки сохранены. Если нужно сбросить home page в API полностью, это можно добавить отдельным action.');
            }
        });
    }

    document.getElementById('saveSettingsBtn').addEventListener('click', saveSettings);
    document.getElementById('reloadBtn').addEventListener('click', function () {
        loadSite(function () {
            loadPages(renderSite);
        });
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
        loadPages(renderSite);
    });
})();
</script>
</body>
</html>


---

4. Сбрось opcache

Открой один раз:

/local/sitebuilder/opcache_reset.php


---

5. Что проверить

Открывай:

/local/sitebuilder/settings.php?siteId=1

Проверь:

1. грузится ли сайт


2. меняется ли name


3. меняется ли slug


4. сохраняется ли containerWidth


5. сохраняется ли accent


6. сохраняется ли logoFileId


7. устанавливается ли home page




---

Что важно знать сейчас

Сейчас homePageId = 0 через UI не отправляется отдельным action на сброс, я это специально оставил простым, чтобы не ломать текущую схему. Если захочешь — следующим сообщением добавим site.clearHome.


---

Куда идём дальше

После settings.php логично:

1. вернуть layout.* в роутер


2. проверить layout.php


3. потом уже перейти к “сделать красиво”:

разрезать JS

вынести CSS

нормальные диалоги

prettier UI




Напиши, как отработал settings.php, и следующим сообщением я дам уже рабочий пакет для layout.php.