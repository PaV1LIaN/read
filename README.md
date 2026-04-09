Отлично. Тогда делаем первый реально рабочий пакет:

ping

site.list

site.get

site.create

site.delete

site.setHome

page.list

page.create


Смысл такой: ты уже сможешь подключить новый API и проверить, что основа живая.


---

1. Добавь новый файл /local/sitebuilder/lib/helpers.php

<?php

require_once __DIR__ . '/json.php';

if (!function_exists('sb_slugify')) {
    function sb_slugify(string $name): string
    {
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
}

if (!function_exists('sb_site_exists')) {
    function sb_site_exists(int $siteId): bool
    {
        foreach (sb_read_sites() as $s) {
            if ((int)($s['id'] ?? 0) === $siteId) {
                return true;
            }
        }
        return false;
    }
}

if (!function_exists('sb_find_site')) {
    function sb_find_site(int $siteId): ?array
    {
        foreach (sb_read_sites() as $s) {
            if ((int)($s['id'] ?? 0) === $siteId) {
                return $s;
            }
        }
        return null;
    }
}

if (!function_exists('sb_find_page')) {
    function sb_find_page(int $pageId): ?array
    {
        foreach (sb_read_pages() as $p) {
            if ((int)($p['id'] ?? 0) === $pageId) {
                return $p;
            }
        }
        return null;
    }
}

if (!function_exists('sb_find_block')) {
    function sb_find_block(int $blockId): ?array
    {
        foreach (sb_read_blocks() as $b) {
            if ((int)($b['id'] ?? 0) === $blockId) {
                return $b;
            }
        }
        return null;
    }
}


---

2. Обнови /local/sitebuilder/api/bootstrap.php

Нужно просто подключить новый helpers.php.

<?php

define('NO_KEEP_STATISTIC', true);
define('NO_AGENT_STATISTIC', true);
define('DisableEventsCheck', true);

require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

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

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/json.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/response.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/access.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/helpers.php';


---

3. Замени /local/sitebuilder/api/handlers/common.php

<?php

if ($action === 'ping') {
    global $USER;

    sb_json_ok([
        'time' => date('c'),
        'userId' => (int)$USER->GetID(),
        'login' => (string)$USER->GetLogin(),
    ]);
}


---

4. Замени /local/sitebuilder/api/handlers/site.php

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

5. Замени /local/sitebuilder/api/handlers/page.php

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
    foreach ($pages as $p) {
        $maxId = max($maxId, (int)($p['id'] ?? 0));
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

    $base = $slug;
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
        'sort' => 500,
        'status' => 'draft',
        'publishedAt' => '',
        'createdBy' => (int)$USER->GetID(),
        'createdAt' => date('c'),
    ];

    $pages[] = $page;
    sb_write_pages($pages);

    sb_json_ok(['page' => $page]);
}

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'page',
    'action' => $action,
]);


---

6. Пока остальные handlers оставь заглушками

Вот так и оставляй, это нормально на данном этапе:

block.php

access.php

file.php

menu.php

template.php

layout.php



---

7. Теперь можно переключить точку входа

Если у тебя старая папка отдельно, а новая чистая папка — отдельно, то в новом проекте уже можно сделать так:

/local/sitebuilder/api.php

<?php
require_once __DIR__ . '/api/index.php';


---

8. Что уже можно проверить

Сейчас у тебя должны заработать:

Проверка API

action=ping

Сайты

site.list

site.get

site.create

site.delete

site.setHome


Страницы

page.list

page.create



---

9. Как быстро проверить руками

Через твой фронт или через временный тест.

Пример POST:

action=site.list

sessid=...


или:

action=site.create

name=Тестовый сайт

slug=test-site

sessid=...



---

10. На что обратить внимание

1. Путь до проекта

Я в коде использую: /local/sitebuilder/...

Если новая чистая папка у тебя называется не sitebuilder, а иначе — замени путь в:

bootstrap.php


Вот эти строки:

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/json.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/response.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/access.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/helpers.php';

2. Данные всё ещё хранятся в:

/upload/sitebuilder/

То есть новая чистая папка проекта использует старые JSON-данные. Это удобно для миграции.


---

11. Что делаем следующим сообщением

После того как вставишь это, следующий логичный шаг:

page.delete

page.duplicate

page.updateMeta

page.setStatus

page.setParent

page.move


Это лучше сделать следующим пакетом, потому что page-ветка у тебя уже почти замкнётся целиком.

Напиши мне одним сообщением: “вставил, давай page-пакет дальше”
и я пришлю тебе следующий готовый код.