Отлично. Идем правильно: сначала **PostgreSQL-ядро для `site + page + block`**, без ломки handlers.

Ниже даю **первый пакет**:

1. SQL таблицы
2. `lib/db.php`
3. новый слой чтения/записи для `site/page/block`
4. что подключить в `bootstrap.php`

---

# 1. SQL для PostgreSQL

Создай эти таблицы в схеме `sitebuilder`.

## `sitebuilder.site`

```sql
CREATE TABLE sitebuilder.site (
    id              BIGSERIAL PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    home_page_id    BIGINT NULL,
    disk_folder_id  BIGINT NULL,
    top_menu_id     BIGINT NULL,
    settings_json   JSONB NOT NULL DEFAULT '{}'::jsonb,
    layout_json     JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_by      BIGINT NULL,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by      BIGINT NULL,
    updated_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_site_slug ON sitebuilder.site(slug);
CREATE INDEX idx_site_updated_at ON sitebuilder.site(updated_at);
```

---

## `sitebuilder.page`

```sql
CREATE TABLE sitebuilder.page (
    id              BIGSERIAL PRIMARY KEY,
    site_id         BIGINT NOT NULL,
    title           VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL,
    parent_id       BIGINT NULL,
    sort            INTEGER NOT NULL DEFAULT 500,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    published_at    TIMESTAMP NULL,
    created_by      BIGINT NULL,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by      BIGINT NULL,
    updated_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_page_site_id ON sitebuilder.page(site_id);
CREATE INDEX idx_page_parent_id ON sitebuilder.page(parent_id);
CREATE INDEX idx_page_sort ON sitebuilder.page(site_id, parent_id, sort);
CREATE INDEX idx_page_status ON sitebuilder.page(status);
CREATE UNIQUE INDEX uq_page_site_slug ON sitebuilder.page(site_id, slug);
```

---

## `sitebuilder.block`

```sql
CREATE TABLE sitebuilder.block (
    id              BIGSERIAL PRIMARY KEY,
    page_id         BIGINT NOT NULL,
    type            VARCHAR(100) NOT NULL,
    sort            INTEGER NOT NULL DEFAULT 500,
    content_json    JSONB NOT NULL DEFAULT '{}'::jsonb,
    props_json      JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_by      BIGINT NULL,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by      BIGINT NULL,
    updated_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_block_page_id ON sitebuilder.block(page_id);
CREATE INDEX idx_block_sort ON sitebuilder.block(page_id, sort);
CREATE INDEX idx_block_type ON sitebuilder.block(type);
```

---

# 2. Новый файл `/local/sitebuilder/lib/db.php`

Если у тебя уже есть свой PDO-wrapper, можно потом подменить только этот файл.

```php
<?php

function sb_db(): PDO
{
    static $pdo = null;

    if ($pdo instanceof PDO) {
        return $pdo;
    }

    /*
     * Подставь свои реальные параметры.
     * Или замени этот блок на подключение через ваш существующий wrapper.
     */
    $dsn = 'pgsql:host=127.0.0.1;port=5432;dbname=sitebuilder';
    $user = 'sitebuilder_user';
    $pass = 'sitebuilder_pass';

    $pdo = new PDO($dsn, $user, $pass, [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    ]);

    $pdo->exec("SET search_path TO sitebuilder, public");

    return $pdo;
}

function sb_db_fetch_all(string $sql, array $params = []): array
{
    $stmt = sb_db()->prepare($sql);
    $stmt->execute($params);
    return $stmt->fetchAll();
}

function sb_db_fetch_one(string $sql, array $params = []): ?array
{
    $stmt = sb_db()->prepare($sql);
    $stmt->execute($params);
    $row = $stmt->fetch();
    return $row !== false ? $row : null;
}

function sb_db_execute(string $sql, array $params = []): bool
{
    $stmt = sb_db()->prepare($sql);
    return $stmt->execute($params);
}

function sb_db_last_insert_id(?string $sequence = null): int
{
    return (int)sb_db()->lastInsertId($sequence);
}

function sb_json_decode_assoc($value): array
{
    if (is_array($value)) {
        return $value;
    }

    if ($value === null || $value === '') {
        return [];
    }

    $decoded = json_decode((string)$value, true);
    return is_array($decoded) ? $decoded : [];
}
```

---

# 3. Подключить `db.php` в `bootstrap.php`

В `/local/sitebuilder/api/bootstrap.php` или в общем bootstrap проекта добавь:

```php
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/db.php';
```

Подключить надо **до** функций хранения.

---

# 4. Новый файл `/local/sitebuilder/lib/storage_db.php`

Это слой для `site + page + block`.

```php
<?php

function sb_read_sites(): array
{
    $rows = sb_db_fetch_all("
        SELECT
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
        FROM site
        ORDER BY id ASC
    ");

    return array_map('sb_map_site_row', $rows);
}

function sb_write_sites(array $sites): bool
{
    $pdo = sb_db();
    $pdo->beginTransaction();

    try {
        $existingRows = sb_db_fetch_all("SELECT id FROM site");
        $existingIds = array_map('intval', array_column($existingRows, 'id'));

        $incomingIds = [];

        foreach ($sites as $site) {
            $id = (int)($site['id'] ?? 0);

            if ($id > 0) {
                $incomingIds[] = $id;

                sb_db_execute("
                    UPDATE site
                    SET
                        name = :name,
                        slug = :slug,
                        home_page_id = :home_page_id,
                        disk_folder_id = :disk_folder_id,
                        top_menu_id = :top_menu_id,
                        settings_json = :settings_json::jsonb,
                        layout_json = :layout_json::jsonb,
                        created_by = :created_by,
                        created_at = :created_at,
                        updated_by = :updated_by,
                        updated_at = :updated_at
                    WHERE id = :id
                ", [
                    ':id' => $id,
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
            } else {
                sb_db_execute("
                    INSERT INTO site (
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
                ", [
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
        }

        $idsToDelete = array_diff($existingIds, $incomingIds);
        if (!empty($idsToDelete)) {
            $placeholders = implode(',', array_fill(0, count($idsToDelete), '?'));
            $stmt = $pdo->prepare("DELETE FROM site WHERE id IN ($placeholders)");
            $stmt->execute(array_values($idsToDelete));
        }

        $pdo->commit();
        return true;
    } catch (Throwable $e) {
        $pdo->rollBack();
        throw $e;
    }
}

function sb_read_pages(): array
{
    $rows = sb_db_fetch_all("
        SELECT
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
        FROM page
        ORDER BY site_id ASC, sort ASC, id ASC
    ");

    return array_map('sb_map_page_row', $rows);
}

function sb_write_pages(array $pages): bool
{
    $pdo = sb_db();
    $pdo->beginTransaction();

    try {
        $existingRows = sb_db_fetch_all("SELECT id FROM page");
        $existingIds = array_map('intval', array_column($existingRows, 'id'));

        $incomingIds = [];

        foreach ($pages as $page) {
            $id = (int)($page['id'] ?? 0);

            if ($id > 0) {
                $incomingIds[] = $id;

                sb_db_execute("
                    UPDATE page
                    SET
                        site_id = :site_id,
                        title = :title,
                        slug = :slug,
                        parent_id = :parent_id,
                        sort = :sort,
                        status = :status,
                        published_at = :published_at,
                        created_by = :created_by,
                        created_at = :created_at,
                        updated_by = :updated_by,
                        updated_at = :updated_at
                    WHERE id = :id
                ", [
                    ':id' => $id,
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
            } else {
                sb_db_execute("
                    INSERT INTO page (
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
                ", [
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
        }

        $idsToDelete = array_diff($existingIds, $incomingIds);
        if (!empty($idsToDelete)) {
            $placeholders = implode(',', array_fill(0, count($idsToDelete), '?'));
            $stmt = $pdo->prepare("DELETE FROM page WHERE id IN ($placeholders)");
            $stmt->execute(array_values($idsToDelete));
        }

        $pdo->commit();
        return true;
    } catch (Throwable $e) {
        $pdo->rollBack();
        throw $e;
    }
}

function sb_read_blocks(): array
{
    $rows = sb_db_fetch_all("
        SELECT
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
        FROM block
        ORDER BY page_id ASC, sort ASC, id ASC
    ");

    return array_map('sb_map_block_row', $rows);
}

function sb_write_blocks(array $blocks): bool
{
    $pdo = sb_db();
    $pdo->beginTransaction();

    try {
        $existingRows = sb_db_fetch_all("SELECT id FROM block");
        $existingIds = array_map('intval', array_column($existingRows, 'id'));

        $incomingIds = [];

        foreach ($blocks as $block) {
            $id = (int)($block['id'] ?? 0);

            if ($id > 0) {
                $incomingIds[] = $id;

                sb_db_execute("
                    UPDATE block
                    SET
                        page_id = :page_id,
                        type = :type,
                        sort = :sort,
                        content_json = :content_json::jsonb,
                        props_json = :props_json::jsonb,
                        created_by = :created_by,
                        created_at = :created_at,
                        updated_by = :updated_by,
                        updated_at = :updated_at
                    WHERE id = :id
                ", [
                    ':id' => $id,
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
            } else {
                sb_db_execute("
                    INSERT INTO block (
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
                ", [
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
        }

        $idsToDelete = array_diff($existingIds, $incomingIds);
        if (!empty($idsToDelete)) {
            $placeholders = implode(',', array_fill(0, count($idsToDelete), '?'));
            $stmt = $pdo->prepare("DELETE FROM block WHERE id IN ($placeholders)");
            $stmt->execute(array_values($idsToDelete));
        }

        $pdo->commit();
        return true;
    } catch (Throwable $e) {
        $pdo->rollBack();
        throw $e;
    }
}

function sb_map_site_row(array $row): array
{
    return [
        'id' => (int)$row['id'],
        'name' => (string)$row['name'],
        'slug' => (string)$row['slug'],
        'homePageId' => !empty($row['home_page_id']) ? (int)$row['home_page_id'] : 0,
        'diskFolderId' => !empty($row['disk_folder_id']) ? (int)$row['disk_folder_id'] : 0,
        'topMenuId' => !empty($row['top_menu_id']) ? (int)$row['top_menu_id'] : 0,
        'settings' => sb_json_decode_assoc($row['settings_json'] ?? '{}'),
        'layout' => sb_json_decode_assoc($row['layout_json'] ?? '{}'),
        'createdBy' => isset($row['created_by']) ? (int)$row['created_by'] : 0,
        'createdAt' => (string)($row['created_at'] ?? ''),
        'updatedBy' => isset($row['updated_by']) ? (int)$row['updated_by'] : 0,
        'updatedAt' => (string)($row['updated_at'] ?? ''),
    ];
}

function sb_map_page_row(array $row): array
{
    return [
        'id' => (int)$row['id'],
        'siteId' => (int)$row['site_id'],
        'title' => (string)$row['title'],
        'slug' => (string)$row['slug'],
        'parentId' => !empty($row['parent_id']) ? (int)$row['parent_id'] : 0,
        'sort' => (int)($row['sort'] ?? 500),
        'status' => (string)($row['status'] ?? 'draft'),
        'publishedAt' => !empty($row['published_at']) ? (string)$row['published_at'] : null,
        'createdBy' => isset($row['created_by']) ? (int)$row['created_by'] : 0,
        'createdAt' => (string)($row['created_at'] ?? ''),
        'updatedBy' => isset($row['updated_by']) ? (int)$row['updated_by'] : 0,
        'updatedAt' => (string)($row['updated_at'] ?? ''),
    ];
}

function sb_map_block_row(array $row): array
{
    return [
        'id' => (int)$row['id'],
        'pageId' => (int)$row['page_id'],
        'type' => (string)$row['type'],
        'sort' => (int)($row['sort'] ?? 500),
        'content' => sb_json_decode_assoc($row['content_json'] ?? '{}'),
        'props' => sb_json_decode_assoc($row['props_json'] ?? '{}'),
        'createdBy' => isset($row['created_by']) ? (int)$row['created_by'] : 0,
        'createdAt' => (string)($row['created_at'] ?? ''),
        'updatedBy' => isset($row['updated_by']) ? (int)$row['updated_by'] : 0,
        'updatedAt' => (string)($row['updated_at'] ?? ''),
    ];
}
```

---

# 5. Что подключить вместо JSON-слоя

Если сейчас `bootstrap.php` подключает старый `lib/json.php`, то на этом этапе лучше:

* либо заменить содержимое `lib/json.php` на код из `storage_db.php`,
* либо внутри `lib/json.php` подключить `db.php` и вставить туда эти функции.

Самый простой путь:

## `/local/sitebuilder/lib/json.php`

временно заменить на:

```php
<?php
require_once __DIR__ . '/db.php';
require_once __DIR__ . '/storage_db.php';
```

Если в старом `json.php` есть еще полезные helpers, тогда не заменяй целиком, а просто добавь `db.php` и вставь функции чтения/записи туда.

---

# 6. Что это даст уже сейчас

После этого без переписывания handlers начнут работать через PostgreSQL:

* `site.*`
* `page.*`
* `block.*`

То есть уже можно тестировать:

* создание сайта
* создание страницы
* создание блока
* редактирование страницы
* редактирование блока

---

# 7. Ограничение текущего этапа

Пока **не переведены**:

* `access`
* `menu`
* `layout`

Значит часть интерфейса может еще опираться на старый JSON, если ты не переведешь их следующим шагом.

---

# 8. Что делать дальше

Дальше логичный второй пакет:

1. `sitebuilder.access`
2. `sitebuilder.menu`
3. `sitebuilder.layout`
4. миграционный скрипт из старых JSON в PostgreSQL

Следующим сообщением я дам именно **второй пакет: access + menu + layout + migration script**.
