Нашли проблему точно.

У тебя site.list уходит не в site.php, а заканчивается в access.php, и тот отвечает своим хвостом:

{"ok":false,"error":"NOT_MOVED_YET","handler":"access","action":"site.list"}

Значит проблема не в AJAX и не в Bitrix. Проблема в роутере api/index.php или в том, что файл site.php пустой/не тот.

Что это означает

Вот этот ответ:

{"ok":false,"error":"NOT_MOVED_YET","handler":"access","action":"site.list"}

говорит, что:

запрос до PHP доходит

api.php и bootstrap.php работают

sessid нормальный

роутинг сейчас настроен неправильно


То есть мы уже сильно продвинулись: инфраструктура живая, сломан только dispatch.


---

Что проверить первым

Открой файл:

/local/ТВОЯ_ПАПКА/api/index.php

И проверь, что там точно такой маппинг для site.*:

'site.list' => __DIR__ . '/handlers/site.php',
'site.get' => __DIR__ . '/handlers/site.php',
'site.create' => __DIR__ . '/handlers/site.php',
'site.update' => __DIR__ . '/handlers/site.php',
'site.delete' => __DIR__ . '/handlers/site.php',
'site.setHome' => __DIR__ . '/handlers/site.php',

А не access.php.


---

Сразу дам тебе правильный полный api/index.php

Замени файл целиком.

/local/ТВОЯ_ПАПКА/api/index.php

<?php

require_once __DIR__ . '/bootstrap.php';

$action = (string)($_POST['action'] ?? '');

$map = [
    'ping' => __DIR__ . '/handlers/common.php',

    'site.list' => __DIR__ . '/handlers/site.php',
    'site.get' => __DIR__ . '/handlers/site.php',
    'site.create' => __DIR__ . '/handlers/site.php',
    'site.update' => __DIR__ . '/handlers/site.php',
    'site.delete' => __DIR__ . '/handlers/site.php',
    'site.setHome' => __DIR__ . '/handlers/site.php',

    'page.list' => __DIR__ . '/handlers/page.php',
    'page.create' => __DIR__ . '/handlers/page.php',
    'page.delete' => __DIR__ . '/handlers/page.php',
    'page.duplicate' => __DIR__ . '/handlers/page.php',
    'page.updateMeta' => __DIR__ . '/handlers/page.php',
    'page.setStatus' => __DIR__ . '/handlers/page.php',
    'page.setParent' => __DIR__ . '/handlers/page.php',
    'page.move' => __DIR__ . '/handlers/page.php',

    'block.list' => __DIR__ . '/handlers/block.php',
    'block.create' => __DIR__ . '/handlers/block.php',
    'block.update' => __DIR__ . '/handlers/block.php',
    'block.delete' => __DIR__ . '/handlers/block.php',
    'block.duplicate' => __DIR__ . '/handlers/block.php',
    'block.move' => __DIR__ . '/handlers/block.php',
    'block.reorder' => __DIR__ . '/handlers/block.php',

    'access.list' => __DIR__ . '/handlers/access.php',
    'access.set' => __DIR__ . '/handlers/access.php',
    'access.delete' => __DIR__ . '/handlers/access.php',

    'file.list' => __DIR__ . '/handlers/file.php',
    'file.upload' => __DIR__ . '/handlers/file.php',
    'file.delete' => __DIR__ . '/handlers/file.php',

    'menu.list' => __DIR__ . '/handlers/menu.php',
    'menu.create' => __DIR__ . '/handlers/menu.php',
    'menu.update' => __DIR__ . '/handlers/menu.php',
    'menu.delete' => __DIR__ . '/handlers/menu.php',
    'menu.setTop' => __DIR__ . '/handlers/menu.php',
    'menu.item.add' => __DIR__ . '/handlers/menu.php',
    'menu.item.update' => __DIR__ . '/handlers/menu.php',
    'menu.item.delete' => __DIR__ . '/handlers/menu.php',
    'menu.item.move' => __DIR__ . '/handlers/menu.php',

    'template.list' => __DIR__ . '/handlers/template.php',
    'template.createFromPage' => __DIR__ . '/handlers/template.php',
    'template.applyToPage' => __DIR__ . '/handlers/template.php',
    'template.rename' => __DIR__ . '/handlers/template.php',
    'template.delete' => __DIR__ . '/handlers/template.php',

    'layout.get' => __DIR__ . '/handlers/layout.php',
    'layout.updateSettings' => __DIR__ . '/handlers/layout.php',
    'layout.block.list' => __DIR__ . '/handlers/layout.php',
    'layout.block.create' => __DIR__ . '/handlers/layout.php',
    'layout.block.update' => __DIR__ . '/handlers/layout.php',
    'layout.block.delete' => __DIR__ . '/handlers/layout.php',
    'layout.block.move' => __DIR__ . '/handlers/layout.php',
];

if (!isset($map[$action])) {
    sb_json_error('UNKNOWN_ACTION', 400, ['action' => $action]);
}

require $map[$action];


---

Вторая обязательная проверка

Открой файл:

/local/ТВОЯ_ПАПКА/api/handlers/site.php

И убедись, что там именно код для site.*, а не случайно содержимое access.php.

Иногда при ручной сборке файлов бывает банальная ошибка: скопировали не тот файл не туда.


---

Правильный site.php

Если хочешь, просто замени его целиком на этот вариант.

/local/ТВОЯ_ПАПКА/api/handlers/site.php

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

    sb_json_ok(['sites' => $sites]);
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

    sb_json_ok(['site' => $site]);
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

    $base = $slug;
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
    ];
    sb_write_access($access);

    sb_json_ok(['site' => $site]);
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
    $pages = array_values(array_filter($pages, static function ($p) use ($id) {
        return (int)($p['siteId'] ?? 0) !== $id;
    }));
    sb_write_pages($pages);

    $pageIdsNow = [];
    foreach ($pages as $p) {
        $pageIdsNow[(int)($p['id'] ?? 0)] = true;
    }

    $blocks = sb_read_blocks();
    $blocks = array_values(array_filter($blocks, static function ($b) use ($pageIdsNow) {
        return isset($pageIdsNow[(int)($b['pageId'] ?? 0)]);
    }));
    sb_write_blocks($blocks);

    $access = sb_read_access();
    $access = array_values(array_filter($access, static function ($r) use ($id) {
        return (int)($r['siteId'] ?? 0) !== $id;
    }));
    sb_write_access($access);

    sb_json_ok();
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
    sb_json_ok();
}

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'site',
    'action' => $action,
]);


---

Почему это произошло

Судя по симптомам, у тебя одна из двух причин:

Вариант 1

В api/index.php action site.list замаплен не туда.

Вариант 2

В api/handlers/site.php лежит не тот код, а access.php или заглушка.

По твоему ответу в network это именно роутинг/handler issue, не backend-core.


---

Что должно быть после фикса

В test_api.php

Кнопка Ping:

{"ok":true,...}

Кнопка Site list:

{"ok":true,"sites":[]}

или массив сайтов.

В index.php

Вместо 501 должны появиться карточки сайтов или пустой список.


---

Что сделать прямо сейчас

1. Замени api/index.php целиком кодом выше


2. Проверь api/handlers/site.php


3. Обнови test_api.php


4. Нажми Site list



Если после этого будет уже другой ответ, пришли его — и пойдём дальше без гадания.