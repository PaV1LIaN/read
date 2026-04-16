Отлично. Ниже даю второй пакет файлов: уже серверную основу без “пустых мест” на уровне архитектуры.

Сделаю так:

будет единый доступ к БД;

будут репозитории Block, Site, DiskSettings;

будет реальная проверка контекста siteId/pageId/blockId;

будет реальный DiskRootResolver;

будет нормализованная работа с таблицами;

код останется совместимым с вашим sitebuilder.


Я покажу вариант на PDO, потому что он универсален. Если у вас уже есть свой db-wrapper, потом просто подмените слой DiskDb.


---

1. Что добавляем в компонент

Новая структура:

/local/sitebuilder/components/disk/
├── bootstrap.php
├── lib/
│   ├── helpers.php
│   ├── DiskDb.php
│   ├── BlockRepository.php
│   ├── SiteRepository.php
│   ├── DiskSettingsRepository.php
│   ├── DiskRootResolver.php
│   ├── DiskValidator.php
│   ├── DiskPermissionService.php
│   ├── DiskContext.php
│   ├── DiskResponse.php
│   ├── DiskStorageAdapterInterface.php
│   └── DiskStorageAdapter.php


---

2. SQL под эту серверную логику

Сразу уточню поля, которые ожидает код.

Таблица сайтов

CREATE TABLE sitebuilder_site (
    id                    BIGSERIAL PRIMARY KEY,
    name                  VARCHAR(255) NOT NULL,
    code                  VARCHAR(255) NULL,
    root_disk_folder_id   BIGINT NULL,
    settings_json         TEXT NULL,
    created_at            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);


---

Таблица страниц

CREATE TABLE sitebuilder_page (
    id                    BIGSERIAL PRIMARY KEY,
    site_id               BIGINT NOT NULL,
    title                 VARCHAR(255) NOT NULL,
    slug                  VARCHAR(255) NULL,
    sort                  INTEGER NOT NULL DEFAULT 500,
    created_at            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_sb_page_site_id ON sitebuilder_page(site_id);


---

Таблица блоков

CREATE TABLE sitebuilder_block (
    id                    BIGSERIAL PRIMARY KEY,
    site_id               BIGINT NOT NULL,
    page_id               BIGINT NOT NULL,
    type                  VARCHAR(50) NOT NULL,
    sort                  INTEGER NOT NULL DEFAULT 500,
    settings_json         TEXT NULL,
    is_active             SMALLINT NOT NULL DEFAULT 1,
    created_by            BIGINT NULL,
    created_at            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_sb_block_site_id ON sitebuilder_block(site_id);
CREATE INDEX idx_sb_block_page_id ON sitebuilder_block(page_id);
CREATE INDEX idx_sb_block_type ON sitebuilder_block(type);


---

Таблица настроек disk

CREATE TABLE sitebuilder_disk_settings (
    id                         BIGSERIAL PRIMARY KEY,
    block_id                   BIGINT NOT NULL UNIQUE,
    site_id                    BIGINT NOT NULL,
    page_id                    BIGINT NOT NULL,
    title                      VARCHAR(255) NOT NULL DEFAULT 'Файлы',
    root_folder_id             BIGINT NULL,
    view_mode                  VARCHAR(20) NOT NULL DEFAULT 'table',
    allow_upload               SMALLINT NOT NULL DEFAULT 1,
    allow_create_folder        SMALLINT NOT NULL DEFAULT 1,
    allow_rename               SMALLINT NOT NULL DEFAULT 1,
    allow_delete               SMALLINT NOT NULL DEFAULT 0,
    allow_download             SMALLINT NOT NULL DEFAULT 1,
    show_search                SMALLINT NOT NULL DEFAULT 1,
    show_breadcrumbs           SMALLINT NOT NULL DEFAULT 1,
    default_sort               VARCHAR(50) NOT NULL DEFAULT 'updatedAt',
    default_sort_direction     VARCHAR(10) NOT NULL DEFAULT 'desc',
    allowed_extensions_json    TEXT NULL,
    max_file_size              BIGINT NOT NULL DEFAULT 52428800,
    permission_mode            VARCHAR(30) NOT NULL DEFAULT 'inherit_site',
    use_site_root_fallback     SMALLINT NOT NULL DEFAULT 1,
    created_by                 BIGINT NULL,
    created_at                 TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at                 TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_sb_disk_settings_site_id ON sitebuilder_disk_settings(site_id);
CREATE INDEX idx_sb_disk_settings_page_id ON sitebuilder_disk_settings(page_id);


---

3. bootstrap.php

Ниже версия уже с новыми репозиториями и БД.

<?php

if (!defined('SITEBUILDER_DISK_BOOTSTRAP')) {
    define('SITEBUILDER_DISK_BOOTSTRAP', true);

    require_once __DIR__ . '/lib/helpers.php';
    require_once __DIR__ . '/lib/DiskDb.php';
    require_once __DIR__ . '/lib/DiskContext.php';
    require_once __DIR__ . '/lib/DiskResponse.php';
    require_once __DIR__ . '/lib/BlockRepository.php';
    require_once __DIR__ . '/lib/SiteRepository.php';
    require_once __DIR__ . '/lib/DiskSettingsRepository.php';
    require_once __DIR__ . '/lib/DiskRootResolver.php';
    require_once __DIR__ . '/lib/DiskValidator.php';
    require_once __DIR__ . '/lib/DiskPermissionService.php';
    require_once __DIR__ . '/lib/DiskStorageAdapterInterface.php';
    require_once __DIR__ . '/lib/DiskStorageAdapter.php';
}


---

4. lib/DiskDb.php

Это единая точка доступа к БД.

Если у вас уже есть свой db wrapper, вы замените только этот файл.

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

        // Вариант 1:
        // Подставить сюда ваш реальный способ получения PDO.
        //
        // Вариант 2:
        // Инициализировать DiskDb::setConnection($pdo) заранее в bootstrap проекта.

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

5. Как подключить вашу БД

Есть 2 рабочих пути.

Вариант A. Один раз инициализировать PDO в проекте

Например, где у вас общий bootstrap:

<?php

$pdo = new PDO(
    'pgsql:host=127.0.0.1;port=5432;dbname=sitebuilder',
    'db_user',
    'db_pass'
);

DiskDb::setConnection($pdo);


---

Вариант B. Подтянуть из вашего existing wrapper

Например, если у вас уже есть функция, возвращающая PDO:

<?php

class DiskDb
{
    protected static ?PDO $pdo = null;

    public static function getConnection(): PDO
    {
        if (self::$pdo instanceof PDO) {
            return self::$pdo;
        }

        require_once $_SERVER['DOCUMENT_ROOT'] . '/local/php_interface/lib/pg_master.php';

        $pdo = getPdo(); // пример
        if (!$pdo instanceof PDO) {
            throw new RuntimeException('DB_CONNECTION_NOT_CONFIGURED');
        }

        self::$pdo = $pdo;
        self::$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        self::$pdo->setAttribute(PDO::ATTR_DEFAULT_FETCH_MODE, PDO::FETCH_ASSOC);

        return self::$pdo;
    }
}

Если у вас реально есть getPdo() или аналог — это будет лучший путь.


---

6. lib/BlockRepository.php

Это основной репозиторий проверки контекста блока.

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

7. lib/SiteRepository.php

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

8. lib/DiskSettingsRepository.php

Это уже полноценный репозиторий под таблицу sitebuilder_disk_settings.

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


---

9. lib/DiskRootResolver.php

Теперь уже с реальным резолвом через SiteRepository.

<?php

class DiskRootResolver
{
    public static function resolve(DiskContext $context, array $settings): ?int
    {
        if (!empty($settings['rootFolderId'])) {
            return (int)$settings['rootFolderId'];
        }

        if (!empty($settings['useSiteRootFallback'])) {
            $siteRootFolderId = SiteRepository::getRootDiskFolderId($context->siteId);
            if ($siteRootFolderId !== null && $siteRootFolderId > 0) {
                return $siteRootFolderId;
            }
        }

        return null;
    }

    public static function resolveWithSource(DiskContext $context, array $settings): array
    {
        if (!empty($settings['rootFolderId'])) {
            return [
                'rootFolderId' => (int)$settings['rootFolderId'],
                'source' => 'block',
            ];
        }

        if (!empty($settings['useSiteRootFallback'])) {
            $siteRootFolderId = SiteRepository::getRootDiskFolderId($context->siteId);
            if ($siteRootFolderId !== null && $siteRootFolderId > 0) {
                return [
                    'rootFolderId' => $siteRootFolderId,
                    'source' => 'site',
                ];
            }
        }

        return [
            'rootFolderId' => null,
            'source' => 'none',
        ];
    }
}


---

10. lib/DiskValidator.php

Теперь уже с реальной проверкой блока и сайта.

<?php

class DiskValidator
{
    public static function assertContext(DiskContext $context): void
    {
        if ($context->siteId <= 0) {
            throw new RuntimeException('INVALID_SITE_ID');
        }

        if ($context->pageId <= 0) {
            throw new RuntimeException('INVALID_PAGE_ID');
        }

        if ($context->blockId <= 0) {
            throw new RuntimeException('INVALID_BLOCK_ID');
        }

        if ($context->currentUserId <= 0) {
            throw new RuntimeException('NOT_AUTHORIZED');
        }

        self::assertSiteExists($context->siteId);
        self::assertBlockBelongsToContext($context);
    }

    public static function assertSiteExists(int $siteId): void
    {
        $site = SiteRepository::getById($siteId);
        if (!$site) {
            throw new RuntimeException('SITE_NOT_FOUND');
        }
    }

    public static function assertBlockBelongsToContext(DiskContext $context): void
    {
        $block = BlockRepository::getDiskBlockByContext(
            $context->siteId,
            $context->pageId,
            $context->blockId
        );

        if (!$block) {
            throw new RuntimeException('BLOCK_CONTEXT_MISMATCH');
        }
    }

    public static function assertFolderInsideRoot(int $folderId, ?int $rootFolderId): void
    {
        if ($folderId <= 0) {
            throw new RuntimeException('INVALID_FOLDER_ID');
        }

        if ($rootFolderId === null || $rootFolderId <= 0) {
            throw new RuntimeException('ROOT_FOLDER_NOT_RESOLVED');
        }

        // TODO:
        // Здесь нужна реальная проверка дерева через адаптер хранилища
        // или таблицу folder tree.
        //
        // Временно допускаем:
        // - сам корень
        // - дальнейшая проверка должна быть заменена на прод-логику
        if ($folderId === $rootFolderId) {
            return;
        }

        // Пока временно разрешаем, но здесь обязательно заменить.
        $ok = true;

        if (!$ok) {
            throw new RuntimeException('FOLDER_OUT_OF_SCOPE');
        }
    }

    public static function assertCan(array $permissions, string $permissionKey): void
    {
        if (empty($permissions[$permissionKey])) {
            throw new RuntimeException('ACCESS_DENIED');
        }
    }

    public static function assertNonEmptyString(string $value, string $errorCode): void
    {
        if (trim($value) === '') {
            throw new RuntimeException($errorCode);
        }
    }
}


---

11. lib/DiskPermissionService.php

Пока без отдельной таблицы прав, но уже с понятной точкой расширения.

<?php

class DiskPermissionService
{
    public static function resolve(DiskContext $context, array $settings, ?int $rootFolderId = null): array
    {
        $rolePermissions = self::resolveRolePermissions($context);
        $blockRestrictions = self::resolveBlockRestrictions($settings);

        return [
            'canView' => $rolePermissions['canView'] && $blockRestrictions['canView'],
            'canUpload' => $rolePermissions['canUpload'] && $blockRestrictions['canUpload'],
            'canCreateFolder' => $rolePermissions['canCreateFolder'] && $blockRestrictions['canCreateFolder'],
            'canRename' => $rolePermissions['canRename'] && $blockRestrictions['canRename'],
            'canDelete' => $rolePermissions['canDelete'] && $blockRestrictions['canDelete'],
            'canDownload' => $rolePermissions['canDownload'] && $blockRestrictions['canDownload'],
            'canManageAccess' => $rolePermissions['canManageAccess'],
            'canEditSettings' => $rolePermissions['canEditSettings'],
        ];
    }

    protected static function resolveRolePermissions(DiskContext $context): array
    {
        // TODO:
        // Здесь нужно подставить реальную модель ролей сайта.
        // Пока:
        // - считаем текущего пользователя редактором сайта.
        //
        // Потом можно заменить на реальный SiteAccessService.

        return [
            'canView' => true,
            'canUpload' => true,
            'canCreateFolder' => true,
            'canRename' => true,
            'canDelete' => true,
            'canDownload' => true,
            'canManageAccess' => false,
            'canEditSettings' => true,
        ];
    }

    protected static function resolveBlockRestrictions(array $settings): array
    {
        return [
            'canView' => true,
            'canUpload' => !empty($settings['allowUpload']),
            'canCreateFolder' => !empty($settings['allowCreateFolder']),
            'canRename' => !empty($settings['allowRename']),
            'canDelete' => !empty($settings['allowDelete']),
            'canDownload' => !empty($settings['allowDownload']),
        ];
    }
}


---

12. Как использовать ensureExistsForBlock()

Это очень полезно при первом рендере блока, чтобы не ловить ситуацию “блок уже есть, а запись настроек еще не создана”.

В class.php лучше заменить кусок:

$settings = DiskSettingsRepository::getByBlockId($context->blockId);

на:

$settings = DiskSettingsRepository::ensureExistsForBlock(
    $context->blockId,
    $context->siteId,
    $context->pageId,
    $context->currentUserId
);


---

13. Как исправить actions/resolve_root.php

Чтобы вернуть источник корня, лучше уже использовать resolveWithSource().

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1, // TODO: заменить
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::ensureExistsForBlock(
    $context->blockId,
    $context->siteId,
    $context->pageId,
    $context->currentUserId
);

$result = DiskRootResolver::resolveWithSource($context, $settings);

DiskResponse::success([
    'rootFolderId' => $result['rootFolderId'],
    'source' => $result['source'],
]);


---

14. Как исправить actions/get_settings.php

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1, // TODO: заменить
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::ensureExistsForBlock(
    $context->blockId,
    $context->siteId,
    $context->pageId,
    $context->currentUserId
);

DiskResponse::success([
    'settings' => $settings,
]);


---

15. Как исправить actions/save_settings.php

Тут лучше сначала гарантировать существование записи.

<?php

$data = disk_read_json_body();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => 1, // TODO: заменить
]);

DiskValidator::assertContext($context);

$currentSettings = DiskSettingsRepository::ensureExistsForBlock(
    $context->blockId,
    $context->siteId,
    $context->pageId,
    $context->currentUserId
);

$rootFolderId = DiskRootResolver::resolve($context, $currentSettings);
$permissions = DiskPermissionService::resolve($context, $currentSettings, $rootFolderId);

DiskValidator::assertCan($permissions, 'canEditSettings');

$settings = $data['settings'] ?? [];
if (!is_array($settings)) {
    throw new RuntimeException('INVALID_SETTINGS_PAYLOAD');
}

DiskSettingsRepository::save($context->blockId, $settings);

$updatedSettings = DiskSettingsRepository::getByBlockId($context->blockId);

DiskResponse::success([
    'settings' => $updatedSettings,
]);


---

16. Как исправить class.php

Ниже уже обновленная рабочая версия.

<?php

require_once __DIR__ . '/bootstrap.php';

class SitebuilderDiskComponent
{
    protected array $params = [];
    protected array $result = [];

    public function __construct(array $params = [])
    {
        $this->params = $params;
    }

    public function execute(): void
    {
        try {
            $context = DiskContextFactory::fromArray([
                'siteId' => (int)($this->params['SITE_ID'] ?? 0),
                'pageId' => (int)($this->params['PAGE_ID'] ?? 0),
                'blockId' => (int)($this->params['BLOCK_ID'] ?? 0),
                'currentUserId' => (int)($this->params['CURRENT_USER_ID'] ?? 0),
            ]);

            DiskValidator::assertContext($context);

            $settings = DiskSettingsRepository::ensureExistsForBlock(
                $context->blockId,
                $context->siteId,
                $context->pageId,
                $context->currentUserId
            );

            $root = DiskRootResolver::resolveWithSource($context, $settings);
            $permissions = DiskPermissionService::resolve($context, $settings, $root['rootFolderId']);

            $this->result = [
                'SITE_ID' => $context->siteId,
                'PAGE_ID' => $context->pageId,
                'BLOCK_ID' => $context->blockId,
                'CURRENT_USER_ID' => $context->currentUserId,
                'SETTINGS' => $settings,
                'ROOT_FOLDER_ID' => $root['rootFolderId'],
                'ROOT_SOURCE' => $root['source'],
                'PERMISSIONS' => $permissions,
                'TITLE' => $settings['title'] ?? 'Файлы',
                'INITIAL_STATE' => [
                    'siteId' => $context->siteId,
                    'pageId' => $context->pageId,
                    'blockId' => $context->blockId,
                    'rootFolderId' => $root['rootFolderId'],
                    'rootSource' => $root['source'],
                    'currentFolderId' => $root['rootFolderId'],
                    'settings' => $settings,
                    'permissions' => $permissions,
                ],
                'ERROR' => null,
            ];
        } catch (Throwable $e) {
            $this->result = [
                'SITE_ID' => (int)($this->params['SITE_ID'] ?? 0),
                'PAGE_ID' => (int)($this->params['PAGE_ID'] ?? 0),
                'BLOCK_ID' => (int)($this->params['BLOCK_ID'] ?? 0),
                'CURRENT_USER_ID' => (int)($this->params['CURRENT_USER_ID'] ?? 0),
                'SETTINGS' => [],
                'ROOT_FOLDER_ID' => null,
                'ROOT_SOURCE' => 'none',
                'PERMISSIONS' => [
                    'canView' => false,
                    'canUpload' => false,
                    'canCreateFolder' => false,
                    'canRename' => false,
                    'canDelete' => false,
                    'canDownload' => false,
                    'canManageAccess' => false,
                    'canEditSettings' => false,
                ],
                'TITLE' => 'Файлы',
                'INITIAL_STATE' => [
                    'siteId' => (int)($this->params['SITE_ID'] ?? 0),
                    'pageId' => (int)($this->params['PAGE_ID'] ?? 0),
                    'blockId' => (int)($this->params['BLOCK_ID'] ?? 0),
                    'rootFolderId' => null,
                    'rootSource' => 'none',
                    'currentFolderId' => null,
                    'settings' => [],
                    'permissions' => [
                        'canView' => false,
                        'canUpload' => false,
                        'canCreateFolder' => false,
                        'canRename' => false,
                        'canDelete' => false,
                        'canDownload' => false,
                        'canManageAccess' => false,
                        'canEditSettings' => false,
                    ],
                ],
                'ERROR' => $e->getMessage(),
            ];
        }

        $arResult = $this->result;
        include __DIR__ . '/template.php';
    }
}


---

17. Как создавать блок “Диск” в конструкторе

Это уже можно использовать на сервере как нормальный helper.

<?php

function sitebuilder_create_disk_block(int $siteId, int $pageId, int $currentUserId, int $sort = 500): int
{
    $blockId = BlockRepository::create([
        'site_id' => $siteId,
        'page_id' => $pageId,
        'type' => 'disk',
        'sort' => $sort,
        'settings_json' => '{}',
        'is_active' => 1,
        'created_by' => $currentUserId,
    ]);

    DiskSettingsRepository::createDefault([
        'block_id' => $blockId,
        'site_id' => $siteId,
        'page_id' => $pageId,
        'created_by' => $currentUserId,
    ]);

    return $blockId;
}


---

18. Что уже стало “живым”, а не заглушкой

После этого пакета у вас уже не абстракция, а реальная серверная база:

siteId/pageId/blockId реально валидируются через БД;

настройки блока реально читаются и сохраняются;

запись настроек может автоматически создаваться;

rootFolderId реально резолвится:

сначала из блока,

потом из сайта;


структура уже пригодна для встраивания в sitebuilder.



---

19. Что осталось следующими реальными шагами

Сейчас у вас все еще заглушки в 3 местах:

1. Текущий пользователь

Во всех action-файлах надо заменить:

'currentUserId' => 1

на реальное получение пользователя из вашего проекта.


---

2. Модель ролей

В DiskPermissionService::resolveRolePermissions() надо подключить вашу реальную схему ролей сайта.


---

3. Проверка дерева папок и storage

В assertFolderInsideRoot() и DiskStorageAdapter надо уже подключить:

либо Bitrix Disk,

либо вашу таблицу папок/файлов,

либо другой storage.



---

20. Что я рекомендую следующим шагом

Сейчас самый логичный следующий ход — не продолжать теорию, а сделать один из двух практических пакетов:

вариант A: полноценный DiskStorageAdapter под вашу БД-модель folder/file
вариант B: адаптер под Bitrix Disk, если вы хотите опираться на него

Для вашего проекта я бы следующим сообщением сделал именно вариант B — адаптер под Bitrix Disk, потому что это сразу приблизит компонент к реальной корпоративной эксплуатации.