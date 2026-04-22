Причина найдена.

Сейчас создание **не падает**, но и **не сохраняется**, потому что твои `site.create / page.create / block.create` сначала **сами назначают новый `id`** (`maxId + 1`, `sb_next_id(...)`), а потом вызывают `sb_write_sites / sb_write_pages / sb_write_blocks`. В handler `site.create` это видно прямо в коде: новый сайт получает `$id = $maxId + 1`, добавляется в массив и потом уходит в `sb_write_sites($sites)`.

А в твоем DB-layer логика сейчас такая:

* если `id > 0` → делается только `UPDATE`
* если `id = 0` → `INSERT`

Для нового сайта `id` уже больше нуля, значит идет `UPDATE`, но такой строки в БД еще нет, поэтому обновляется **0 строк**. Ошибки нет, но записи тоже нет.

Это же касается не только сайта, но и страниц/блоков: у `page.duplicate` и `block.duplicate` новые `id` тоже назначаются на PHP-стороне заранее.

---

# Что нужно сделать

Нужно переделать `sb_write_sites()`, `sb_write_pages()`, `sb_write_blocks()` на **UPSERT по `id`**, а не на “`UPDATE если id > 0`”.

Ниже готовые замены.

---

## Замени `sb_write_sites()` в `/local/sitebuilder/lib/storage_db.php`

```php
function sb_write_sites(array $sites): bool
{
    $pdo = sb_db();
    $pdo->beginTransaction();

    try {
        $existingRows = sb_db_fetch_all("SELECT id FROM sitebuilder.site");
        $existingIds = array_map('intval', array_column($existingRows, 'id'));

        $incomingIds = [];

        foreach ($sites as $site) {
            $id = (int)($site['id'] ?? 0);
            if ($id <= 0) {
                continue;
            }

            $incomingIds[] = $id;

            sb_db_execute("
                INSERT INTO sitebuilder.site (
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
                ':id' => $id,
                ':name' => (string)($site['name'] ?? ''),
                ':slug' => (string)($site['slug'] ?? ''),
                ':home_page_id' => !empty($site['homePageId']) ? (int)$site['homePageId'] : null,
                ':disk_folder_id' => !empty($site['diskFolderId']) ? (int)$site['diskFolderId'] : null,
                ':top_menu_id' => !empty($site['topMenuId']) ? (int)$site['topMenuId'] : null,
                ':settings_json' => json_encode($site['settings'] ?? [], JSON_UNESCAPED_UNICODE),
                ':layout_json' => json_encode($site['layout'] ?? [], JSON_UNESCAPED_UNICODE),
                ':created_by' => isset($site['createdBy']) ? (int)$site['createdBy'] : null,
                ':created_at' => (string)($site['createdAt'] ?? date('c')),
                ':updated_by' => isset($site['updatedBy']) ? (int)$site['updatedBy'] : null,
                ':updated_at' => (string)($site['updatedAt'] ?? date('c')),
            ]);
        }

        $idsToDelete = array_diff($existingIds, $incomingIds);
        if (!empty($idsToDelete)) {
            $placeholders = implode(',', array_fill(0, count($idsToDelete), '?'));
            $stmt = $pdo->prepare("DELETE FROM sitebuilder.site WHERE id IN ($placeholders)");
            $stmt->execute(array_values($idsToDelete));
        }

        $pdo->commit();
        return true;
    } catch (Throwable $e) {
        $pdo->rollBack();
        throw $e;
    }
}
```

---

## Замени `sb_write_pages()` в `/local/sitebuilder/lib/storage_db.php`

```php
function sb_write_pages(array $pages): bool
{
    $pdo = sb_db();
    $pdo->beginTransaction();

    try {
        $existingRows = sb_db_fetch_all("SELECT id FROM sitebuilder.page");
        $existingIds = array_map('intval', array_column($existingRows, 'id'));

        $incomingIds = [];

        foreach ($pages as $page) {
            $id = (int)($page['id'] ?? 0);
            if ($id <= 0) {
                continue;
            }

            $incomingIds[] = $id;

            sb_db_execute("
                INSERT INTO sitebuilder.page (
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
                ':id' => $id,
                ':site_id' => (int)($page['siteId'] ?? 0),
                ':title' => (string)($page['title'] ?? ''),
                ':slug' => (string)($page['slug'] ?? ''),
                ':parent_id' => !empty($page['parentId']) ? (int)$page['parentId'] : null,
                ':sort' => (int)($page['sort'] ?? 500),
                ':status' => (string)($page['status'] ?? 'draft'),
                ':published_at' => !empty($page['publishedAt']) ? (string)$page['publishedAt'] : null,
                ':created_by' => isset($page['createdBy']) ? (int)$page['createdBy'] : null,
                ':created_at' => (string)($page['createdAt'] ?? date('c')),
                ':updated_by' => isset($page['updatedBy']) ? (int)$page['updatedBy'] : null,
                ':updated_at' => (string)($page['updatedAt'] ?? date('c')),
            ]);
        }

        $idsToDelete = array_diff($existingIds, $incomingIds);
        if (!empty($idsToDelete)) {
            $placeholders = implode(',', array_fill(0, count($idsToDelete), '?'));
            $stmt = $pdo->prepare("DELETE FROM sitebuilder.page WHERE id IN ($placeholders)");
            $stmt->execute(array_values($idsToDelete));
        }

        $pdo->commit();
        return true;
    } catch (Throwable $e) {
        $pdo->rollBack();
        throw $e;
    }
}
```

---

## Замени `sb_write_blocks()` в `/local/sitebuilder/lib/storage_db.php`

```php
function sb_write_blocks(array $blocks): bool
{
    $pdo = sb_db();
    $pdo->beginTransaction();

    try {
        $existingRows = sb_db_fetch_all("SELECT id FROM sitebuilder.block");
        $existingIds = array_map('intval', array_column($existingRows, 'id'));

        $incomingIds = [];

        foreach ($blocks as $block) {
            $id = (int)($block['id'] ?? 0);
            if ($id <= 0) {
                continue;
            }

            $incomingIds[] = $id;

            sb_db_execute("
                INSERT INTO sitebuilder.block (
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
                ':id' => $id,
                ':page_id' => (int)($block['pageId'] ?? 0),
                ':type' => (string)($block['type'] ?? ''),
                ':sort' => (int)($block['sort'] ?? 500),
                ':content_json' => json_encode($block['content'] ?? [], JSON_UNESCAPED_UNICODE),
                ':props_json' => json_encode($block['props'] ?? [], JSON_UNESCAPED_UNICODE),
                ':created_by' => isset($block['createdBy']) ? (int)$block['createdBy'] : null,
                ':created_at' => (string)($block['createdAt'] ?? date('c')),
                ':updated_by' => isset($block['updatedBy']) ? (int)$block['updatedBy'] : null,
                ':updated_at' => (string)($block['updatedAt'] ?? date('c')),
            ]);
        }

        $idsToDelete = array_diff($existingIds, $incomingIds);
        if (!empty($idsToDelete)) {
            $placeholders = implode(',', array_fill(0, count($idsToDelete), '?'));
            $stmt = $pdo->prepare("DELETE FROM sitebuilder.block WHERE id IN ($placeholders)");
            $stmt->execute(array_values($idsToDelete));
        }

        $pdo->commit();
        return true;
    } catch (Throwable $e) {
        $pdo->rollBack();
        throw $e;
    }
}
```

---

# Почему это точно нужно

Потому что текущие handlers создают новые сущности **с уже заполненным `id`**, а не ждут автоинкремента из БД. Это есть и в `site.create`, и в `page.duplicate`, и в `block.duplicate`.

---

# Что проверить после замены

Сразу проверь по порядку:

1. создание сайта
2. создание страницы
3. создание блока
4. дублирование страницы
5. дублирование блока

Если `site.create` заработает, но потом упрутся `menu/layout`, это уже следующий слой.

---

# Очень вероятный следующий баг

После этого может всплыть такая же логика в `sb_write_menus()`:

* если новый menu создается с `id > 0`, он тоже может “обновляться в пустоту”.

Если так произойдет, скажи — я дам готовую `sb_write_menus()` в том же стиле.
