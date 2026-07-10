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
