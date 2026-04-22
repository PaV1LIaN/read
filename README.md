Отлично. Тогда второй пакет: **`access + menu + layout + migration`**.

Идея та же:

* handlers **не трогаем**;
* меняем только storage layer;
* оставляем те же функции, которые уже дергает проект.

---

# 1. SQL таблицы PostgreSQL

Создай их в схеме `sitebuilder`.

## `sitebuilder.access`

```sql id="1g2y3y"
CREATE TABLE sitebuilder.access (
    id              BIGSERIAL PRIMARY KEY,
    site_id         BIGINT NOT NULL,
    access_code     VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL,
    created_by      BIGINT NULL,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by      BIGINT NULL,
    updated_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE UNIQUE INDEX uq_access_site_code
    ON sitebuilder.access(site_id, access_code);

CREATE INDEX idx_access_site_id
    ON sitebuilder.access(site_id);

CREATE INDEX idx_access_code
    ON sitebuilder.access(access_code);

CREATE INDEX idx_access_role
    ON sitebuilder.access(role);
```

---

## `sitebuilder.menu`

```sql id="do7k43"
CREATE TABLE sitebuilder.menu (
    id              BIGSERIAL PRIMARY KEY,
    site_id         BIGINT NOT NULL,
    name            VARCHAR(255) NOT NULL,
    items_json      JSONB NOT NULL DEFAULT '[]'::jsonb,
    created_by      BIGINT NULL,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by      BIGINT NULL,
    updated_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_menu_site_id
    ON sitebuilder.menu(site_id);

CREATE INDEX idx_menu_updated_at
    ON sitebuilder.menu(updated_at);
```

---

## `sitebuilder.layout`

```sql id="12q087"
CREATE TABLE sitebuilder.layout (
    site_id         BIGINT PRIMARY KEY,
    settings_json   JSONB NOT NULL DEFAULT '{}'::jsonb,
    zones_json      JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_by      BIGINT NULL,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by      BIGINT NULL,
    updated_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_layout_updated_at
    ON sitebuilder.layout(updated_at);
```

---

# 2. Новый файл `/local/sitebuilder/lib/storage_db_extra.php`

Этот файл добавляет storage для:

* access
* menus
* layouts

```php id="9vxtqs"
<?php

function sb_read_access(): array
{
    $rows = sb_db_fetch_all("
        SELECT
            id,
            site_id,
            access_code,
            role,
            created_by,
            created_at,
            updated_by,
            updated_at
        FROM access
        ORDER BY site_id ASC, access_code ASC
    ");

    return array_map('sb_map_access_row', $rows);
}

function sb_write_access(array $rows): bool
{
    $pdo = sb_db();
    $pdo->beginTransaction();

    try {
        $existingRows = sb_db_fetch_all("SELECT id FROM access");
        $existingIds = array_map('intval', array_column($existingRows, 'id'));

        $incomingIds = [];

        foreach ($rows as $row) {
            $id = (int)($row['id'] ?? 0);

            if ($id > 0) {
                $incomingIds[] = $id;

                sb_db_execute("
                    UPDATE access
                    SET
                        site_id = :site_id,
                        access_code = :access_code,
                        role = :role,
                        created_by = :created_by,
                        created_at = :created_at,
                        updated_by = :updated_by,
                        updated_at = :updated_at
                    WHERE id = :id
                ", [
                    ':id' => $id,
                    ':site_id' => (int)($row['siteId'] ?? 0),
                    ':access_code' => (string)($row['accessCode'] ?? ''),
                    ':role' => (string)($row['role'] ?? ''),
                    ':created_by' => isset($row['createdBy']) ? (int)$row['createdBy'] : null,
                    ':created_at' => (string)($row['createdAt'] ?? date('Y-m-d H:i:s')),
                    ':updated_by' => isset($row['updatedBy']) ? (int)$row['updatedBy'] : null,
                    ':updated_at' => (string)($row['updatedAt'] ?? date('Y-m-d H:i:s')),
                ]);
            } else {
                sb_db_execute("
                    INSERT INTO access (
                        site_id,
                        access_code,
                        role,
                        created_by,
                        created_at,
                        updated_by,
                        updated_at
                    ) VALUES (
                        :site_id,
                        :access_code,
                        :role,
                        :created_by,
                        :created_at,
                        :updated_by,
                        :updated_at
                    )
                ", [
                    ':site_id' => (int)($row['siteId'] ?? 0),
                    ':access_code' => (string)($row['accessCode'] ?? ''),
                    ':role' => (string)($row['role'] ?? ''),
                    ':created_by' => isset($row['createdBy']) ? (int)$row['createdBy'] : null,
                    ':created_at' => (string)($row['createdAt'] ?? date('Y-m-d H:i:s')),
                    ':updated_by' => isset($row['updatedBy']) ? (int)$row['updatedBy'] : null,
                    ':updated_at' => (string)($row['updatedAt'] ?? date('Y-m-d H:i:s')),
                ]);
            }
        }

        $idsToDelete = array_diff($existingIds, $incomingIds);
        if (!empty($idsToDelete)) {
            $placeholders = implode(',', array_fill(0, count($idsToDelete), '?'));
            $stmt = $pdo->prepare("DELETE FROM access WHERE id IN ($placeholders)");
            $stmt->execute(array_values($idsToDelete));
        }

        $pdo->commit();
        return true;
    } catch (Throwable $e) {
        $pdo->rollBack();
        throw $e;
    }
}

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

function sb_access_rows_for_site(int $siteId): array
{
    return array_values(array_filter(sb_read_access(), static function ($row) use ($siteId) {
        return (int)($row['siteId'] ?? 0) === $siteId;
    }));
}

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

function sb_read_menus(): array
{
    $rows = sb_db_fetch_all("
        SELECT
            id,
            site_id,
            name,
            items_json,
            created_by,
            created_at,
            updated_by,
            updated_at
        FROM menu
        ORDER BY site_id ASC, id ASC
    ");

    return array_map('sb_map_menu_row', $rows);
}

function sb_write_menus(array $menus): bool
{
    $pdo = sb_db();
    $pdo->beginTransaction();

    try {
        $existingRows = sb_db_fetch_all("SELECT id FROM menu");
        $existingIds = array_map('intval', array_column($existingRows, 'id'));

        $incomingIds = [];

        foreach ($menus as $menu) {
            $id = (int)($menu['id'] ?? 0);

            if ($id > 0) {
                $incomingIds[] = $id;

                sb_db_execute("
                    UPDATE menu
                    SET
                        site_id = :site_id,
                        name = :name,
                        items_json = :items_json::jsonb,
                        created_by = :created_by,
                        created_at = :created_at,
                        updated_by = :updated_by,
                        updated_at = :updated_at
                    WHERE id = :id
                ", [
                    ':id' => $id,
                    ':site_id' => (int)($menu['siteId'] ?? 0),
                    ':name' => (string)($menu['name'] ?? ''),
                    ':items_json' => json_encode(array_values($menu['items'] ?? []), JSON_UNESCAPED_UNICODE),
                    ':created_by' => isset($menu['createdBy']) ? (int)$menu['createdBy'] : null,
                    ':created_at' => (string)($menu['createdAt'] ?? date('Y-m-d H:i:s')),
                    ':updated_by' => isset($menu['updatedBy']) ? (int)$menu['updatedBy'] : null,
                    ':updated_at' => (string)($menu['updatedAt'] ?? date('Y-m-d H:i:s')),
                ]);
            } else {
                sb_db_execute("
                    INSERT INTO menu (
                        site_id,
                        name,
                        items_json,
                        created_by,
                        created_at,
                        updated_by,
                        updated_at
                    ) VALUES (
                        :site_id,
                        :name,
                        :items_json::jsonb,
                        :created_by,
                        :created_at,
                        :updated_by,
                        :updated_at
                    )
                ", [
                    ':site_id' => (int)($menu['siteId'] ?? 0),
                    ':name' => (string)($menu['name'] ?? ''),
                    ':items_json' => json_encode(array_values($menu['items'] ?? []), JSON_UNESCAPED_UNICODE),
                    ':created_by' => isset($menu['createdBy']) ? (int)$menu['createdBy'] : null,
                    ':created_at' => (string)($menu['createdAt'] ?? date('Y-m-d H:i:s')),
                    ':updated_by' => isset($menu['updatedBy']) ? (int)$menu['updatedBy'] : null,
                    ':updated_at' => (string)($menu['updatedAt'] ?? date('Y-m-d H:i:s')),
                ]);
            }
        }

        $idsToDelete = array_diff($existingIds, $incomingIds);
        if (!empty($idsToDelete)) {
            $placeholders = implode(',', array_fill(0, count($idsToDelete), '?'));
            $stmt = $pdo->prepare("DELETE FROM menu WHERE id IN ($placeholders)");
            $stmt->execute(array_values($idsToDelete));
        }

        $pdo->commit();
        return true;
    } catch (Throwable $e) {
        $pdo->rollBack();
        throw $e;
    }
}

function sb_find_menu(int $id): ?array
{
    foreach (sb_read_menus() as $menu) {
        if ((int)($menu['id'] ?? 0) === $id) {
            return $menu;
        }
    }

    return null;
}

function sb_next_menu_item_id(array $items): int
{
    $maxId = 0;

    foreach ($items as $item) {
        $maxId = max($maxId, (int)($item['id'] ?? 0));
    }

    return $maxId + 1;
}

function sb_menu_next_item_sort(array $items): int
{
    $maxSort = 0;

    foreach ($items as $item) {
        $maxSort = max($maxSort, (int)($item['sort'] ?? 0));
    }

    return $maxSort + 10;
}

function sb_read_layouts(): array
{
    $rows = sb_db_fetch_all("
        SELECT
            site_id,
            settings_json,
            zones_json,
            created_by,
            created_at,
            updated_by,
            updated_at
        FROM layout
        ORDER BY site_id ASC
    ");

    return array_map('sb_map_layout_row', $rows);
}

function sb_write_layouts(array $layouts): bool
{
    $pdo = sb_db();
    $pdo->beginTransaction();

    try {
        $existingRows = sb_db_fetch_all("SELECT site_id FROM layout");
        $existingIds = array_map('intval', array_column($existingRows, 'site_id'));

        $incomingIds = [];

        foreach ($layouts as $layout) {
            $siteId = (int)($layout['siteId'] ?? 0);
            if ($siteId <= 0) {
                continue;
            }

            $incomingIds[] = $siteId;

            $zones = $layout['zones'] ?? [];
            $settings = $layout['settings'] ?? [];

            $exists = in_array($siteId, $existingIds, true);

            if ($exists) {
                sb_db_execute("
                    UPDATE layout
                    SET
                        settings_json = :settings_json::jsonb,
                        zones_json = :zones_json::jsonb,
                        created_by = :created_by,
                        created_at = :created_at,
                        updated_by = :updated_by,
                        updated_at = :updated_at
                    WHERE site_id = :site_id
                ", [
                    ':site_id' => $siteId,
                    ':settings_json' => json_encode($settings, JSON_UNESCAPED_UNICODE),
                    ':zones_json' => json_encode($zones, JSON_UNESCAPED_UNICODE),
                    ':created_by' => isset($layout['createdBy']) ? (int)$layout['createdBy'] : null,
                    ':created_at' => (string)($layout['createdAt'] ?? date('Y-m-d H:i:s')),
                    ':updated_by' => isset($layout['updatedBy']) ? (int)$layout['updatedBy'] : null,
                    ':updated_at' => (string)($layout['updatedAt'] ?? date('Y-m-d H:i:s')),
                ]);
            } else {
                sb_db_execute("
                    INSERT INTO layout (
                        site_id,
                        settings_json,
                        zones_json,
                        created_by,
                        created_at,
                        updated_by,
                        updated_at
                    ) VALUES (
                        :site_id,
                        :settings_json::jsonb,
                        :zones_json::jsonb,
                        :created_by,
                        :created_at,
                        :updated_by,
                        :updated_at
                    )
                ", [
                    ':site_id' => $siteId,
                    ':settings_json' => json_encode($settings, JSON_UNESCAPED_UNICODE),
                    ':zones_json' => json_encode($zones, JSON_UNESCAPED_UNICODE),
                    ':created_by' => isset($layout['createdBy']) ? (int)$layout['createdBy'] : null,
                    ':created_at' => (string)($layout['createdAt'] ?? date('Y-m-d H:i:s')),
                    ':updated_by' => isset($layout['updatedBy']) ? (int)$layout['updatedBy'] : null,
                    ':updated_at' => (string)($layout['updatedAt'] ?? date('Y-m-d H:i:s')),
                ]);
            }
        }

        $idsToDelete = array_diff($existingIds, $incomingIds);
        if (!empty($idsToDelete)) {
            $placeholders = implode(',', array_fill(0, count($idsToDelete), '?'));
            $stmt = $pdo->prepare("DELETE FROM layout WHERE site_id IN ($placeholders)");
            $stmt->execute(array_values($idsToDelete));
        }

        $pdo->commit();
        return true;
    } catch (Throwable $e) {
        $pdo->rollBack();
        throw $e;
    }
}

function sb_map_access_row(array $row): array
{
    return [
        'id' => isset($row['id']) ? (int)$row['id'] : 0,
        'siteId' => (int)($row['site_id'] ?? 0),
        'accessCode' => (string)($row['access_code'] ?? ''),
        'role' => (string)($row['role'] ?? ''),
        'createdBy' => isset($row['created_by']) ? (int)$row['created_by'] : 0,
        'createdAt' => (string)($row['created_at'] ?? ''),
        'updatedBy' => isset($row['updated_by']) ? (int)$row['updated_by'] : 0,
        'updatedAt' => (string)($row['updated_at'] ?? ''),
    ];
}

function sb_map_menu_row(array $row): array
{
    $items = sb_json_decode_assoc($row['items_json'] ?? '[]');
    if (!is_array($items)) {
        $items = [];
    }

    return [
        'id' => (int)$row['id'],
        'siteId' => (int)($row['site_id'] ?? 0),
        'name' => (string)($row['name'] ?? ''),
        'items' => array_values($items),
        'createdBy' => isset($row['created_by']) ? (int)$row['created_by'] : 0,
        'createdAt' => (string)($row['created_at'] ?? ''),
        'updatedBy' => isset($row['updated_by']) ? (int)$row['updated_by'] : 0,
        'updatedAt' => (string)($row['updated_at'] ?? ''),
    ];
}

function sb_map_layout_row(array $row): array
{
    $settings = sb_json_decode_assoc($row['settings_json'] ?? '{}');
    $zones = sb_json_decode_assoc($row['zones_json'] ?? '{}');

    if (!isset($zones['header']) || !is_array($zones['header'])) $zones['header'] = [];
    if (!isset($zones['footer']) || !is_array($zones['footer'])) $zones['footer'] = [];
    if (!isset($zones['left']) || !is_array($zones['left'])) $zones['left'] = [];
    if (!isset($zones['right']) || !is_array($zones['right'])) $zones['right'] = [];

    return [
        'siteId' => (int)($row['site_id'] ?? 0),
        'settings' => is_array($settings) ? $settings : [],
        'zones' => $zones,
        'createdBy' => isset($row['created_by']) ? (int)$row['created_by'] : 0,
        'createdAt' => (string)($row['created_at'] ?? ''),
        'updatedBy' => isset($row['updated_by']) ? (int)$row['updated_by'] : 0,
        'updatedAt' => (string)($row['updated_at'] ?? ''),
    ];
}
```

---

# 3. Подключение в проект

Если ты уже сделал:

* `/local/sitebuilder/lib/db.php`
* `/local/sitebuilder/lib/storage_db.php`

то теперь просто подключи новый файл вместе с ними.

## В `lib/json.php`

Сделай так:

```php id="brh6of"
<?php
require_once __DIR__ . '/db.php';
require_once __DIR__ . '/storage_db.php';
require_once __DIR__ . '/storage_db_extra.php';
```

Если в старом `json.php` есть другие функции, которые нужны проекту, тогда не затирай их полностью, а просто добавь эти `require_once`.

---

# 4. Миграция из JSON в PostgreSQL

Теперь дам готовый скрипт миграции.

## Новый файл `/local/sitebuilder/tools/migrate_json_to_pg.php`

```php id="sq09cb"
<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/db.php';

function sb_migrate_read_json_file(string $path): array
{
    if (!is_file($path)) {
        return [];
    }

    $raw = file_get_contents($path);
    if ($raw === false || trim($raw) === '') {
        return [];
    }

    $decoded = json_decode($raw, true);
    return is_array($decoded) ? $decoded : [];
}

function sb_migrate_upsert_sites(array $sites): void
{
    foreach ($sites as $site) {
        sb_db_execute("
            INSERT INTO site (
                id,
                name,
                slug,
                home_page_id,
                disk_folder_id,
                top_menu_id,
                settings_json,
                layout_json,
                created_by,
                created_at,
                updated_by,
                updated_at
            ) VALUES (
                :id,
                :name,
                :slug,
                :home_page_id,
                :disk_folder_id,
                :top_menu_id,
                :settings_json::jsonb,
                :layout_json::jsonb,
                :created_by,
                :created_at,
                :updated_by,
                :updated_at
            )
            ON CONFLICT (id)
            DO UPDATE SET
                name = EXCLUDED.name,
                slug = EXCLUDED.slug,
                home_page_id = EXCLUDED.home_page_id,
                disk_folder_id = EXCLUDED.disk_folder_id,
                top_menu_id = EXCLUDED.top_menu_id,
                settings_json = EXCLUDED.settings_json,
                layout_json = EXCLUDED.layout_json,
                created_by = EXCLUDED.created_by,
                created_at = EXCLUDED.created_at,
                updated_by = EXCLUDED.updated_by,
                updated_at = EXCLUDED.updated_at
        ", [
            ':id' => (int)($site['id'] ?? 0),
            ':name' => (string)($site['name'] ?? ''),
            ':slug' => (string)($site['slug'] ?? ''),
            ':home_page_id' => !empty($site['homePageId']) ? (int)$site['homePageId'] : null,
            ':disk_folder_id' => !empty($site['diskFolderId']) ? (int)$site['diskFolderId'] : null,
            ':top_menu_id' => !empty($site['topMenuId']) ? (int)$site['topMenuId'] : null,
            ':settings_json' => json_encode($site['settings'] ?? [], JSON_UNESCAPED_UNICODE),
            ':layout_json' => json_encode($site['layout'] ?? [], JSON_UNESCAPED_UNICODE),
            ':created_by' => isset($site['createdBy']) ? (int)$site['createdBy'] : null,
            ':created_at' => (string)($site['createdAt'] ?? date('Y-m-d H:i:s')),
            ':updated_by' => isset($site['updatedBy']) ? (int)$site['updatedBy'] : null,
            ':updated_at' => (string)($site['updatedAt'] ?? date('Y-m-d H:i:s')),
        ]);
    }

    sb_db()->exec("SELECT setval(pg_get_serial_sequence('site', 'id'), COALESCE((SELECT MAX(id) FROM site), 1), true)");
}

function sb_migrate_upsert_pages(array $pages): void
{
    foreach ($pages as $page) {
        sb_db_execute("
            INSERT INTO page (
                id,
                site_id,
                title,
                slug,
                parent_id,
                sort,
                status,
                published_at,
                created_by,
                created_at,
                updated_by,
                updated_at
            ) VALUES (
                :id,
                :site_id,
                :title,
                :slug,
                :parent_id,
                :sort,
                :status,
                :published_at,
                :created_by,
                :created_at,
                :updated_by,
                :updated_at
            )
            ON CONFLICT (id)
            DO UPDATE SET
                site_id = EXCLUDED.site_id,
                title = EXCLUDED.title,
                slug = EXCLUDED.slug,
                parent_id = EXCLUDED.parent_id,
                sort = EXCLUDED.sort,
                status = EXCLUDED.status,
                published_at = EXCLUDED.published_at,
                created_by = EXCLUDED.created_by,
                created_at = EXCLUDED.created_at,
                updated_by = EXCLUDED.updated_by,
                updated_at = EXCLUDED.updated_at
        ", [
            ':id' => (int)($page['id'] ?? 0),
            ':site_id' => (int)($page['siteId'] ?? 0),
            ':title' => (string)($page['title'] ?? ''),
            ':slug' => (string)($page['slug'] ?? ''),
            ':parent_id' => !empty($page['parentId']) ? (int)$page['parentId'] : null,
            ':sort' => (int)($page['sort'] ?? 500),
            ':status' => (string)($page['status'] ?? 'draft'),
            ':published_at' => !empty($page['publishedAt']) ? (string)$page['publishedAt'] : null,
            ':created_by' => isset($page['createdBy']) ? (int)$page['createdBy'] : null,
            ':created_at' => (string)($page['createdAt'] ?? date('Y-m-d H:i:s')),
            ':updated_by' => isset($page['updatedBy']) ? (int)$page['updatedBy'] : null,
            ':updated_at' => (string)($page['updatedAt'] ?? date('Y-m-d H:i:s')),
        ]);
    }

    sb_db()->exec("SELECT setval(pg_get_serial_sequence('page', 'id'), COALESCE((SELECT MAX(id) FROM page), 1), true)");
}

function sb_migrate_upsert_blocks(array $blocks): void
{
    foreach ($blocks as $block) {
        sb_db_execute("
            INSERT INTO block (
                id,
                page_id,
                type,
                sort,
                content_json,
                props_json,
                created_by,
                created_at,
                updated_by,
                updated_at
            ) VALUES (
                :id,
                :page_id,
                :type,
                :sort,
                :content_json::jsonb,
                :props_json::jsonb,
                :created_by,
                :created_at,
                :updated_by,
                :updated_at
            )
            ON CONFLICT (id)
            DO UPDATE SET
                page_id = EXCLUDED.page_id,
                type = EXCLUDED.type,
                sort = EXCLUDED.sort,
                content_json = EXCLUDED.content_json,
                props_json = EXCLUDED.props_json,
                created_by = EXCLUDED.created_by,
                created_at = EXCLUDED.created_at,
                updated_by = EXCLUDED.updated_by,
                updated_at = EXCLUDED.updated_at
        ", [
            ':id' => (int)($block['id'] ?? 0),
            ':page_id' => (int)($block['pageId'] ?? 0),
            ':type' => (string)($block['type'] ?? ''),
            ':sort' => (int)($block['sort'] ?? 500),
            ':content_json' => json_encode($block['content'] ?? [], JSON_UNESCAPED_UNICODE),
            ':props_json' => json_encode($block['props'] ?? [], JSON_UNESCAPED_UNICODE),
            ':created_by' => isset($block['createdBy']) ? (int)$block['createdBy'] : null,
            ':created_at' => (string)($block['createdAt'] ?? date('Y-m-d H:i:s')),
            ':updated_by' => isset($block['updatedBy']) ? (int)$block['updatedBy'] : null,
            ':updated_at' => (string)($block['updatedAt'] ?? date('Y-m-d H:i:s')),
        ]);
    }

    sb_db()->exec("SELECT setval(pg_get_serial_sequence('block', 'id'), COALESCE((SELECT MAX(id) FROM block), 1), true)");
}

function sb_migrate_upsert_access(array $rows): void
{
    foreach ($rows as $row) {
        sb_db_execute("
            INSERT INTO access (
                site_id,
                access_code,
                role,
                created_by,
                created_at,
                updated_by,
                updated_at
            ) VALUES (
                :site_id,
                :access_code,
                :role,
                :created_by,
                :created_at,
                :updated_by,
                :updated_at
            )
            ON CONFLICT (site_id, access_code)
            DO UPDATE SET
                role = EXCLUDED.role,
                created_by = EXCLUDED.created_by,
                created_at = EXCLUDED.created_at,
                updated_by = EXCLUDED.updated_by,
                updated_at = EXCLUDED.updated_at
        ", [
            ':site_id' => (int)($row['siteId'] ?? 0),
            ':access_code' => (string)($row['accessCode'] ?? ''),
            ':role' => (string)($row['role'] ?? ''),
            ':created_by' => isset($row['createdBy']) ? (int)$row['createdBy'] : null,
            ':created_at' => (string)($row['createdAt'] ?? date('Y-m-d H:i:s')),
            ':updated_by' => isset($row['updatedBy']) ? (int)$row['updatedBy'] : null,
            ':updated_at' => (string)($row['updatedAt'] ?? date('Y-m-d H:i:s')),
        ]);
    }
}

function sb_migrate_upsert_menus(array $menus): void
{
    foreach ($menus as $menu) {
        sb_db_execute("
            INSERT INTO menu (
                id,
                site_id,
                name,
                items_json,
                created_by,
                created_at,
                updated_by,
                updated_at
            ) VALUES (
                :id,
                :site_id,
                :name,
                :items_json::jsonb,
                :created_by,
                :created_at,
                :updated_by,
                :updated_at
            )
            ON CONFLICT (id)
            DO UPDATE SET
                site_id = EXCLUDED.site_id,
                name = EXCLUDED.name,
                items_json = EXCLUDED.items_json,
                created_by = EXCLUDED.created_by,
                created_at = EXCLUDED.created_at,
                updated_by = EXCLUDED.updated_by,
                updated_at = EXCLUDED.updated_at
        ", [
            ':id' => (int)($menu['id'] ?? 0),
            ':site_id' => (int)($menu['siteId'] ?? 0),
            ':name' => (string)($menu['name'] ?? ''),
            ':items_json' => json_encode(array_values($menu['items'] ?? []), JSON_UNESCAPED_UNICODE),
            ':created_by' => isset($menu['createdBy']) ? (int)$menu['createdBy'] : null,
            ':created_at' => (string)($menu['createdAt'] ?? date('Y-m-d H:i:s')),
            ':updated_by' => isset($menu['updatedBy']) ? (int)$menu['updatedBy'] : null,
            ':updated_at' => (string)($menu['updatedAt'] ?? date('Y-m-d H:i:s')),
        ]);
    }

    sb_db()->exec("SELECT setval(pg_get_serial_sequence('menu', 'id'), COALESCE((SELECT MAX(id) FROM menu), 1), true)");
}

function sb_migrate_upsert_layouts(array $layouts): void
{
    foreach ($layouts as $layout) {
        sb_db_execute("
            INSERT INTO layout (
                site_id,
                settings_json,
                zones_json,
                created_by,
                created_at,
                updated_by,
                updated_at
            ) VALUES (
                :site_id,
                :settings_json::jsonb,
                :zones_json::jsonb,
                :created_by,
                :created_at,
                :updated_by,
                :updated_at
            )
            ON CONFLICT (site_id)
            DO UPDATE SET
                settings_json = EXCLUDED.settings_json,
                zones_json = EXCLUDED.zones_json,
                created_by = EXCLUDED.created_by,
                created_at = EXCLUDED.created_at,
                updated_by = EXCLUDED.updated_by,
                updated_at = EXCLUDED.updated_at
        ", [
            ':site_id' => (int)($layout['siteId'] ?? 0),
            ':settings_json' => json_encode($layout['settings'] ?? [], JSON_UNESCAPED_UNICODE),
            ':zones_json' => json_encode($layout['zones'] ?? [], JSON_UNESCAPED_UNICODE),
            ':created_by' => isset($layout['createdBy']) ? (int)$layout['createdBy'] : null,
            ':created_at' => (string)($layout['createdAt'] ?? date('Y-m-d H:i:s')),
            ':updated_by' => isset($layout['updatedBy']) ? (int)$layout['updatedBy'] : null,
            ':updated_at' => (string)($layout['updatedAt'] ?? date('Y-m-d H:i:s')),
        ]);
    }
}

$storageDir = $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/storage';

$sites = sb_migrate_read_json_file($storageDir . '/sites.json');
$pages = sb_migrate_read_json_file($storageDir . '/pages.json');
$blocks = sb_migrate_read_json_file($storageDir . '/blocks.json');
$access = sb_migrate_read_json_file($storageDir . '/access.json');
$menus = sb_migrate_read_json_file($storageDir . '/menus.json');
$layouts = sb_migrate_read_json_file($storageDir . '/layouts.json');

echo '<pre>';

try {
    sb_db()->beginTransaction();

    sb_migrate_upsert_sites($sites);
    echo "sites migrated: " . count($sites) . PHP_EOL;

    sb_migrate_upsert_pages($pages);
    echo "pages migrated: " . count($pages) . PHP_EOL;

    sb_migrate_upsert_blocks($blocks);
    echo "blocks migrated: " . count($blocks) . PHP_EOL;

    sb_migrate_upsert_access($access);
    echo "access migrated: " . count($access) . PHP_EOL;

    sb_migrate_upsert_menus($menus);
    echo "menus migrated: " . count($menus) . PHP_EOL;

    sb_migrate_upsert_layouts($layouts);
    echo "layouts migrated: " . count($layouts) . PHP_EOL;

    sb_db()->commit();
    echo PHP_EOL . "DONE" . PHP_EOL;
} catch (Throwable $e) {
    sb_db()->rollBack();
    echo 'ERROR: ' . $e->getMessage() . PHP_EOL;
}

echo '</pre>';
```

---

# 5. Что это даст

После этого у тебя через PostgreSQL уже будут работать все основные сущности:

* sites
* pages
* blocks
* access
* menus
* layouts

То есть фактически **основной storage sitebuilder уже будет сидеть в PostgreSQL**, а не в JSON.

---

# 6. Что важно проверить сразу после миграции

После запуска `migrate_json_to_pg.php` проверь:

1. `site.list`
2. `site.get`
3. `page.list`
4. `page.create`
5. `block.list`
6. `block.create`
7. `menu.list`
8. `layout.get`
9. `access.list`

Если все это работает — значит ядро хранения уже переведено.

---

# 7. Что останется следующим шагом

Следующий логичный пакет после этого:

1. **убрать старую JSON-зависимость окончательно**
2. **добавить foreign keys**
3. **перевести handlers с read/write whole array на точечные SQL-операции**
4. **сделать нормальный repository layer**
5. **добавить транзакции в site/page/block delete/move operations**

То есть следующий шаг — уже не “миграция на PostgreSQL”, а **нормализация архитектуры под PostgreSQL**.

Если хочешь, следующим сообщением я дам тебе **третий пакет: foreign keys + cleanup + repository layer plan + точечный SQL для handlers**.
