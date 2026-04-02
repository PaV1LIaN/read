Отлично. Тогда идём правильно: сначала заполняем эти 5 файлов и сразу облегчаем api.php.

Ниже — готовое содержимое для каждого файла.


---

lib/bootstrap.php

<?php
declare(strict_types=1);

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


---

lib/response.php

<?php
declare(strict_types=1);

function sb_ok(array $data = []): void
{
    echo json_encode(array_merge(['ok' => true], $data), JSON_UNESCAPED_UNICODE);
    exit;
}

function sb_error(int $status, string $error, array $extra = []): void
{
    http_response_code($status);
    echo json_encode(array_merge([
        'ok' => false,
        'error' => $error,
    ], $extra), JSON_UNESCAPED_UNICODE);
    exit;
}


---

lib/storage.php

<?php
declare(strict_types=1);

function sb_data_path(string $file): string
{
    return $_SERVER['DOCUMENT_ROOT'] . '/upload/sitebuilder/' . $file;
}

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
        $raw = (string)stream_get_contents($fp);
        flock($fp, LOCK_UN);
    } else {
        $raw = (string)stream_get_contents($fp);
    }
    fclose($fp);

    if (strncmp($raw, "\xEF\xBB\xBF", 3) === 0) {
        $raw = substr($raw, 3);
    }

    $data = json_decode($raw, true);
    return is_array($data) ? $data : [];
}

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

function sb_read_sites(): array { return sb_read_json_file('sites.json'); }
function sb_write_sites(array $sites): void { sb_write_json_file('sites.json', $sites, 'Cannot open sites.json'); }

function sb_read_pages(): array { return sb_read_json_file('pages.json'); }
function sb_write_pages(array $pages): void { sb_write_json_file('pages.json', $pages, 'Cannot open pages.json'); }

function sb_read_blocks(): array { return sb_read_json_file('blocks.json'); }
function sb_write_blocks(array $blocks): void { sb_write_json_file('blocks.json', $blocks, 'Cannot open blocks.json'); }

function sb_read_access(): array { return sb_read_json_file('access.json'); }
function sb_write_access(array $access): void { sb_write_json_file('access.json', $access, 'Cannot open access.json'); }

function sb_read_menus(): array { return sb_read_json_file('menus.json'); }
function sb_write_menus(array $menus): void { sb_write_json_file('menus.json', $menus, 'Cannot open menus.json'); }

function sb_read_templates(): array { return sb_read_json_file('templates.json'); }
function sb_write_templates(array $templates): void { sb_write_json_file('templates.json', $templates, 'Cannot open templates.json'); }

function sb_read_layouts(): array { return sb_read_json_file('layouts.json'); }
function sb_write_layouts(array $layouts): void { sb_write_json_file('layouts.json', $layouts, 'Cannot open layouts.json'); }


---

lib/access.php

<?php
declare(strict_types=1);

function sb_user_access_code(): string
{
    return 'U' . (int)$GLOBALS['USER']->GetID();
}

function sb_get_role(int $siteId, string $accessCode): ?string
{
    $access = sb_read_access();

    foreach ($access as $r) {
        if ((int)($r['siteId'] ?? 0) === $siteId && (string)($r['accessCode'] ?? '') === $accessCode) {
            $role = strtoupper((string)($r['role'] ?? ''));
            return $role !== '' ? $role : null;
        }
    }

    return null;
}

function sb_role_rank(?string $role): int
{
    $role = strtoupper((string)$role);

    return match ($role) {
        'OWNER'  => 4,
        'ADMIN'  => 3,
        'EDITOR' => 2,
        'VIEWER' => 1,
        default  => 0,
    };
}

function sb_require_site_role(int $siteId, int $minRank): void
{
    $role = sb_get_role($siteId, sb_user_access_code());

    if (sb_role_rank($role) < $minRank) {
        sb_error(403, 'FORBIDDEN', [
            'siteId' => $siteId,
            'role' => $role,
        ]);
    }
}

function sb_require_owner(int $siteId): void { sb_require_site_role($siteId, 4); }
function sb_require_admin(int $siteId): void { sb_require_site_role($siteId, 3); }
function sb_require_editor(int $siteId): void { sb_require_site_role($siteId, 2); }
function sb_require_viewer(int $siteId): void { sb_require_site_role($siteId, 1); }

function sb_site_exists(int $siteId): bool
{
    foreach (sb_read_sites() as $s) {
        if ((int)($s['id'] ?? 0) === $siteId) {
            return true;
        }
    }
    return false;
}

function sb_find_site(int $siteId): ?array
{
    foreach (sb_read_sites() as $s) {
        if ((int)($s['id'] ?? 0) === $siteId) {
            return $s;
        }
    }
    return null;
}

function sb_find_page(int $pageId): ?array
{
    foreach (sb_read_pages() as $p) {
        if ((int)($p['id'] ?? 0) === $pageId) {
            return $p;
        }
    }
    return null;
}

function sb_find_block(int $blockId): ?array
{
    foreach (sb_read_blocks() as $b) {
        if ((int)($b['id'] ?? 0) === $blockId) {
            return $b;
        }
    }
    return null;
}


---

lib/slug.php

<?php
declare(strict_types=1);

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


---

Что сделать в api.php сейчас

В самый верх api.php вместо старого bootstrap и этих helper-функций вставь:

<?php
declare(strict_types=1);

require __DIR__ . '/lib/bootstrap.php';
require __DIR__ . '/lib/response.php';
require __DIR__ . '/lib/storage.php';
require __DIR__ . '/lib/access.php';
require __DIR__ . '/lib/slug.php';

И после этого из api.php можно удалить:

весь старый bootstrap

старые функции из этих 5 файлов:

JSON storage

access helpers

slugify

ручные echo json_encode(...); exit; пока можно не все удалять, но новые места уже лучше писать через sb_ok() и sb_error()




---

Чтобы не сломать проект

Сделай это по порядку:

1. Заполни 5 файлов кодом выше.


2. В api.php подключи их через require.


3. Не удаляй пока сразу все старые функции.


4. Сначала убедись, что api.php вообще открывается без fatal error.


5. Потом начинай удалять дубли.




---

Самый безопасный следующий шаг

После этого я дам тебе следующий набор файлов:

lib/menu.php

lib/layout.php


А потом уже вынесем disk-слой и начнём резать api.php на action-файлы.

Когда вставишь это, напиши, и пойдём дальше.