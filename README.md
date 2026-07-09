Да, начнём правильно: сначала добавляем таблицу и backend-сервис проверки прав, ничего старое не ломаем.

Сейчас делаем:

page.view — чтение страницы
page.edit — редактирование страницы
include_children — распространять на подстраницы


---

1. Создай SQL-миграцию

Файл:

/local/sitebuilder/migrations/2026_07_09_page_access.sql

Код:

CREATE SCHEMA IF NOT EXISTS sitebuilder;

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
);

CREATE INDEX IF NOT EXISTS ix_page_access_site_id
    ON sitebuilder.page_access(site_id);

CREATE INDEX IF NOT EXISTS ix_page_access_page_id
    ON sitebuilder.page_access(page_id);

CREATE INDEX IF NOT EXISTS ix_page_access_access_code
    ON sitebuilder.page_access(access_code);

Выполни в PostgreSQL.


---

2. Создай репозиторий прав страниц

Файл:

/local/sitebuilder/lib/PageAccessRepository.php

Код:

<?php

use Bitrix\Main\Application;

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
        $siteId = (int)$siteId;
        $pageId = (int)$pageId;

        if ($siteId <= 0 || $pageId <= 0) {
            return [];
        }

        $connection = Application::getConnection();

        $sql = "
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
            WHERE site_id = {$siteId}
              AND page_id = {$pageId}
            ORDER BY id DESC
        ";

        $result = $connection->query($sql);

        $items = [];

        while ($row = $result->fetch()) {
            $items[] = self::mapRow($row);
        }

        return $items;
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
        $siteId = (int)$siteId;
        $pageId = (int)$pageId;
        $createdBy = (int)$createdBy;

        if ($siteId <= 0) {
            throw new RuntimeException('INVALID_SITE_ID');
        }

        if ($pageId <= 0) {
            throw new RuntimeException('INVALID_PAGE_ID');
        }

        $accessCode = self::normalizeAccessCode($accessCode);

        /*
         * Редактирование автоматически включает чтение.
         */
        if ($canEdit) {
            $canView = true;
        }

        $connection = Application::getConnection();
        $sqlHelper = $connection->getSqlHelper();

        $accessCodeSql = "'" . $sqlHelper->forSql($accessCode) . "'";
        $canViewSql = $canView ? 'TRUE' : 'FALSE';
        $canEditSql = $canEdit ? 'TRUE' : 'FALSE';
        $includeChildrenSql = $includeChildren ? 'TRUE' : 'FALSE';
        $createdBySql = $createdBy > 0 ? (string)$createdBy : 'NULL';

        $sql = "
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
                {$siteId},
                {$pageId},
                {$accessCodeSql},
                {$canViewSql},
                {$canEditSql},
                {$includeChildrenSql},
                {$createdBySql},
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
        ";

        $row = $connection->query($sql)->fetch();

        if (!$row) {
            throw new RuntimeException('PAGE_ACCESS_SAVE_ERROR');
        }

        return self::mapRow($row);
    }

    public static function delete(int $id, int $siteId = 0, int $pageId = 0): bool
    {
        $id = (int)$id;
        $siteId = (int)$siteId;
        $pageId = (int)$pageId;

        if ($id <= 0) {
            throw new RuntimeException('INVALID_PAGE_ACCESS_ID');
        }

        $where = "id = {$id}";

        if ($siteId > 0) {
            $where .= " AND site_id = {$siteId}";
        }

        if ($pageId > 0) {
            $where .= " AND page_id = {$pageId}";
        }

        $connection = Application::getConnection();

        $connection->queryExecute("
            DELETE FROM sitebuilder.page_access
            WHERE {$where}
        ");

        return true;
    }

    public static function hasAnyPageAccess(int $siteId, string $accessCode): bool
    {
        $siteId = (int)$siteId;

        if ($siteId <= 0) {
            return false;
        }

        $accessCode = self::normalizeAccessCode($accessCode);

        $connection = Application::getConnection();
        $sqlHelper = $connection->getSqlHelper();

        $accessCodeSql = "'" . $sqlHelper->forSql($accessCode) . "'";

        $row = $connection->query("
            SELECT id
            FROM sitebuilder.page_access
            WHERE site_id = {$siteId}
              AND access_code = {$accessCodeSql}
            LIMIT 1
        ")->fetch();

        return !empty($row);
    }

    public static function hasPagePermission(
        int $siteId,
        int $pageId,
        string $accessCode,
        string $permission
    ): bool {
        $siteId = (int)$siteId;
        $pageId = (int)$pageId;

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

        $idsSql = implode(',', array_map('intval', $pageAndParents));

        $connection = Application::getConnection();
        $sqlHelper = $connection->getSqlHelper();

        $accessCodeSql = "'" . $sqlHelper->forSql($accessCode) . "'";

        $result = $connection->query("
            SELECT
                page_id,
                can_view,
                can_edit,
                include_children
            FROM sitebuilder.page_access
            WHERE site_id = {$siteId}
              AND access_code = {$accessCodeSql}
              AND page_id IN ({$idsSql})
        ");

        $rulesByPageId = [];

        while ($row = $result->fetch()) {
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
             * index = 0 — это сама страница.
             * index > 0 — это родитель.
             */
            $isDirectPage = $index === 0;

            if (!$isDirectPage && !$rule['includeChildren']) {
                continue;
            }

            if ($permission === 'view') {
                if ($rule['canView'] || $rule['canEdit']) {
                    return true;
                }
            }

            if ($permission === 'edit') {
                if ($rule['canEdit']) {
                    return true;
                }
            }
        }

        return false;
    }

    public static function getPageAndParentIds(int $siteId, int $pageId): array
    {
        $siteId = (int)$siteId;
        $pageId = (int)$pageId;

        if ($siteId <= 0 || $pageId <= 0) {
            return [];
        }

        $connection = Application::getConnection();

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

            $row = $connection->query("
                SELECT id, parent_id
                FROM sitebuilder.pages
                WHERE site_id = {$siteId}
                  AND id = {$currentPageId}
                LIMIT 1
            ")->fetch();

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
        $siteId = (int)$siteId;

        if ($siteId <= 0) {
            return [];
        }

        $accessCode = self::normalizeAccessCode($accessCode);

        $connection = Application::getConnection();
        $sqlHelper = $connection->getSqlHelper();

        $accessCodeSql = "'" . $sqlHelper->forSql($accessCode) . "'";

        $result = $connection->query("
            SELECT page_id
            FROM sitebuilder.page_access
            WHERE site_id = {$siteId}
              AND access_code = {$accessCodeSql}
        ");

        $ids = [];

        while ($row = $result->fetch()) {
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

3. Создай сервис проверки прав

Файл:

/local/sitebuilder/lib/PageAccessService.php

Код:

<?php

require_once __DIR__ . '/PageAccessRepository.php';

use Bitrix\Main\Application;

class PageAccessService
{
    public static function canViewPage(int $siteId, int $pageId, int $userId): bool
    {
        $siteId = (int)$siteId;
        $pageId = (int)$pageId;
        $userId = (int)$userId;

        if ($siteId <= 0 || $pageId <= 0 || $userId <= 0) {
            return false;
        }

        if (self::isBitrixAdmin()) {
            return true;
        }

        /*
         * Старое глобальное право на сайт.
         * VIEWER и выше видят весь сайт.
         */
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
        $siteId = (int)$siteId;
        $pageId = (int)$pageId;
        $userId = (int)$userId;

        if ($siteId <= 0 || $pageId <= 0 || $userId <= 0) {
            return false;
        }

        if (self::isBitrixAdmin()) {
            return true;
        }

        /*
         * Старое глобальное право на сайт.
         * EDITOR и выше могут редактировать весь сайт.
         */
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
        $siteId = (int)$siteId;
        $userId = (int)$userId;

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
        $siteId = (int)$siteId;
        $userId = (int)$userId;

        if ($siteId <= 0 || $userId <= 0) {
            return false;
        }

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

        $key = $siteId . ':' . $userId;

        if (isset($cache[$key])) {
            return $cache[$key];
        }

        $accessCode = PageAccessRepository::userAccessCode($userId);

        $connection = Application::getConnection();
        $sqlHelper = $connection->getSqlHelper();

        $accessCodeSql = "'" . $sqlHelper->forSql($accessCode) . "'";

        /*
         * Старый доступ сайта.
         * Ожидаем таблицу sitebuilder.access:
         * role: 1 VIEWER, 2 EDITOR, 3 ADMIN, 4 OWNER
         */
        try {
            $row = $connection->query("
                SELECT role
                FROM sitebuilder.access
                WHERE site_id = {$siteId}
                  AND access_code = {$accessCodeSql}
                LIMIT 1
            ")->fetch();
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
            return (int)$role;
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

4. Подключи новые файлы в общий bootstrap

Найди общий bootstrap проекта, скорее всего:

/local/sitebuilder/api/bootstrap.php

или файл, где подключаются репозитории.

Добавь туда:

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/PageAccessRepository.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/PageAccessService.php';


---

5. Что уже будет работать после этого

После этого у нас появятся backend-функции:

PageAccessService::canViewPage($siteId, $pageId, $userId);
PageAccessService::canEditPage($siteId, $pageId, $userId);

Пример:

if (!PageAccessService::canViewPage($siteId, $pageId, $USER->GetID())) {
    throw new RuntimeException('PAGE_ACCESS_DENIED');
}

И сохранение права:

PageAccessRepository::save(
    $siteId,
    $pageId,
    'U99',
    true,
    false,
    true,
    $USER->GetID()
);

Это значит:

Пользователю ID 99 дать чтение страницы и всех подстраниц.

Следующий шаг — добавляем API:

pageAccess.list
pageAccess.save
pageAccess.delete

И после этого уже подключим интерфейс выдачи прав в редакторе страницы.