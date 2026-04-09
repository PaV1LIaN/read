Отлично. Дальше идём без переписывания всего проекта, а делаем первый безопасный шаг:

Что делаем сейчас

1. Создаём папки:



/local/sitebuilder/api

/local/sitebuilder/lib


2. Выносим в lib:



JSON storage

JSON response

ACL / роли


3. Ставим новый роутер:



api.php остаётся старой точкой входа для совместимости

внутри он просто подключает api/index.php


Это уже даст нормальную базу, и ничего не сломает на фронте.


---

1. Создай папки

/local/sitebuilder/api
/local/sitebuilder/lib


---

2. Новый файл /local/sitebuilder/lib/json.php

<?php

if (!function_exists('sb_data_path')) {
    function sb_data_path(string $file): string
    {
        return $_SERVER['DOCUMENT_ROOT'] . '/upload/sitebuilder/' . $file;
    }
}

if (!function_exists('sb_read_json_file')) {
    function sb_read_json_file(string $file): array
    {
        $path = sb_data_path($file);
        if (!file_exists($path)) {
            return [];
        }

        $fp = fopen($path, 'rb');
        if (!$fp) {
            return [];
        }

        $raw = '';
        if (flock($fp, LOCK_SH)) {
            $raw = stream_get_contents($fp);
            flock($fp, LOCK_UN);
        } else {
            $raw = stream_get_contents($fp);
        }

        fclose($fp);

        // Удаляем BOM, если вдруг файл с ним
        if (strncmp($raw, "\xEF\xBB\xBF", 3) === 0) {
            $raw = substr($raw, 3);
        }

        $data = json_decode((string)$raw, true);
        return is_array($data) ? $data : [];
    }
}

if (!function_exists('sb_write_json_file')) {
    function sb_write_json_file(string $file, array $data, string $errMsg): void
    {
        $dir = dirname(sb_data_path($file));
        if (!is_dir($dir)) {
            mkdir($dir, 0775, true);
        }

        $path = sb_data_path($file);
        $fp = fopen($path, 'c+');
        if (!$fp) {
            throw new RuntimeException($errMsg);
        }

        if (!flock($fp, LOCK_EX)) {
            fclose($fp);
            throw new RuntimeException('Cannot lock ' . $file);
        }

        ftruncate($fp, 0);
        rewind($fp);
        fwrite($fp, json_encode(array_values($data), JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT));
        fflush($fp);
        flock($fp, LOCK_UN);
        fclose($fp);
    }
}

if (!function_exists('sb_read_sites')) {
    function sb_read_sites(): array { return sb_read_json_file('sites.json'); }
}
if (!function_exists('sb_write_sites')) {
    function sb_write_sites(array $sites): void { sb_write_json_file('sites.json', $sites, 'Cannot open sites.json'); }
}

if (!function_exists('sb_read_pages')) {
    function sb_read_pages(): array { return sb_read_json_file('pages.json'); }
}
if (!function_exists('sb_write_pages')) {
    function sb_write_pages(array $pages): void { sb_write_json_file('pages.json', $pages, 'Cannot open pages.json'); }
}

if (!function_exists('sb_read_blocks')) {
    function sb_read_blocks(): array { return sb_read_json_file('blocks.json'); }
}
if (!function_exists('sb_write_blocks')) {
    function sb_write_blocks(array $blocks): void { sb_write_json_file('blocks.json', $blocks, 'Cannot open blocks.json'); }
}

if (!function_exists('sb_read_access')) {
    function sb_read_access(): array { return sb_read_json_file('access.json'); }
}
if (!function_exists('sb_write_access')) {
    function sb_write_access(array $access): void { sb_write_json_file('access.json', $access, 'Cannot open access.json'); }
}

if (!function_exists('sb_read_menus')) {
    function sb_read_menus(): array { return sb_read_json_file('menus.json'); }
}
if (!function_exists('sb_write_menus')) {
    function sb_write_menus(array $menus): void { sb_write_json_file('menus.json', $menus, 'Cannot open menus.json'); }
}

if (!function_exists('sb_read_templates')) {
    function sb_read_templates(): array { return sb_read_json_file('templates.json'); }
}
if (!function_exists('sb_write_templates')) {
    function sb_write_templates(array $templates): void { sb_write_json_file('templates.json', $templates, 'Cannot open templates.json'); }
}

if (!function_exists('sb_read_layouts')) {
    function sb_read_layouts(): array { return sb_read_json_file('layouts.json'); }
}
if (!function_exists('sb_write_layouts')) {
    function sb_write_layouts(array $layouts): void { sb_write_json_file('layouts.json', $layouts, 'Cannot open layouts.json'); }
}


---

3. Новый файл /local/sitebuilder/lib/response.php

<?php

if (!function_exists('sb_json_response')) {
    function sb_json_response(array $payload, int $status = 200): void
    {
        http_response_code($status);
        echo json_encode($payload, JSON_UNESCAPED_UNICODE);
        exit;
    }
}

if (!function_exists('sb_json_ok')) {
    function sb_json_ok(array $data = []): void
    {
        sb_json_response(array_merge(['ok' => true], $data), 200);
    }
}

if (!function_exists('sb_json_error')) {
    function sb_json_error(string $error, int $status = 400, array $extra = []): void
    {
        sb_json_response(array_merge([
            'ok' => false,
            'error' => $error,
        ], $extra), $status);
    }
}


---

4. Новый файл /local/sitebuilder/lib/access.php

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

5. Новый файл /local/sitebuilder/api/bootstrap.php

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


---

6. Новый файл /local/sitebuilder/api/index.php

Это новый диспетчер.

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

7. Создай папку /local/sitebuilder/api/handlers

Внутри пока сделаем заглушки, чтобы проект не падал.

/local/sitebuilder/api/handlers/common.php

<?php

if ($action === 'ping') {
    sb_json_ok(['pong' => true]);
}


---

/local/sitebuilder/api/handlers/site.php

<?php

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'site',
    'action' => $action,
]);


---

/local/sitebuilder/api/handlers/page.php

<?php

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'page',
    'action' => $action,
]);


---

/local/sitebuilder/api/handlers/block.php

<?php

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'block',
    'action' => $action,
]);


---

/local/sitebuilder/api/handlers/access.php

<?php

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'access',
    'action' => $action,
]);


---

/local/sitebuilder/api/handlers/file.php

<?php

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'file',
    'action' => $action,
]);


---

/local/sitebuilder/api/handlers/menu.php

<?php

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'menu',
    'action' => $action,
]);


---

/local/sitebuilder/api/handlers/template.php

<?php

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'template',
    'action' => $action,
]);


---

/local/sitebuilder/api/handlers/layout.php

<?php

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'layout',
    'action' => $action,
]);


---

8. Заменить текущий /local/sitebuilder/api.php

Старый api.php не удаляй, но на этом шаге его содержимое надо заменить на совместимый враппер.

Новый /local/sitebuilder/api.php

<?php

require_once __DIR__ . '/api/index.php';


---

9. Что это даст прямо сейчас

После этого:

фронт по-прежнему будет стучаться в /local/sitebuilder/api.php

но реально запросы будут идти через новый роутер

инфраструктура будет вынесена из монолита

ты сможешь переносить action-блоки по одному, а не переписывать 3678 строк разом



---

10. Важный момент

После замены api.php на враппер у тебя все действия, кроме ping, начнут отвечать NOT_MOVED_YET, пока мы не перенесём их в handlers.

Поэтому безопасный рабочий вариант такой:

Вариант А — правильный

Сначала:

создать lib/*

создать api/*

не трогать старый api.php

потом я тебе дам следующий код: как перенести site.* и page.list/page.create


Вариант Б — быстрый

Сразу заменить api.php на враппер и переносить обработчики пачками. Но в промежутке всё, что не перенесли, работать не будет.


---

11. Как сделать безопасно без простоя

Я рекомендую так:

Шаг 1

Создай:

lib/json.php

lib/response.php

lib/access.php

api/bootstrap.php

api/index.php

api/handlers/*


Шаг 2

Старый api.php пока не трогай

Шаг 3

Я следующим сообщением дам тебе:

готовый перенос site.list

site.get

site.create

site.delete

page.list


После этого уже можно будет переключать api.php на новый роутер.


---

12. Что делать прямо сейчас

Сделай сначала именно это:

1. создай папки api, api/handlers, lib


2. вставь файлы кодом выше


3. не заменяй пока старый api.php


4. напиши мне: “готово”



И следующим сообщением я пришлю тебе уже первый рабочий пакет обработчиков, которые можно вставить и начать реально переносить монолит.