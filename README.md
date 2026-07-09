Да, верно. Тогда нужно не через Bitrix Application::getConnection(), а через:

/local/sitebuilder/lib/db.php

У тебя там уже есть:

sb_db(): PDO

Значит переделываем 2 файла:

/local/sitebuilder/lib/PageAccessRepository.php
/local/sitebuilder/lib/PageAccessService.php


---

1. Замени PageAccessRepository.php

Файл:

/local/sitebuilder/lib/PageAccessRepository.php

Полностью замени на это:

<?php

require_once __DIR__ . '/db.php';

class PageAccessRepository
{
    public static function userAccessCode(int $userId): string
    {
        return 'U' . $userId;
    }

    public static function normalizeAccessCode(string $accessCode): string
    {
        $accessCode = trim($accessCode);
        $accessCode = mb_strtoupper($accessCode);

        if ($accessCode === '') {
            throw new RuntimeException('EMPTY_ACCESS_CODE');
        }

        if (!preg_match('/^[A-Z]\d+$/', $accessCode)) {
            throw new RuntimeException('INVALID_ACCESS_CODE');
        }

        return $accessCode;
    }

    public static function listByPage(int $siteId, int $pageId): array
    {
        if ($siteId <= 0 || $pageId <= 0) {
            return [];
        }

        $pdo = sb_db();

        $stmt = $pdo->prepare("
            SELECT
                id,
                site_id,
                page_id,
                access_code,
                can_view,
                can_edit,
                include_children,
                created_by,
                created_at,
                updated_at
            FROM sitebuilder.page_access
            WHERE site_id = :site_id
              AND page_id = :page_id
            ORDER BY id DESC
        ");

        $stmt->execute([
            ':site_id' => $siteId,
            ':page_id' => $pageId,
        ]);

        $rows = $stmt->fetchAll(PDO::FETCH_ASSOC);

        return array_map([self::class, 'mapRow'], $rows ?: []);
    }

    public static function save(
        int $siteId,
        int $pageId,
        string $accessCode,
        bool $canView,
        bool $canEdit,
        bool $includeChildren,
        int $createdBy = 0
    ): array {
        if ($siteId <= 0) {
            throw new RuntimeException('INVALID_SITE_ID');
        }

        if ($pageId <= 0) {
            throw new RuntimeException('INVALID_PAGE_ID');
        }

        $accessCode = self::normalizeAccessCode($accessCode);

        if ($canEdit) {
            $canView = true;
        }

        $pdo = sb_db();

        $stmt = $pdo->prepare("
            INSERT INTO sitebuilder.page_access (
                site_id,
                page_id,
                access_code,
                can_view,
                can_edit,
                include_children,
                created_by,
                created_at,
                updated_at
            )
            VALUES (
                :site_id,
                :page_id,
                :access_code,
                :can_view,
                :can_edit,
                :include_children,
                :created_by,
                NOW(),
                NOW()
            )
            ON CONFLICT (site_id, page_id, access_code)
            DO UPDATE SET
                can_view = EXCLUDED.can_view,
                can_edit = EXCLUDED.can_edit,
                include_children = EXCLUDED.include_children,
                updated_at = NOW()
            RETURNING
                id,
                site_id,
                page_id,
                access_code,
                can_view,
                can_edit,
                include_children,
                created_by,
                created_at,
                updated_at
        ");

        $stmt->execute([
            ':site_id' => $siteId,
            ':page_id' => $pageId,
            ':access_code' => $accessCode,
            ':can_view' => $canView,
            ':can_edit' => $canEdit,
            ':include_children' => $includeChildren,
            ':created_by' => $createdBy > 0 ? $createdBy : null,
        ]);

        $row = $stmt->fetch(PDO::FETCH_ASSOC);

        if (!$row) {
            throw new RuntimeException('PAGE_ACCESS_SAVE_ERROR');
        }

        return self::mapRow($row);
    }

    public static function delete(int $id, int $siteId = 0, int $pageId = 0): bool
    {
        if ($id <= 0) {
            throw new RuntimeException('INVALID_PAGE_ACCESS_ID');
        }

        $pdo = sb_db();

        $where = ['id = :id'];
        $params = [
            ':id' => $id,
        ];

        if ($siteId > 0) {
            $where[] = 'site_id = :site_id';
            $params[':site_id'] = $siteId;
        }

        if ($pageId > 0) {
            $where[] = 'page_id = :page_id';
            $params[':page_id'] = $pageId;
        }

        $stmt = $pdo->prepare("
            DELETE FROM sitebuilder.page_access
            WHERE " . implode(' AND ', $where)
        );

        $stmt->execute($params);

        return true;
    }

    public static function hasAnyPageAccess(int $siteId, string $accessCode): bool
    {
        if ($siteId <= 0) {
            return false;
        }

        $accessCode = self::normalizeAccessCode($accessCode);

        $pdo = sb_db();

        $stmt = $pdo->prepare("
            SELECT id
            FROM sitebuilder.page_access
            WHERE site_id = :site_id
              AND access_code = :access_code
            LIMIT 1
        ");

        $stmt->execute([
            ':site_id' => $siteId,
            ':access_code' => $accessCode,
        ]);

        return (bool)$stmt->fetch(PDO::FETCH_ASSOC);
    }

    public static function hasPagePermission(
        int $siteId,
        int $pageId,
        string $accessCode,
        string $permission
    ): bool {
        if ($siteId <= 0 || $pageId <= 0) {
            return false;
        }

        $accessCode = self::normalizeAccessCode($accessCode);

        if (!in_array($permission, ['view', 'edit'], true)) {
            return false;
        }

        $pageAndParents = self::getPageAndParentIds($siteId, $pageId);

        if (empty($pageAndParents)) {
            return false;
        }

        $placeholders = [];

        foreach ($pageAndParents as $index => $id) {
            $placeholders[] = ':page_id_' . $index;
        }

        $pdo = sb_db();

        $params = [
            ':site_id' => $siteId,
            ':access_code' => $accessCode,
        ];

        foreach ($pageAndParents as $index => $id) {
            $params[':page_id_' . $index] = (int)$id;
        }

        $stmt = $pdo->prepare("
            SELECT
                page_id,
                can_view,
                can_edit,
                include_children
            FROM sitebuilder.page_access
            WHERE site_id = :site_id
              AND access_code = :access_code
              AND page_id IN (" . implode(',', $placeholders) . ")
        ");

        $stmt->execute($params);

        $rulesByPageId = [];

        while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
            $rulesByPageId[(int)$row['page_id']] = [
                'canView' => self::boolValue($row['can_view']),
                'canEdit' => self::boolValue($row['can_edit']),
                'includeChildren' => self::boolValue($row['include_children']),
            ];
        }

        foreach ($pageAndParents as $index => $currentPageId) {
            $currentPageId = (int)$currentPageId;

            if (!isset($rulesByPageId[$currentPageId])) {
                continue;
            }

            $rule = $rulesByPageId[$currentPageId];

            /*
             * index = 0 — сама страница.
             * index > 0 — родительская страница.
             */
            $isDirectPage = $index === 0;

            if (!$isDirectPage && !$rule['includeChildren']) {
                continue;
            }

            if ($permission === 'view' && ($rule['canView'] || $rule['canEdit'])) {
                return true;
            }

            if ($permission === 'edit' && $rule['canEdit']) {
                return true;
            }
        }

        return false;
    }

    public static function getPageAndParentIds(int $siteId, int $pageId): array
    {
        if ($siteId <= 0 || $pageId <= 0) {
            return [];
        }

        $pdo = sb_db();

        $ids = [];
        $visited = [];
        $currentPageId = $pageId;

        for ($i = 0; $i < 100; $i++) {
            if ($currentPageId <= 0) {
                break;
            }

            if (isset($visited[$currentPageId])) {
                break;
            }

            $visited[$currentPageId] = true;

            $stmt = $pdo->prepare("
                SELECT id, parent_id
                FROM sitebuilder.pages
                WHERE site_id = :site_id
                  AND id = :page_id
                LIMIT 1
            ");

            $stmt->execute([
                ':site_id' => $siteId,
                ':page_id' => $currentPageId,
            ]);

            $row = $stmt->fetch(PDO::FETCH_ASSOC);

            if (!$row) {
                break;
            }

            $ids[] = (int)$row['id'];
            $currentPageId = (int)($row['parent_id'] ?? 0);
        }

        return $ids;
    }

    public static function getPageIdsWithAccess(int $siteId, string $accessCode): array
    {
        if ($siteId <= 0) {
            return [];
        }

        $accessCode = self::normalizeAccessCode($accessCode);

        $pdo = sb_db();

        $stmt = $pdo->prepare("
            SELECT page_id
            FROM sitebuilder.page_access
            WHERE site_id = :site_id
              AND access_code = :access_code
        ");

        $stmt->execute([
            ':site_id' => $siteId,
            ':access_code' => $accessCode,
        ]);

        $ids = [];

        while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
            $ids[] = (int)$row['page_id'];
        }

        return array_values(array_unique($ids));
    }

    private static function mapRow(array $row): array
    {
        return [
            'id' => (int)$row['id'],
            'siteId' => (int)$row['site_id'],
            'pageId' => (int)$row['page_id'],
            'accessCode' => (string)$row['access_code'],
            'canView' => self::boolValue($row['can_view']),
            'canEdit' => self::boolValue($row['can_edit']),
            'includeChildren' => self::boolValue($row['include_children']),
            'createdBy' => isset($row['created_by']) ? (int)$row['created_by'] : 0,
            'createdAt' => (string)($row['created_at'] ?? ''),
            'updatedAt' => (string)($row['updated_at'] ?? ''),
        ];
    }

    private static function boolValue($value): bool
    {
        return $value === true
            || $value === 1
            || $value === '1'
            || $value === 't'
            || $value === 'true'
            || $value === 'Y';
    }
}


---

2. Замени PageAccessService.php

Файл:

/local/sitebuilder/lib/PageAccessService.php

Полностью замени на это:

<?php

require_once __DIR__ . '/db.php';
require_once __DIR__ . '/PageAccessRepository.php';

class PageAccessService
{
    public static function canViewPage(int $siteId, int $pageId, int $userId): bool
    {
        if ($siteId <= 0 || $pageId <= 0 || $userId <= 0) {
            return false;
        }

        if (self::isBitrixAdmin()) {
            return true;
        }

        if (self::hasGlobalSiteAccess($siteId, $userId, 'view')) {
            return true;
        }

        $accessCode = PageAccessRepository::userAccessCode($userId);

        return PageAccessRepository::hasPagePermission(
            $siteId,
            $pageId,
            $accessCode,
            'view'
        );
    }

    public static function canEditPage(int $siteId, int $pageId, int $userId): bool
    {
        if ($siteId <= 0 || $pageId <= 0 || $userId <= 0) {
            return false;
        }

        if (self::isBitrixAdmin()) {
            return true;
        }

        if (self::hasGlobalSiteAccess($siteId, $userId, 'edit')) {
            return true;
        }

        $accessCode = PageAccessRepository::userAccessCode($userId);

        return PageAccessRepository::hasPagePermission(
            $siteId,
            $pageId,
            $accessCode,
            'edit'
        );
    }

    public static function hasAnyPageAccess(int $siteId, int $userId): bool
    {
        if ($siteId <= 0 || $userId <= 0) {
            return false;
        }

        if (self::isBitrixAdmin()) {
            return true;
        }

        $accessCode = PageAccessRepository::userAccessCode($userId);

        return PageAccessRepository::hasAnyPageAccess($siteId, $accessCode);
    }

    public static function getPageAccessInfo(int $siteId, int $pageId, int $userId): array
    {
        return [
            'canView' => self::canViewPage($siteId, $pageId, $userId),
            'canEdit' => self::canEditPage($siteId, $pageId, $userId),
        ];
    }

    public static function filterVisiblePages(array $pages, int $siteId, int $userId): array
    {
        $filtered = [];

        foreach ($pages as $page) {
            $pageId = (int)($page['id'] ?? $page['ID'] ?? 0);

            if ($pageId <= 0) {
                continue;
            }

            if (self::canViewPage($siteId, $pageId, $userId)) {
                $page['access'] = self::getPageAccessInfo($siteId, $pageId, $userId);
                $filtered[] = $page;
            }
        }

        return $filtered;
    }

    public static function hasGlobalSiteAccess(int $siteId, int $userId, string $permission): bool
    {
        $role = self::getGlobalSiteRole($siteId, $userId);

        if ($permission === 'view') {
            return $role >= 1;
        }

        if ($permission === 'edit') {
            return $role >= 2;
        }

        if ($permission === 'admin') {
            return $role >= 3;
        }

        if ($permission === 'owner') {
            return $role >= 4;
        }

        return false;
    }

    private static function getGlobalSiteRole(int $siteId, int $userId): int
    {
        static $cache = [];

        if ($siteId <= 0 || $userId <= 0) {
            return 0;
        }

        $key = $siteId . ':' . $userId;

        if (isset($cache[$key])) {
            return $cache[$key];
        }

        $accessCode = PageAccessRepository::userAccessCode($userId);

        try {
            $pdo = sb_db();

            $stmt = $pdo->prepare("
                SELECT role
                FROM sitebuilder.access
                WHERE site_id = :site_id
                  AND access_code = :access_code
                LIMIT 1
            ");

            $stmt->execute([
                ':site_id' => $siteId,
                ':access_code' => $accessCode,
            ]);

            $row = $stmt->fetch(PDO::FETCH_ASSOC);
        } catch (Throwable $e) {
            $cache[$key] = 0;
            return 0;
        }

        if (!$row) {
            $cache[$key] = 0;
            return 0;
        }

        $role = $row['role'] ?? 0;

        if (is_numeric($role)) {
            $cache[$key] = (int)$role;
            return $cache[$key];
        }

        $roleString = mb_strtoupper((string)$role);

        $map = [
            'VIEWER' => 1,
            'EDITOR' => 2,
            'ADMIN' => 3,
            'OWNER' => 4,
        ];

        $cache[$key] = $map[$roleString] ?? 0;

        return $cache[$key];
    }

    private static function isBitrixAdmin(): bool
    {
        global $USER;

        return is_object($USER)
            && method_exists($USER, 'IsAdmin')
            && $USER->IsAdmin();
    }
}


---

3. Миграцию тоже запускай через sb_db()

Если ты создавал миграцию через Bitrix connection — лучше переделать.

Файл:

/local/sitebuilder/migrations/run_page_access.php

Код:

<?php

require_once $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/db.php';

global $USER;

if (!is_object($USER) || !$USER->IsAdmin()) {
    die('ACCESS_DENIED');
}

$pdo = sb_db();

$sql = [];

$sql[] = "CREATE SCHEMA IF NOT EXISTS sitebuilder";

$sql[] = "
CREATE TABLE IF NOT EXISTS sitebuilder.page_access (
    id BIGSERIAL PRIMARY KEY,

    site_id BIGINT NOT NULL,
    page_id BIGINT NOT NULL,

    access_code VARCHAR(64) NOT NULL,

    can_view BOOLEAN NOT NULL DEFAULT TRUE,
    can_edit BOOLEAN NOT NULL DEFAULT FALSE,

    include_children BOOLEAN NOT NULL DEFAULT FALSE,

    created_by BIGINT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_page_access_site_page_code UNIQUE (site_id, page_id, access_code)
)
";

$sql[] = "
CREATE INDEX IF NOT EXISTS ix_page_access_site_id
ON sitebuilder.page_access(site_id)
";

$sql[] = "
CREATE INDEX IF NOT EXISTS ix_page_access_page_id
ON sitebuilder.page_access(page_id)
";

$sql[] = "
CREATE INDEX IF NOT EXISTS ix_page_access_access_code
ON sitebuilder.page_access(access_code)
";

foreach ($sql as $query) {
    $pdo->exec($query);
}

$stmt = $pdo->query("
    SELECT
        to_regclass('sitebuilder.page_access') AS page_access,
        to_regclass('sitebuilder.pages') AS pages,
        to_regclass('sitebuilder.access') AS access
");

$check = $stmt->fetch(PDO::FETCH_ASSOC);

echo '<pre>';
echo "OK: migration completed through /local/sitebuilder/lib/db.php\n\n";
print_r($check);
echo '</pre>';

Открой:

/local/sitebuilder/migrations/run_page_access.php

Должно показать:

[page_access] => sitebuilder.page_access
[pages] => sitebuilder.pages
[access] => sitebuilder.access

После успешной проверки файл миграции лучше удалить.


---

После этого снова запусти проверку pageAccess.save. Ошибка из bitrix/modules/main/lib/db/pgsqlconnection.php уйдёт, потому что запросы пойдут через /local/sitebuilder/lib/db.php.