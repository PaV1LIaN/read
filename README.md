Отлично. Делаем access-пакет.

Сейчас закроем:

access.list

access.set

access.delete


Логика будет такая:

доступы хранятся в access.json

одна запись = один siteId + accessCode

при set запись либо создаётся, либо обновляется

владельца (OWNER) через обычный access.set менять не даём

удалить последнего владельца тоже не даём



---

1. Обнови /local/sitebuilder/lib/helpers.php

Добавь в конец файла:

<?php

if (!function_exists('sb_find_access_row')) {
    function sb_find_access_row(int $siteId, string $accessCode): ?array
    {
        foreach (sb_read_access() as $row) {
            if (
                (int)($row['siteId'] ?? 0) === $siteId
                && (string)($row['accessCode'] ?? '') === $accessCode
            ) {
                return $row;
            }
        }

        return null;
    }
}

if (!function_exists('sb_access_rows_for_site')) {
    function sb_access_rows_for_site(int $siteId): array
    {
        return array_values(array_filter(sb_read_access(), static function ($row) use ($siteId) {
            return (int)($row['siteId'] ?? 0) === $siteId;
        }));
    }
}

if (!function_exists('sb_normalize_access_role')) {
    function sb_normalize_access_role(string $role): string
    {
        $role = strtoupper(trim($role));

        if (!in_array($role, ['VIEWER', 'EDITOR', 'ADMIN', 'OWNER'], true)) {
            return '';
        }

        return $role;
    }
}

if (!function_exists('sb_count_site_owners')) {
    function sb_count_site_owners(int $siteId): int
    {
        $count = 0;

        foreach (sb_read_access() as $row) {
            if (
                (int)($row['siteId'] ?? 0) === $siteId
                && (string)($row['role'] ?? '') === 'OWNER'
            ) {
                $count++;
            }
        }

        return $count;
    }
}


---

2. Полностью замени /local/sitebuilder/api/handlers/access.php

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

    sb_json_ok();
}

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'access',
    'action' => $action,
]);


---

3. Что теперь должно работать

После вставки:

access.list

access.set

access.delete



---

4. Как использовать accessCode

Сейчас логика завязана на том же формате, что и раньше:

пользователь: U<ID>, например U1

дальше при необходимости можно добавить группы, но пока не усложняем


Примеры:

U1

U25



---

5. Какие роли поддерживаются

Только:

VIEWER

EDITOR

ADMIN

OWNER



---

6. Что намеренно запрещено

Сейчас я специально запретил через обычный API:

назначать OWNER через access.set

понижать OWNER до другой роли

удалять OWNER


Почему так:

владелец сейчас создаётся автоматически при site.create

это защищает от случайного “отрубания” себе доступа


Если позже захочешь, можно будет сделать отдельный action:

access.transferOwnership


Но сейчас лучше не добавлять это, чтобы не сломать права.


---

7. Что проверить руками

Проверь по порядку:

1. access.list


2. access.set для нового U<ID> с ролью VIEWER


3. изменить этому же пользователю роль на EDITOR


4. изменить на ADMIN


5. access.delete для этого пользователя


6. попробовать поставить OWNER через access.set — должна быть ошибка


7. попробовать удалить владельца — должна быть ошибка




---

8. Что дальше лучше делать

Сейчас у тебя уже закрыты:

site

page

block

menu

access


Самый логичный следующий шаг:

1. template


2. layout


3. file / Disk



Я бы следующим делом делал template-пакет, потому что он ближе к редактору страниц и обычно проще, чем Disk.

Напиши: давай template-пакет