Отлично, теперь видно главное:

api/index.php — правильный

api/handlers/common.php — правильный

api/handlers/access.php — отдельный файл и тоже выглядит нормально

но проблема всё ещё остаётся


Значит, почти наверняка ошибка уже не в api/handlers/access.php, а в одном из подключаемых lib-файлов, скорее всего:

lib/access.php

реже api/bootstrap.php


Потому что:

bootstrap.php всегда подключает lib/access.php

если ты случайно перезаписал lib/access.php кодом handler-а, то при каждом запросе он будет сразу отдавать: {"ok":false,"error":"NOT_MOVED_YET","handler":"access","action":"ping"}


И это идеально совпадает с твоим симптомом.


---

Что проверить прямо сейчас

Снова используй диагностический api.php, но теперь посмотри bootstrap и lib.

Замени /local/sitebuilder/api.php на это

<?php
header('Content-Type: application/json; charset=UTF-8');

$files = [
    'api_bootstrap' => __DIR__ . '/api/bootstrap.php',
    'lib_json'      => __DIR__ . '/lib/json.php',
    'lib_response'  => __DIR__ . '/lib/response.php',
    'lib_access'    => __DIR__ . '/lib/access.php',
    'lib_helpers'   => __DIR__ . '/lib/helpers.php',
    'lib_disk'      => __DIR__ . '/lib/disk.php',
];

$result = [
    'ok' => true,
    'apiFile' => __FILE__,
    'time' => date('c'),
    'files' => [],
];

foreach ($files as $key => $path) {
    $exists = file_exists($path);
    $content = $exists ? file_get_contents($path) : '';
    $result['files'][$key] = [
        'path' => $path,
        'exists' => $exists,
        'md5' => $exists ? md5_file($path) : null,
        'size' => $exists ? filesize($path) : null,
        'head' => $exists ? substr($content, 0, 600) : null,
    ];
}

echo json_encode($result, JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT);
exit;

Открой:

https://portal24.itsnn.ru/local/sitebuilder/api.php

и пришли JSON.


---

Что я ожидаю увидеть

Нормальный lib/access.php должен начинаться примерно так:

<?php

require_once __DIR__ . '/json.php';
require_once __DIR__ . '/response.php';

if (!function_exists('sb_user_access_code')) {

Если вместо этого там будет что-то вроде:

global $USER;

if ($action === 'access.list') {

то всё, причина найдена:
ты случайно записал handler-код в lib/access.php.


---

Если не хочешь ждать диагностику — вот правильный lib/access.php

Можешь сразу сравнить или просто заменить.

/local/sitebuilder/lib/access.php

<?php

require_once __DIR__ . '/json.php';
require_once __DIR__ . '/response.php';

if (!function_exists('sb_user_access_code')) {
    function sb_user_access_code(): string
    {
        global $USER;
        return 'U' . (int)$USER->GetID();
    }
}

if (!function_exists('sb_get_role')) {
    function sb_get_role(int $siteId, string $accessCode): ?string
    {
        $access = sb_read_access();

        foreach ($access as $row) {
            if (
                (int)($row['siteId'] ?? 0) === $siteId
                && (string)($row['accessCode'] ?? '') === $accessCode
            ) {
                return (string)($row['role'] ?? '');
            }
        }

        return null;
    }
}

if (!function_exists('sb_role_rank')) {
    function sb_role_rank(?string $role): int
    {
        switch ((string)$role) {
            case 'VIEWER':
                return 1;
            case 'EDITOR':
                return 2;
            case 'ADMIN':
                return 3;
            case 'OWNER':
                return 4;
            default:
                return 0;
        }
    }
}

if (!function_exists('sb_require_site_role')) {
    function sb_require_site_role(int $siteId, int $minRank): void
    {
        $role = sb_get_role($siteId, sb_user_access_code());

        if (sb_role_rank($role) < $minRank) {
            sb_json_error('ACCESS_DENIED', 403);
        }
    }
}

if (!function_exists('sb_require_owner')) {
    function sb_require_owner(int $siteId): void
    {
        sb_require_site_role($siteId, 4);
    }
}

if (!function_exists('sb_require_admin')) {
    function sb_require_admin(int $siteId): void
    {
        sb_require_site_role($siteId, 3);
    }
}

if (!function_exists('sb_require_editor')) {
    function sb_require_editor(int $siteId): void
    {
        sb_require_site_role($siteId, 2);
    }
}

if (!function_exists('sb_require_viewer')) {
    function sb_require_viewer(int $siteId): void
    {
        sb_require_site_role($siteId, 1);
    }
}


---

И правильный api/bootstrap.php

На всякий случай тоже сверяй. Он должен быть таким:

/local/sitebuilder/api/bootstrap.php

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

$projectRoot = dirname(__DIR__);

require_once $projectRoot . '/lib/json.php';
require_once $projectRoot . '/lib/response.php';
require_once $projectRoot . '/lib/access.php';
require_once $projectRoot . '/lib/helpers.php';
require_once $projectRoot . '/lib/disk.php';


---

Что делать после исправления

После того как проверишь/исправишь lib/access.php, сделай:

1. верни api.php обратно:



<?php
ini_set('display_errors', '1');
error_reporting(E_ALL);

require_once __DIR__ . '/api/index.php';

2. открой opcache_reset.php


3. снова нажми Ping




---

Что должно прийти

Ожидаемый ответ:

{
  "ok": true,
  "pong": true,
  "handler": "common",
  "file": "/srv/bx/docroot/local/sitebuilder/api/handlers/common.php"
}

Если хочешь, можешь сразу не делать диагностику, а просто:

заменить lib/access.php кодом выше

вернуть api.php

сбросить opcache

проверить Ping


И прислать результат.