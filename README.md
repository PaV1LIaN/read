Теперь картина ясная: у тебя реально подключается не тот handler-код, и это уже не проблема AJAX.

Вот ключевая улика:

{"ok":false,"error":"NOT_MOVED_YET","handler":"access","action":"ping"}

ping вообще не должен попадать в access.php. Значит у тебя одно из двух:

перепутано содержимое файлов в api/handlers

либо api/index.php не тот, который мы ожидаем

либо PHP держит старые файлы через opcache


Чтобы не гадать, сейчас делаем жёсткий сброс на минимально рабочую версию.


---

1. Полностью замени /local/sitebuilder/api/index.php

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
    $action === 'access.list' ||
    $action === 'access.set' ||
    $action === 'access.delete'
) {
    require __DIR__ . '/handlers/access.php';
    exit;
}

if (
    $action === 'template.list' ||
    $action === 'template.createFromPage' ||
    $action === 'template.applyToPage' ||
    $action === 'template.rename' ||
    $action === 'template.delete'
) {
    require __DIR__ . '/handlers/template.php';
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

if (
    $action === 'file.list' ||
    $action === 'file.upload' ||
    $action === 'file.delete'
) {
    require __DIR__ . '/handlers/file.php';
    exit;
}

sb_json_error('UNKNOWN_ACTION', 400, ['action' => $action]);


---

2. Полностью замени /local/sitebuilder/api/handlers/common.php

<?php

global $USER;

if ($action === 'ping') {
    sb_json_ok([
        'pong' => true,
        'time' => date('c'),
        'userId' => (int)$USER->GetID(),
        'login' => (string)$USER->GetLogin(),
        'handler' => 'common',
        'file' => __FILE__,
    ]);
}

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'common',
    'action' => $action,
    'file' => __FILE__,
]);


---

3. Полностью замени /local/sitebuilder/api/handlers/access.php

<?php

global $USER;

if ($action === 'access.list') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    sb_require_admin($siteId);

    $rows = sb_access_rows_for_site($siteId);

    usort($rows, static function ($a, $b) {
        $roleCmp = sb_role_rank((string)($b['role'] ?? '')) <=> sb_role_rank((string)($a['role'] ?? ''));
        if ($roleCmp !== 0) {
            return $roleCmp;
        }

        return strcmp((string)($a['accessCode'] ?? ''), (string)($b['accessCode'] ?? ''));
    });

    sb_json_ok([
        'access' => $rows,
        'handler' => 'access',
        'file' => __FILE__,
    ]);
}

if ($action === 'access.set') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $accessCode = trim((string)($_POST['accessCode'] ?? ''));
    $role = sb_normalize_access_role((string)($_POST['role'] ?? ''));

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }
    if ($accessCode === '') {
        sb_json_error('ACCESS_CODE_REQUIRED', 422);
    }
    if ($role === '') {
        sb_json_error('BAD_ROLE', 422);
    }

    sb_require_admin($siteId);

    if (!sb_site_exists($siteId)) {
        sb_json_error('SITE_NOT_FOUND', 404);
    }

    $current = sb_find_access_row($siteId, $accessCode);

    if ($current && (string)($current['role'] ?? '') === 'OWNER' && $role !== 'OWNER') {
        sb_json_error('CANNOT_DOWNGRADE_OWNER', 422);
    }

    if ($role === 'OWNER') {
        sb_json_error('OWNER_ASSIGNMENT_FORBIDDEN', 422);
    }

    $rows = sb_read_access();
    $updated = null;
    $found = false;

    foreach ($rows as &$row) {
        if (
            (int)($row['siteId'] ?? 0) === $siteId
            && (string)($row['accessCode'] ?? '') === $accessCode
        ) {
            $row['role'] = $role;
            $row['updatedAt'] = date('c');
            $row['updatedBy'] = (int)$USER->GetID();
            $updated = $row;
            $found = true;
            break;
        }
    }
    unset($row);

    if (!$found) {
        $updated = [
            'siteId' => $siteId,
            'accessCode' => $accessCode,
            'role' => $role,
            'createdBy' => (int)$USER->GetID(),
            'createdAt' => date('c'),
            'updatedAt' => date('c'),
            'updatedBy' => (int)$USER->GetID(),
        ];

        $rows[] = $updated;
    }

    sb_write_access($rows);

    sb_json_ok([
        'accessRow' => $updated,
        'handler' => 'access',
        'file' => __FILE__,
    ]);
}

if ($action === 'access.delete') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $accessCode = trim((string)($_POST['accessCode'] ?? ''));

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }
    if ($accessCode === '') {
        sb_json_error('ACCESS_CODE_REQUIRED', 422);
    }

    sb_require_admin($siteId);

    $row = sb_find_access_row($siteId, $accessCode);
    if (!$row) {
        sb_json_error('ACCESS_NOT_FOUND', 404);
    }

    $role = (string)($row['role'] ?? '');
    if ($role === 'OWNER') {
        if (sb_count_site_owners($siteId) <= 1) {
            sb_json_error('CANNOT_DELETE_LAST_OWNER', 422);
        }
        sb_json_error('OWNER_DELETE_FORBIDDEN', 422);
    }

    $rows = sb_read_access();
    $before = count($rows);

    $rows = array_values(array_filter($rows, static function ($r) use ($siteId, $accessCode) {
        return !(
            (int)($r['siteId'] ?? 0) === $siteId
            && (string)($r['accessCode'] ?? '') === $accessCode
        );
    }));

    if (count($rows) === $before) {
        sb_json_error('ACCESS_NOT_FOUND', 404);
    }

    sb_write_access($rows);

    sb_json_ok([
        'handler' => 'access',
        'file' => __FILE__,
    ]);
}

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'access',
    'action' => $action,
    'file' => __FILE__,
]);


---

4. Полностью замени /local/sitebuilder/api.php

<?php

ini_set('display_errors', '1');
error_reporting(E_ALL);

require_once __DIR__ . '/api/index.php';


---

5. Сбрось opcache

Создай временный файл /local/sitebuilder/opcache_reset.php:

<?php
header('Content-Type: text/plain; charset=UTF-8');

if (function_exists('opcache_reset')) {
    var_dump(opcache_reset());
} else {
    echo "opcache_reset unavailable";
}

Открой его в браузере один раз.

Если увидишь:

bool(true)

значит opcache сброшен.


---

6. Что должно быть после этого

В test_api.php кнопка Ping

должна вернуть примерно это:

{
  "ok": true,
  "pong": true,
  "handler": "common",
  "file": "/srv/bx/docroot/local/sitebuilder/api/handlers/common.php"
}

Если всё ещё придёт:

{"ok":false,"error":"NOT_MOVED_YET","handler":"access","action":"ping"}

значит PHP всё ещё исполняет старый файл из opcache, или в common.php у тебя лежит не тот код.


---

7. Самая важная проверка после замены

Открой напрямую эти файлы в редакторе и посмотри, что внутри:

/srv/bx/docroot/local/sitebuilder/api/index.php

/srv/bx/docroot/local/sitebuilder/api/handlers/common.php

/srv/bx/docroot/local/sitebuilder/api/handlers/access.php


Потому что по симптомам у тебя либо реально перепутаны содержимые файлов, либо не сброшен opcache.


---

8. Что пришли мне следующим сообщением

Только два результата:

1. что вернул opcache_reset.php


2. что вернула кнопка Ping после замены файлов



Тогда я скажу уже точный следующий шаг.