Да, делаем в 2 этапа: сначала API для управления ролями, потом добавим блок в editor.php.

1. Замени /local/sitebuilder/api/index.php

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
    $action === 'site.setHome' ||
    $action === 'site.syncAccess' ||
    $action === 'site.ensureGroup' ||
    $action === 'site.accessList' ||
    $action === 'site.accessSet' ||
    $action === 'site.accessRemove'
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

if (strpos($action, 'page.') === 0) {
    require __DIR__ . '/handlers/page.php';
    exit;
}

sb_json_error('UNKNOWN_ACTION', 400, [
    'action' => $action,
    'file' => __FILE__,
]);


---

2. Создай файл /local/sitebuilder/lib/SiteAccessManagementService.php

<?php

use Bitrix\Main\Loader;

class SiteAccessManagementService
{
    public static function list(int $siteId): array
    {
        if ($siteId <= 0) {
            throw new RuntimeException('EMPTY_SITE_ID');
        }

        $access = sb_read_access();
        $items = [];

        foreach ($access as $row) {
            if ((int)($row['siteId'] ?? 0) !== $siteId) {
                continue;
            }

            $accessCode = (string)($row['accessCode'] ?? '');
            $role = (string)($row['role'] ?? '');

            if ($accessCode === '' || $role === '') {
                continue;
            }

            $userId = self::extractUserId($accessCode);

            $items[] = [
                'siteId' => $siteId,
                'accessCode' => $accessCode,
                'userId' => $userId,
                'userName' => $userId > 0 ? self::getUserName($userId) : '',
                'role' => $role,
                'createdBy' => (int)($row['createdBy'] ?? 0),
                'createdAt' => (string)($row['createdAt'] ?? ''),
                'updatedBy' => (int)($row['updatedBy'] ?? 0),
                'updatedAt' => (string)($row['updatedAt'] ?? ''),
            ];
        }

        usort($items, static function ($a, $b) {
            $rankA = function_exists('sb_role_rank') ? sb_role_rank($a['role'] ?? '') : 0;
            $rankB = function_exists('sb_role_rank') ? sb_role_rank($b['role'] ?? '') : 0;

            if ($rankA !== $rankB) {
                return $rankB <=> $rankA;
            }

            return (int)($a['userId'] ?? 0) <=> (int)($b['userId'] ?? 0);
        });

        return $items;
    }

    public static function setRole(int $siteId, int $targetUserId, string $role, int $currentUserId): array
    {
        if ($siteId <= 0) {
            throw new RuntimeException('EMPTY_SITE_ID');
        }

        if ($targetUserId <= 0) {
            throw new RuntimeException('EMPTY_TARGET_USER_ID');
        }

        if ($currentUserId <= 0) {
            throw new RuntimeException('EMPTY_CURRENT_USER_ID');
        }

        $role = self::normalizeRole($role);
        $accessCode = 'U' . $targetUserId;
        $access = sb_read_access();
        $now = date('c');

        $found = false;

        foreach ($access as &$row) {
            if (
                (int)($row['siteId'] ?? 0) === $siteId
                && (string)($row['accessCode'] ?? '') === $accessCode
            ) {
                $row['role'] = $role;
                $row['updatedAt'] = $now;
                $row['updatedBy'] = $currentUserId;
                $found = true;
                break;
            }
        }
        unset($row);

        if (!$found) {
            $access[] = [
                'siteId' => $siteId,
                'accessCode' => $accessCode,
                'role' => $role,
                'createdBy' => $currentUserId,
                'createdAt' => $now,
                'updatedAt' => $now,
                'updatedBy' => $currentUserId,
            ];
        }

        sb_write_access($access);

        $groupSync = self::syncUserToBitrixGroup($siteId, $targetUserId, $role, $currentUserId);

        return [
            'siteId' => $siteId,
            'accessCode' => $accessCode,
            'userId' => $targetUserId,
            'role' => $role,
            'created' => !$found,
            'updated' => $found,
            'groupSync' => $groupSync,
            'items' => self::list($siteId),
        ];
    }

    public static function removeRole(int $siteId, int $targetUserId, int $currentUserId): array
    {
        if ($siteId <= 0) {
            throw new RuntimeException('EMPTY_SITE_ID');
        }

        if ($targetUserId <= 0) {
            throw new RuntimeException('EMPTY_TARGET_USER_ID');
        }

        if ($currentUserId <= 0) {
            throw new RuntimeException('EMPTY_CURRENT_USER_ID');
        }

        $accessCode = 'U' . $targetUserId;
        $access = sb_read_access();

        $targetRole = null;
        $ownerCount = 0;

        foreach ($access as $row) {
            if ((int)($row['siteId'] ?? 0) !== $siteId) {
                continue;
            }

            $role = (string)($row['role'] ?? '');

            if ($role === 'OWNER') {
                $ownerCount++;
            }

            if ((string)($row['accessCode'] ?? '') === $accessCode) {
                $targetRole = $role;
            }
        }

        if ($targetRole === 'OWNER' && $ownerCount <= 1) {
            throw new RuntimeException('LAST_OWNER_CANNOT_BE_REMOVED');
        }

        $next = [];
        $removed = false;

        foreach ($access as $row) {
            if (
                (int)($row['siteId'] ?? 0) === $siteId
                && (string)($row['accessCode'] ?? '') === $accessCode
            ) {
                $removed = true;
                continue;
            }

            $next[] = $row;
        }

        sb_write_access($next);

        $groupSync = self::removeUserFromBitrixGroup($siteId, $targetUserId, $currentUserId);

        return [
            'siteId' => $siteId,
            'accessCode' => $accessCode,
            'userId' => $targetUserId,
            'removed' => $removed,
            'groupSync' => $groupSync,
            'items' => self::list($siteId),
        ];
    }

    protected static function normalizeRole(string $role): string
    {
        $role = strtoupper(trim($role));

        $allowed = [
            'VIEWER',
            'EDITOR',
            'ADMIN',
            'OWNER',
        ];

        if (!in_array($role, $allowed, true)) {
            throw new RuntimeException('INVALID_ROLE');
        }

        return $role;
    }

    protected static function extractUserId(string $accessCode): int
    {
        if (preg_match('/^U(\d+)$/', $accessCode, $m)) {
            return (int)$m[1];
        }

        return 0;
    }

    protected static function getUserName(int $userId): string
    {
        if ($userId <= 0 || !class_exists('CUser')) {
            return '';
        }

        $rs = \CUser::GetByID($userId);
        $row = $rs ? $rs->Fetch() : null;

        if (!$row) {
            return 'Пользователь #' . $userId;
        }

        $name = trim(
            (string)($row['LAST_NAME'] ?? '') . ' ' .
            (string)($row['NAME'] ?? '') . ' ' .
            (string)($row['SECOND_NAME'] ?? '')
        );

        if ($name === '') {
            $name = (string)($row['LOGIN'] ?? '');
        }

        if ($name === '') {
            $name = (string)($row['EMAIL'] ?? '');
        }

        return $name !== '' ? $name : ('Пользователь #' . $userId);
    }

    protected static function getSiteBitrixGroupId(int $siteId): int
    {
        $site = function_exists('sb_find_site') ? sb_find_site($siteId) : null;

        if (is_array($site)) {
            $groupId = (int)(
                $site['bitrixGroupId']
                ?? $site['bitrix_group_id']
                ?? 0
            );

            if ($groupId > 0) {
                return $groupId;
            }
        }

        if (function_exists('sb_db')) {
            $pdo = sb_db();

            $st = $pdo->prepare("
                SELECT bitrix_group_id
                FROM sitebuilder.site
                WHERE id = :site_id
                LIMIT 1
            ");

            $st->execute([
                ':site_id' => $siteId,
            ]);

            $row = $st->fetch(PDO::FETCH_ASSOC);

            if ($row) {
                return (int)($row['bitrix_group_id'] ?? 0);
            }
        }

        return 0;
    }

    protected static function syncUserToBitrixGroup(int $siteId, int $targetUserId, string $role, int $currentUserId): array
    {
        $groupId = self::getSiteBitrixGroupId($siteId);

        if ($groupId <= 0) {
            return [
                'ok' => false,
                'skipped' => true,
                'message' => 'SITE_HAS_NO_BITRIX_GROUP',
            ];
        }

        $sonetRole = self::mapSiteRoleToSonetRole($role);

        if ($sonetRole === '') {
            return [
                'ok' => true,
                'skipped' => true,
                'message' => 'OWNER_ROLE_IS_SITEBUILDER_ONLY',
            ];
        }

        try {
            if (!Loader::includeModule('socialnetwork')) {
                throw new RuntimeException('SOCIALNETWORK_MODULE_NOT_INSTALLED');
            }

            if (!class_exists('CSocNetUserToGroup')) {
                throw new RuntimeException('CSocNetUserToGroup_NOT_FOUND');
            }

            $membership = self::findGroupMembership($groupId, $targetUserId);

            if ($membership) {
                $membershipId = (int)$membership['ID'];
                $currentRole = (string)($membership['ROLE'] ?? '');

                if ($currentRole === $sonetRole) {
                    return [
                        'ok' => true,
                        'skipped' => false,
                        'action' => 'already_exists',
                        'groupId' => $groupId,
                        'sonetRole' => $sonetRole,
                    ];
                }

                $updated = \CSocNetUserToGroup::Update($membershipId, [
                    'ROLE' => $sonetRole,
                ]);

                if (!$updated) {
                    throw new RuntimeException('BITRIX_GROUP_MEMBER_UPDATE_ERROR');
                }

                return [
                    'ok' => true,
                    'skipped' => false,
                    'action' => 'updated',
                    'groupId' => $groupId,
                    'sonetRole' => $sonetRole,
                ];
            }

            $id = \CSocNetUserToGroup::Add([
                'USER_ID' => $targetUserId,
                'GROUP_ID' => $groupId,
                'ROLE' => $sonetRole,
                'INITIATED_BY_TYPE' => defined('SONET_INITIATED_BY_GROUP') ? SONET_INITIATED_BY_GROUP : 'G',
                'INITIATED_BY_USER_ID' => $currentUserId,
                'MESSAGE' => '',
                'SEND_MAIL' => 'N',
            ]);

            if (!$id) {
                throw new RuntimeException('BITRIX_GROUP_MEMBER_ADD_ERROR');
            }

            return [
                'ok' => true,
                'skipped' => false,
                'action' => 'created',
                'groupId' => $groupId,
                'sonetRole' => $sonetRole,
            ];
        } catch (Throwable $e) {
            return [
                'ok' => false,
                'skipped' => false,
                'groupId' => $groupId,
                'error' => $e->getMessage(),
            ];
        }
    }

    protected static function removeUserFromBitrixGroup(int $siteId, int $targetUserId, int $currentUserId): array
    {
        $groupId = self::getSiteBitrixGroupId($siteId);

        if ($groupId <= 0) {
            return [
                'ok' => false,
                'skipped' => true,
                'message' => 'SITE_HAS_NO_BITRIX_GROUP',
            ];
        }

        try {
            if (!Loader::includeModule('socialnetwork')) {
                throw new RuntimeException('SOCIALNETWORK_MODULE_NOT_INSTALLED');
            }

            if (!class_exists('CSocNetUserToGroup')) {
                throw new RuntimeException('CSocNetUserToGroup_NOT_FOUND');
            }

            $membership = self::findGroupMembership($groupId, $targetUserId);

            if (!$membership) {
                return [
                    'ok' => true,
                    'skipped' => true,
                    'message' => 'USER_NOT_IN_GROUP',
                    'groupId' => $groupId,
                ];
            }

            $membershipId = (int)$membership['ID'];
            $sonetRole = (string)($membership['ROLE'] ?? '');

            $ownerRole = defined('SONET_ROLES_OWNER') ? SONET_ROLES_OWNER : 'A';

            if ($sonetRole === $ownerRole || $sonetRole === 'A') {
                return [
                    'ok' => false,
                    'skipped' => true,
                    'message' => 'BITRIX_GROUP_OWNER_NOT_REMOVED',
                    'groupId' => $groupId,
                ];
            }

            $deleted = \CSocNetUserToGroup::Delete($membershipId);

            if (!$deleted) {
                throw new RuntimeException('BITRIX_GROUP_MEMBER_DELETE_ERROR');
            }

            return [
                'ok' => true,
                'skipped' => false,
                'action' => 'deleted',
                'groupId' => $groupId,
            ];
        } catch (Throwable $e) {
            return [
                'ok' => false,
                'skipped' => false,
                'groupId' => $groupId,
                'error' => $e->getMessage(),
            ];
        }
    }

    protected static function findGroupMembership(int $groupId, int $userId): ?array
    {
        $rs = \CSocNetUserToGroup::GetList(
            ['ID' => 'ASC'],
            [
                'GROUP_ID' => $groupId,
                'USER_ID' => $userId,
            ],
            false,
            false,
            [
                'ID',
                'USER_ID',
                'GROUP_ID',
                'ROLE',
            ]
        );

        if ($row = $rs->Fetch()) {
            return $row;
        }

        return null;
    }

    protected static function mapSiteRoleToSonetRole(string $role): string
    {
        $role = strtoupper(trim($role));

        $userRole = defined('SONET_ROLES_USER') ? SONET_ROLES_USER : 'K';
        $moderatorRole = defined('SONET_ROLES_MODERATOR') ? SONET_ROLES_MODERATOR : 'E';

        if ($role === 'VIEWER') {
            return $userRole;
        }

        if ($role === 'EDITOR' || $role === 'ADMIN') {
            return $moderatorRole;
        }

        /*
         * OWNER в SiteBuilder пока не делаем владельцем группы,
         * чтобы случайно не сменить владельца рабочей группы Битрикс24.
         */
        if ($role === 'OWNER') {
            return '';
        }

        return '';
    }
}


---

3. В site.php подключи сервис

В верхней части файла, рядом с подключением остальных сервисов, добавь:

$siteAccessManagementServicePath = $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/SiteAccessManagementService.php';

if (file_exists($siteAccessManagementServicePath)) {
    require_once $siteAccessManagementServicePath;
}


---

4. В site.php замени site.list

Если ещё не заменял, блок site.list сделай таким:

if ($action === 'site.list') {
    $sites = sb_read_sites();
    $myCode = sb_user_access_code();
    $allowedSites = [];

    foreach ($sites as $site) {
        $siteId = (int)($site['id'] ?? 0);

        if ($siteId <= 0) {
            continue;
        }

        if ($USER->IsAdmin()) {
            $role = 'OWNER';
        } else {
            $role = sb_get_role($siteId, $myCode);
        }

        if (sb_role_rank($role) < 1) {
            continue;
        }

        $site['currentUserRole'] = $role;
        $allowedSites[] = $site;
    }

    usort($allowedSites, static function ($a, $b) {
        return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
    });

    sb_json_ok([
        'sites' => $allowedSites,
        'handler' => 'site',
        'file' => __FILE__,
    ]);
}


---

5. В site.php добавь actions ролей

Вставь эти блоки перед финальным:

sb_json_error('NOT_MOVED_YET', 501, [

if ($action === 'site.accessList') {
    $siteId = (int)($_POST['siteId'] ?? 0);

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    if (!$USER->IsAdmin()) {
        sb_require_owner($siteId);
    }

    if (!class_exists('SiteAccessManagementService')) {
        sb_json_error('SiteAccessManagementService.php не подключен', 500);
    }

    try {
        sb_json_ok([
            'items' => SiteAccessManagementService::list($siteId),
            'handler' => 'site',
            'action' => 'site.accessList',
            'file' => __FILE__,
        ]);
    } catch (Throwable $e) {
        sb_json_error($e->getMessage(), 500, [
            'handler' => 'site',
            'action' => 'site.accessList',
            'file' => __FILE__,
        ]);
    }
}

if ($action === 'site.accessSet') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $userId = (int)($_POST['userId'] ?? 0);
    $role = strtoupper(trim((string)($_POST['role'] ?? '')));

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    if ($userId <= 0) {
        sb_json_error('USER_ID_REQUIRED', 422);
    }

    if ($role === '') {
        sb_json_error('ROLE_REQUIRED', 422);
    }

    if (!$USER->IsAdmin()) {
        sb_require_owner($siteId);
    }

    if (!class_exists('SiteAccessManagementService')) {
        sb_json_error('SiteAccessManagementService.php не подключен', 500);
    }

    try {
        $result = SiteAccessManagementService::setRole(
            $siteId,
            $userId,
            $role,
            (int)$USER->GetID()
        );

        sb_json_ok([
            'result' => $result,
            'items' => $result['items'] ?? [],
            'handler' => 'site',
            'action' => 'site.accessSet',
            'file' => __FILE__,
        ]);
    } catch (Throwable $e) {
        sb_json_error($e->getMessage(), 500, [
            'handler' => 'site',
            'action' => 'site.accessSet',
            'file' => __FILE__,
        ]);
    }
}

if ($action === 'site.accessRemove') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $userId = (int)($_POST['userId'] ?? 0);

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    if ($userId <= 0) {
        sb_json_error('USER_ID_REQUIRED', 422);
    }

    if (!$USER->IsAdmin()) {
        sb_require_owner($siteId);
    }

    if (!class_exists('SiteAccessManagementService')) {
        sb_json_error('SiteAccessManagementService.php не подключен', 500);
    }

    try {
        $result = SiteAccessManagementService::removeRole(
            $siteId,
            $userId,
            (int)$USER->GetID()
        );

        sb_json_ok([
            'result' => $result,
            'items' => $result['items'] ?? [],
            'handler' => 'site',
            'action' => 'site.accessRemove',
            'file' => __FILE__,
        ]);
    } catch (Throwable $e) {
        sb_json_error($e->getMessage(), 500, [
            'handler' => 'site',
            'action' => 'site.accessRemove',
            'file' => __FILE__,
        ]);
    }
}


---

6. Быстрая проверка через консоль

fetch('/local/sitebuilder/api.php', {
  method: 'POST',
  headers: {'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'},
  body: new URLSearchParams({
    action: 'site.accessList',
    siteId: '11',
    sessid: BX.bitrix_sessid()
  }),
  credentials: 'same-origin'
})
.then(r => r.json())
.then(console.log);

Выдать роль:

fetch('/local/sitebuilder/api.php', {
  method: 'POST',
  headers: {'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'},
  body: new URLSearchParams({
    action: 'site.accessSet',
    siteId: '11',
    userId: '15',
    role: 'EDITOR',
    sessid: BX.bitrix_sessid()
  }),
  credentials: 'same-origin'
})
.then(r => r.json())
.then(console.log);

После этого пользователь 15 должен появиться в sitebuilder.access, а если у сайта есть группа Битрикс24 — будет добавлен/обновлен в группе как модератор.