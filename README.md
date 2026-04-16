Ниже следующая пачка файлов.


---

/local/sitebuilder/components/disk/lib/DiskDb.php

<?php

class DiskDb
{
    protected static ?PDO $pdo = null;

    public static function setConnection(PDO $pdo): void
    {
        self::$pdo = $pdo;
        self::$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        self::$pdo->setAttribute(PDO::ATTR_DEFAULT_FETCH_MODE, PDO::FETCH_ASSOC);
    }

    public static function getConnection(): PDO
    {
        if (self::$pdo instanceof PDO) {
            return self::$pdo;
        }

        /*
         * ВАЖНО:
         * Здесь подставь свой реальный способ подключения к БД.
         *
         * Вариант 1:
         * require_once $_SERVER['DOCUMENT_ROOT'] . '/local/php_interface/lib/pg_master.php';
         * $pdo = getPdo();
         * self::setConnection($pdo);
         * return self::$pdo;
         *
         * Вариант 2:
         * Инициализируй DiskDb::setConnection($pdo) заранее в bootstrap проекта.
         */

        throw new RuntimeException('DB_CONNECTION_NOT_CONFIGURED');
    }

    public static function fetchOne(string $sql, array $params = []): ?array
    {
        $stmt = self::getConnection()->prepare($sql);
        $stmt->execute($params);
        $row = $stmt->fetch();

        return $row !== false ? $row : null;
    }

    public static function fetchAll(string $sql, array $params = []): array
    {
        $stmt = self::getConnection()->prepare($sql);
        $stmt->execute($params);
        return $stmt->fetchAll();
    }

    public static function execute(string $sql, array $params = []): bool
    {
        $stmt = self::getConnection()->prepare($sql);
        return $stmt->execute($params);
    }

    public static function lastInsertId(?string $name = null): string
    {
        return self::getConnection()->lastInsertId($name);
    }
}


---

/local/sitebuilder/components/disk/lib/BlockRepository.php

<?php

class BlockRepository
{
    public static function getById(int $blockId): ?array
    {
        if ($blockId <= 0) {
            return null;
        }

        $sql = "
            SELECT
                b.id,
                b.site_id,
                b.page_id,
                b.type,
                b.sort,
                b.settings_json,
                b.is_active,
                b.created_by,
                b.created_at,
                b.updated_at
            FROM sitebuilder_block b
            WHERE b.id = :id
            LIMIT 1
        ";

        return DiskDb::fetchOne($sql, [
            ':id' => $blockId,
        ]);
    }

    public static function getDiskBlockByContext(int $siteId, int $pageId, int $blockId): ?array
    {
        if ($siteId <= 0 || $pageId <= 0 || $blockId <= 0) {
            return null;
        }

        $sql = "
            SELECT
                b.id,
                b.site_id,
                b.page_id,
                b.type,
                b.sort,
                b.settings_json,
                b.is_active,
                b.created_by,
                b.created_at,
                b.updated_at
            FROM sitebuilder_block b
            WHERE b.id = :block_id
              AND b.site_id = :site_id
              AND b.page_id = :page_id
              AND b.type = 'disk'
              AND b.is_active = 1
            LIMIT 1
        ";

        return DiskDb::fetchOne($sql, [
            ':block_id' => $blockId,
            ':site_id' => $siteId,
            ':page_id' => $pageId,
        ]);
    }

    public static function create(array $data): int
    {
        $sql = "
            INSERT INTO sitebuilder_block (
                site_id,
                page_id,
                type,
                sort,
                settings_json,
                is_active,
                created_by,
                created_at,
                updated_at
            ) VALUES (
                :site_id,
                :page_id,
                :type,
                :sort,
                :settings_json,
                :is_active,
                :created_by,
                CURRENT_TIMESTAMP,
                CURRENT_TIMESTAMP
            )
        ";

        DiskDb::execute($sql, [
            ':site_id' => (int)$data['site_id'],
            ':page_id' => (int)$data['page_id'],
            ':type' => (string)$data['type'],
            ':sort' => (int)($data['sort'] ?? 500),
            ':settings_json' => (string)($data['settings_json'] ?? '{}'),
            ':is_active' => (int)($data['is_active'] ?? 1),
            ':created_by' => isset($data['created_by']) ? (int)$data['created_by'] : null,
        ]);

        return (int)DiskDb::lastInsertId();
    }

    public static function updateSettingsJson(int $blockId, array $settings): bool
    {
        $sql = "
            UPDATE sitebuilder_block
            SET settings_json = :settings_json,
                updated_at = CURRENT_TIMESTAMP
            WHERE id = :id
        ";

        return DiskDb::execute($sql, [
            ':id' => $blockId,
            ':settings_json' => json_encode($settings, JSON_UNESCAPED_UNICODE),
        ]);
    }
}


---

/local/sitebuilder/components/disk/lib/SiteRepository.php

<?php

class SiteRepository
{
    public static function getById(int $siteId): ?array
    {
        if ($siteId <= 0) {
            return null;
        }

        $sql = "
            SELECT
                s.id,
                s.name,
                s.code,
                s.root_disk_folder_id,
                s.settings_json,
                s.created_at,
                s.updated_at
            FROM sitebuilder_site s
            WHERE s.id = :id
            LIMIT 1
        ";

        return DiskDb::fetchOne($sql, [
            ':id' => $siteId,
        ]);
    }

    public static function getRootDiskFolderId(int $siteId): ?int
    {
        $row = self::getById($siteId);
        if (!$row) {
            return null;
        }

        return !empty($row['root_disk_folder_id'])
            ? (int)$row['root_disk_folder_id']
            : null;
    }

    public static function updateRootDiskFolderId(int $siteId, ?int $folderId): bool
    {
        $sql = "
            UPDATE sitebuilder_site
            SET root_disk_folder_id = :root_disk_folder_id,
                updated_at = CURRENT_TIMESTAMP
            WHERE id = :id
        ";

        return DiskDb::execute($sql, [
            ':id' => $siteId,
            ':root_disk_folder_id' => $folderId,
        ]);
    }
}


---

/local/sitebuilder/components/disk/lib/SiteAccessRepository.php

<?php

class SiteAccessRepository
{
    public static function getUserRole(int $siteId, int $userId): ?string
    {
        if ($siteId <= 0 || $userId <= 0) {
            return null;
        }

        $sql = "
            SELECT role_code
            FROM sitebuilder_site_user_access
            WHERE site_id = :site_id
              AND user_id = :user_id
            LIMIT 1
        ";

        $row = DiskDb::fetchOne($sql, [
            ':site_id' => $siteId,
            ':user_id' => $userId,
        ]);

        return $row ? (string)$row['role_code'] : null;
    }

    public static function setUserRole(int $siteId, int $userId, string $roleCode): bool
    {
        $roleCode = trim($roleCode);

        if (!in_array($roleCode, ['site_admin', 'site_editor', 'site_user', 'site_viewer'], true)) {
            throw new RuntimeException('INVALID_ROLE_CODE');
        }

        $existing = self::getUserRole($siteId, $userId);

        if ($existing === null) {
            $sql = "
                INSERT INTO sitebuilder_site_user_access (
                    site_id,
                    user_id,
                    role_code,
                    created_at,
                    updated_at
                ) VALUES (
                    :site_id,
                    :user_id,
                    :role_code,
                    CURRENT_TIMESTAMP,
                    CURRENT_TIMESTAMP
                )
            ";

            return DiskDb::execute($sql, [
                ':site_id' => $siteId,
                ':user_id' => $userId,
                ':role_code' => $roleCode,
            ]);
        }

        $sql = "
            UPDATE sitebuilder_site_user_access
            SET role_code = :role_code,
                updated_at = CURRENT_TIMESTAMP
            WHERE site_id = :site_id
              AND user_id = :user_id
        ";

        return DiskDb::execute($sql, [
            ':site_id' => $siteId,
            ':user_id' => $userId,
            ':role_code' => $roleCode,
        ]);
    }

    public static function hasAnyAccess(int $siteId, int $userId): bool
    {
        return self::getUserRole($siteId, $userId) !== null;
    }
}


---

/local/sitebuilder/components/disk/lib/DiskSettingsRepository.php

<?php

class DiskSettingsRepository
{
    public static function getByBlockId(int $blockId): array
    {
        if ($blockId <= 0) {
            throw new RuntimeException('INVALID_BLOCK_ID');
        }

        $sql = "
            SELECT
                ds.id,
                ds.block_id,
                ds.site_id,
                ds.page_id,
                ds.title,
                ds.root_folder_id,
                ds.view_mode,
                ds.allow_upload,
                ds.allow_create_folder,
                ds.allow_rename,
                ds.allow_delete,
                ds.allow_download,
                ds.show_search,
                ds.show_breadcrumbs,
                ds.default_sort,
                ds.default_sort_direction,
                ds.allowed_extensions_json,
                ds.max_file_size,
                ds.permission_mode,
                ds.use_site_root_fallback,
                ds.created_by,
                ds.created_at,
                ds.updated_at
            FROM sitebuilder_disk_settings ds
            WHERE ds.block_id = :block_id
            LIMIT 1
        ";

        $row = DiskDb::fetchOne($sql, [
            ':block_id' => $blockId,
        ]);

        if (!$row) {
            return self::getDefaultSettings($blockId);
        }

        return self::mapRowToSettings($row);
    }

    public static function createDefault(array $data): bool
    {
        $defaults = array_merge(self::getDefaultSettings((int)$data['block_id']), [
            'site_id' => (int)$data['site_id'],
            'page_id' => (int)$data['page_id'],
            'created_by' => isset($data['created_by']) ? (int)$data['created_by'] : null,
        ]);

        $sql = "
            INSERT INTO sitebuilder_disk_settings (
                block_id,
                site_id,
                page_id,
                title,
                root_folder_id,
                view_mode,
                allow_upload,
                allow_create_folder,
                allow_rename,
                allow_delete,
                allow_download,
                show_search,
                show_breadcrumbs,
                default_sort,
                default_sort_direction,
                allowed_extensions_json,
                max_file_size,
                permission_mode,
                use_site_root_fallback,
                created_by,
                created_at,
                updated_at
            ) VALUES (
                :block_id,
                :site_id,
                :page_id,
                :title,
                :root_folder_id,
                :view_mode,
                :allow_upload,
                :allow_create_folder,
                :allow_rename,
                :allow_delete,
                :allow_download,
                :show_search,
                :show_breadcrumbs,
                :default_sort,
                :default_sort_direction,
                :allowed_extensions_json,
                :max_file_size,
                :permission_mode,
                :use_site_root_fallback,
                :created_by,
                CURRENT_TIMESTAMP,
                CURRENT_TIMESTAMP
            )
        ";

        return DiskDb::execute($sql, [
            ':block_id' => (int)$defaults['block_id'],
            ':site_id' => (int)$defaults['site_id'],
            ':page_id' => (int)$defaults['page_id'],
            ':title' => (string)$defaults['title'],
            ':root_folder_id' => $defaults['rootFolderId'],
            ':view_mode' => (string)$defaults['viewMode'],
            ':allow_upload' => self::boolToDb($defaults['allowUpload']),
            ':allow_create_folder' => self::boolToDb($defaults['allowCreateFolder']),
            ':allow_rename' => self::boolToDb($defaults['allowRename']),
            ':allow_delete' => self::boolToDb($defaults['allowDelete']),
            ':allow_download' => self::boolToDb($defaults['allowDownload']),
            ':show_search' => self::boolToDb($defaults['showSearch']),
            ':show_breadcrumbs' => self::boolToDb($defaults['showBreadcrumbs']),
            ':default_sort' => (string)$defaults['defaultSort'],
            ':default_sort_direction' => (string)$defaults['defaultSortDirection'],
            ':allowed_extensions_json' => json_encode($defaults['allowedExtensions'], JSON_UNESCAPED_UNICODE),
            ':max_file_size' => (int)$defaults['maxFileSize'],
            ':permission_mode' => (string)$defaults['permissionMode'],
            ':use_site_root_fallback' => self::boolToDb($defaults['useSiteRootFallback']),
            ':created_by' => $defaults['created_by'],
        ]);
    }

    public static function save(int $blockId, array $settings): bool
    {
        $current = self::getByBlockId($blockId);

        $merged = array_merge($current, [
            'title' => trim((string)($settings['title'] ?? $current['title'])),
            'rootFolderId' => array_key_exists('rootFolderId', $settings) ? self::normalizeNullableInt($settings['rootFolderId']) : $current['rootFolderId'],
            'viewMode' => self::normalizeViewMode((string)($settings['viewMode'] ?? $current['viewMode'])),
            'allowUpload' => array_key_exists('allowUpload', $settings) ? disk_normalize_bool($settings['allowUpload']) : $current['allowUpload'],
            'allowCreateFolder' => array_key_exists('allowCreateFolder', $settings) ? disk_normalize_bool($settings['allowCreateFolder']) : $current['allowCreateFolder'],
            'allowRename' => array_key_exists('allowRename', $settings) ? disk_normalize_bool($settings['allowRename']) : $current['allowRename'],
            'allowDelete' => array_key_exists('allowDelete', $settings) ? disk_normalize_bool($settings['allowDelete']) : $current['allowDelete'],
            'allowDownload' => array_key_exists('allowDownload', $settings) ? disk_normalize_bool($settings['allowDownload']) : $current['allowDownload'],
            'showSearch' => array_key_exists('showSearch', $settings) ? disk_normalize_bool($settings['showSearch']) : $current['showSearch'],
            'showBreadcrumbs' => array_key_exists('showBreadcrumbs', $settings) ? disk_normalize_bool($settings['showBreadcrumbs']) : $current['showBreadcrumbs'],
            'defaultSort' => trim((string)($settings['defaultSort'] ?? $current['defaultSort'])),
            'defaultSortDirection' => self::normalizeSortDirection((string)($settings['defaultSortDirection'] ?? $current['defaultSortDirection'])),
            'allowedExtensions' => self::normalizeExtensions($settings['allowedExtensions'] ?? $current['allowedExtensions']),
            'maxFileSize' => max(0, (int)($settings['maxFileSize'] ?? $current['maxFileSize'])),
            'permissionMode' => self::normalizePermissionMode((string)($settings['permissionMode'] ?? $current['permissionMode'])),
            'useSiteRootFallback' => array_key_exists('useSiteRootFallback', $settings) ? disk_normalize_bool($settings['useSiteRootFallback']) : $current['useSiteRootFallback'],
        ]);

        $sql = "
            UPDATE sitebuilder_disk_settings
            SET title = :title,
                root_folder_id = :root_folder_id,
                view_mode = :view_mode,
                allow_upload = :allow_upload,
                allow_create_folder = :allow_create_folder,
                allow_rename = :allow_rename,
                allow_delete = :allow_delete,
                allow_download = :allow_download,
                show_search = :show_search,
                show_breadcrumbs = :show_breadcrumbs,
                default_sort = :default_sort,
                default_sort_direction = :default_sort_direction,
                allowed_extensions_json = :allowed_extensions_json,
                max_file_size = :max_file_size,
                permission_mode = :permission_mode,
                use_site_root_fallback = :use_site_root_fallback,
                updated_at = CURRENT_TIMESTAMP
            WHERE block_id = :block_id
        ";

        return DiskDb::execute($sql, [
            ':block_id' => $blockId,
            ':title' => $merged['title'],
            ':root_folder_id' => $merged['rootFolderId'],
            ':view_mode' => $merged['viewMode'],
            ':allow_upload' => self::boolToDb($merged['allowUpload']),
            ':allow_create_folder' => self::boolToDb($merged['allowCreateFolder']),
            ':allow_rename' => self::boolToDb($merged['allowRename']),
            ':allow_delete' => self::boolToDb($merged['allowDelete']),
            ':allow_download' => self::boolToDb($merged['allowDownload']),
            ':show_search' => self::boolToDb($merged['showSearch']),
            ':show_breadcrumbs' => self::boolToDb($merged['showBreadcrumbs']),
            ':default_sort' => $merged['defaultSort'],
            ':default_sort_direction' => $merged['defaultSortDirection'],
            ':allowed_extensions_json' => json_encode($merged['allowedExtensions'], JSON_UNESCAPED_UNICODE),
            ':max_file_size' => $merged['maxFileSize'],
            ':permission_mode' => $merged['permissionMode'],
            ':use_site_root_fallback' => self::boolToDb($merged['useSiteRootFallback']),
        ]);
    }

    public static function ensureExistsForBlock(int $blockId, int $siteId, int $pageId, ?int $createdBy = null): array
    {
        $existing = self::getRawByBlockId($blockId);
        if ($existing) {
            return self::mapRowToSettings($existing);
        }

        self::createDefault([
            'block_id' => $blockId,
            'site_id' => $siteId,
            'page_id' => $pageId,
            'created_by' => $createdBy,
        ]);

        return self::getByBlockId($blockId);
    }

    protected static function getRawByBlockId(int $blockId): ?array
    {
        $sql = "
            SELECT *
            FROM sitebuilder_disk_settings
            WHERE block_id = :block_id
            LIMIT 1
        ";

        return DiskDb::fetchOne($sql, [
            ':block_id' => $blockId,
        ]);
    }

    protected static function getDefaultSettings(int $blockId): array
    {
        return [
            'block_id' => $blockId,
            'site_id' => 0,
            'page_id' => 0,
            'title' => 'Файлы',
            'rootFolderId' => null,
            'viewMode' => 'table',
            'allowUpload' => true,
            'allowCreateFolder' => true,
            'allowRename' => true,
            'allowDelete' => false,
            'allowDownload' => true,
            'showSearch' => true,
            'showBreadcrumbs' => true,
            'defaultSort' => 'updatedAt',
            'defaultSortDirection' => 'desc',
            'allowedExtensions' => [],
            'maxFileSize' => 52428800,
            'permissionMode' => 'inherit_site',
            'useSiteRootFallback' => true,
        ];
    }

    protected static function mapRowToSettings(array $row): array
    {
        $extensions = [];
        if (!empty($row['allowed_extensions_json'])) {
            $decoded = json_decode((string)$row['allowed_extensions_json'], true);
            if (is_array($decoded)) {
                $extensions = array_values(array_filter(array_map(static function ($value) {
                    return strtolower(trim((string)$value));
                }, $decoded)));
            }
        }

        return [
            'id' => isset($row['id']) ? (int)$row['id'] : 0,
            'block_id' => (int)($row['block_id'] ?? 0),
            'site_id' => (int)($row['site_id'] ?? 0),
            'page_id' => (int)($row['page_id'] ?? 0),
            'title' => (string)($row['title'] ?? 'Файлы'),
            'rootFolderId' => self::normalizeNullableInt($row['root_folder_id'] ?? null),
            'viewMode' => self::normalizeViewMode((string)($row['view_mode'] ?? 'table')),
            'allowUpload' => !empty($row['allow_upload']),
            'allowCreateFolder' => !empty($row['allow_create_folder']),
            'allowRename' => !empty($row['allow_rename']),
            'allowDelete' => !empty($row['allow_delete']),
            'allowDownload' => !empty($row['allow_download']),
            'showSearch' => !empty($row['show_search']),
            'showBreadcrumbs' => !empty($row['show_breadcrumbs']),
            'defaultSort' => (string)($row['default_sort'] ?? 'updatedAt'),
            'defaultSortDirection' => self::normalizeSortDirection((string)($row['default_sort_direction'] ?? 'desc')),
            'allowedExtensions' => $extensions,
            'maxFileSize' => max(0, (int)($row['max_file_size'] ?? 52428800)),
            'permissionMode' => self::normalizePermissionMode((string)($row['permission_mode'] ?? 'inherit_site')),
            'useSiteRootFallback' => !empty($row['use_site_root_fallback']),
        ];
    }

    protected static function boolToDb(bool $value): int
    {
        return $value ? 1 : 0;
    }

    protected static function normalizeNullableInt($value): ?int
    {
        if ($value === null || $value === '' || (int)$value <= 0) {
            return null;
        }

        return (int)$value;
    }

    protected static function normalizeViewMode(string $value): string
    {
        return in_array($value, ['table', 'grid'], true) ? $value : 'table';
    }

    protected static function normalizeSortDirection(string $value): string
    {
        return strtolower($value) === 'asc' ? 'asc' : 'desc';
    }

    protected static function normalizePermissionMode(string $value): string
    {
        return in_array($value, ['inherit_site', 'custom'], true) ? $value : 'inherit_site';
    }

    protected static function normalizeExtensions($value): array
    {
        if (!is_array($value)) {
            return [];
        }

        $result = [];
        foreach ($value as $item) {
            $ext = strtolower(trim((string)$item));
            if ($ext !== '') {
                $result[] = $ext;
            }
        }

        return array_values(array_unique($result));
    }
}

Следом пришлю:

DiskRootResolver.php

DiskValidator.php

DiskPermissionService.php

SiteDiskInitializer.php

BlockDiskInitializer.php

DiskStorageAdapterInterface.php

DiskBitrixStorageAdapter.php