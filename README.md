Да, следующий этап — **секции сайтов**.

Сделаем сначала backend: таблица, API секций и поле `sectionId` у сайта. После проверки добавим группировку на главной странице.

---

## 1. Выполни SQL

```sql
CREATE TABLE IF NOT EXISTS sitebuilder.site_section (
    id bigserial PRIMARY KEY,
    name varchar(255) NOT NULL,
    sort int NOT NULL DEFAULT 500,
    created_by int NULL,
    created_at timestamp without time zone DEFAULT now(),
    updated_by int NULL,
    updated_at timestamp without time zone DEFAULT now()
);

ALTER TABLE sitebuilder.site
ADD COLUMN IF NOT EXISTS section_id bigint NULL;

CREATE INDEX IF NOT EXISTS ix_site_section_sort
ON sitebuilder.site_section(sort, id);

CREATE INDEX IF NOT EXISTS ix_site_section_id
ON sitebuilder.site(section_id);
```

---

## 2. В `storage_db.php` добавь `section_id`

Файл:

```text
/local/sitebuilder/lib/storage_db.php
```

### В `sb_read_sites()` в SELECT добавь:

```sql
section_id,
```

Например:

```php
$rows = sb_db_fetch_all("
    SELECT
        id,
        name,
        slug,
        section_id,
        home_page_id,
        disk_folder_id,
        top_menu_id,
        bitrix_group_id,
        bitrix_group_created_by,
        bitrix_group_created_at,
        settings_json,
        layout_json,
        created_by,
        created_at,
        updated_by,
        updated_at
    FROM sitebuilder.site
    ORDER BY id ASC
");
```

### В `sb_write_sites()` добавь `section_id`

В `INSERT INTO sitebuilder.site (...)` добавь:

```sql
section_id,
```

После `slug,`:

```sql
slug,
section_id,
home_page_id,
```

В `VALUES (...)` добавь:

```sql
:section_id,
```

После `:slug,`:

```sql
:slug,
:section_id,
:home_page_id,
```

В `DO UPDATE SET` добавь:

```sql
section_id = EXCLUDED.section_id,
```

После:

```sql
slug = EXCLUDED.slug,
```

должно быть:

```sql
slug = EXCLUDED.slug,
section_id = EXCLUDED.section_id,
home_page_id = EXCLUDED.home_page_id,
```

В массив параметров добавь:

```php
':section_id' => !empty($site['sectionId']) ? (int)$site['sectionId'] : null,
```

После:

```php
':slug' => (string)($site['slug'] ?? ''),
```

---

## 3. В `sb_map_site_row()` добавь `sectionId`

В функции:

```php
function sb_map_site_row(array $row): array
```

после:

```php
'slug' => (string)$row['slug'],
```

добавь:

```php
'sectionId' => !empty($row['section_id']) ? (int)$row['section_id'] : 0,
```

---

## 4. Создай API-обработчик секций

Создай файл:

```text
/local/sitebuilder/api/handlers/section.php
```

Код:

```php
<?php

global $USER;

function sb_section_require_admin(): void
{
    global $USER;

    if (!$USER || !$USER->IsAdmin()) {
        sb_json_error('BITRIX_ADMIN_REQUIRED', 403);
    }
}

function sb_section_normalize_row(array $row): array
{
    return [
        'id' => (int)($row['id'] ?? 0),
        'name' => (string)($row['name'] ?? ''),
        'sort' => (int)($row['sort'] ?? 500),
        'createdBy' => isset($row['created_by']) ? (int)$row['created_by'] : 0,
        'createdAt' => (string)($row['created_at'] ?? ''),
        'updatedBy' => isset($row['updated_by']) ? (int)$row['updated_by'] : 0,
        'updatedAt' => (string)($row['updated_at'] ?? ''),
    ];
}

if ($action === 'section.list') {
    $rows = sb_db_fetch_all("
        SELECT
            id,
            name,
            sort,
            created_by,
            created_at,
            updated_by,
            updated_at
        FROM sitebuilder.site_section
        ORDER BY sort ASC, id ASC
    ");

    sb_json_ok([
        'sections' => array_map('sb_section_normalize_row', $rows),
        'handler' => 'section',
        'file' => __FILE__,
    ]);
}

if ($action === 'section.create') {
    sb_section_require_admin();

    $name = trim((string)($_POST['name'] ?? ''));
    $sort = (int)($_POST['sort'] ?? 500);

    if ($name === '') {
        sb_json_error('NAME_REQUIRED', 422);
    }

    $currentUserId = (int)$USER->GetID();

    $pdo = sb_db();

    $st = $pdo->prepare("
        INSERT INTO sitebuilder.site_section (
            name,
            sort,
            created_by,
            created_at,
            updated_by,
            updated_at
        ) VALUES (
            :name,
            :sort,
            :created_by,
            now(),
            :updated_by,
            now()
        )
        RETURNING
            id,
            name,
            sort,
            created_by,
            created_at,
            updated_by,
            updated_at
    ");

    $st->execute([
        ':name' => $name,
        ':sort' => $sort,
        ':created_by' => $currentUserId,
        ':updated_by' => $currentUserId,
    ]);

    $row = $st->fetch(PDO::FETCH_ASSOC);

    sb_json_ok([
        'section' => sb_section_normalize_row($row ?: []),
        'handler' => 'section',
        'file' => __FILE__,
    ]);
}

if ($action === 'section.update') {
    sb_section_require_admin();

    $id = (int)($_POST['id'] ?? 0);
    $name = trim((string)($_POST['name'] ?? ''));
    $sort = (int)($_POST['sort'] ?? 500);

    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }

    if ($name === '') {
        sb_json_error('NAME_REQUIRED', 422);
    }

    $currentUserId = (int)$USER->GetID();

    $pdo = sb_db();

    $st = $pdo->prepare("
        UPDATE sitebuilder.site_section
        SET
            name = :name,
            sort = :sort,
            updated_by = :updated_by,
            updated_at = now()
        WHERE id = :id
        RETURNING
            id,
            name,
            sort,
            created_by,
            created_at,
            updated_by,
            updated_at
    ");

    $st->execute([
        ':id' => $id,
        ':name' => $name,
        ':sort' => $sort,
        ':updated_by' => $currentUserId,
    ]);

    $row = $st->fetch(PDO::FETCH_ASSOC);

    if (!$row) {
        sb_json_error('SECTION_NOT_FOUND', 404);
    }

    sb_json_ok([
        'section' => sb_section_normalize_row($row),
        'handler' => 'section',
        'file' => __FILE__,
    ]);
}

if ($action === 'section.delete') {
    sb_section_require_admin();

    $id = (int)($_POST['id'] ?? 0);

    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }

    $pdo = sb_db();
    $pdo->beginTransaction();

    try {
        $st = $pdo->prepare("
            UPDATE sitebuilder.site
            SET section_id = NULL
            WHERE section_id = :id
        ");
        $st->execute([':id' => $id]);

        $st = $pdo->prepare("
            DELETE FROM sitebuilder.site_section
            WHERE id = :id
        ");
        $st->execute([':id' => $id]);

        $pdo->commit();

        sb_json_ok([
            'deleted' => true,
            'id' => $id,
            'handler' => 'section',
            'file' => __FILE__,
        ]);
    } catch (Throwable $e) {
        $pdo->rollBack();

        sb_json_error($e->getMessage(), 500, [
            'handler' => 'section',
            'file' => __FILE__,
        ]);
    }
}

if ($action === 'site.setSection') {
    sb_section_require_admin();

    $siteId = (int)($_POST['siteId'] ?? 0);
    $sectionId = (int)($_POST['sectionId'] ?? 0);

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    if ($sectionId > 0) {
        $exists = sb_db_fetch_all("
            SELECT id
            FROM sitebuilder.site_section
            WHERE id = :id
            LIMIT 1
        ", [
            ':id' => $sectionId,
        ]);

        if (empty($exists)) {
            sb_json_error('SECTION_NOT_FOUND', 404);
        }
    }

    sb_db_execute("
        UPDATE sitebuilder.site
        SET
            section_id = :section_id,
            updated_by = :updated_by,
            updated_at = now()
        WHERE id = :site_id
    ", [
        ':site_id' => $siteId,
        ':section_id' => $sectionId > 0 ? $sectionId : null,
        ':updated_by' => (int)$USER->GetID(),
    ]);

    $site = sb_find_site($siteId);

    sb_json_ok([
        'site' => $site,
        'handler' => 'section',
        'action' => 'site.setSection',
        'file' => __FILE__,
    ]);
}

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'section',
    'action' => $action,
    'file' => __FILE__,
]);
```

---

## 5. В `api/index.php` добавь роутинг секций

Файл:

```text
/local/sitebuilder/api/index.php
```

Перед финальным:

```php
sb_json_error('UNKNOWN_ACTION', 400, [
```

добавь:

```php
if (
    $action === 'section.list' ||
    $action === 'section.create' ||
    $action === 'section.update' ||
    $action === 'section.delete' ||
    $action === 'site.setSection'
) {
    require __DIR__ . '/handlers/section.php';
    exit;
}
```

---

## 6. Проверка через консоль

Создать секцию:

```js
fetch('/local/sitebuilder/api.php', {
  method: 'POST',
  headers: {'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'},
  body: new URLSearchParams({
    action: 'section.create',
    name: 'Информационные порталы',
    sort: '100',
    sessid: BX.bitrix_sessid()
  }),
  credentials: 'same-origin'
})
.then(r => r.json())
.then(console.log);
```

Список секций:

```js
fetch('/local/sitebuilder/api.php', {
  method: 'POST',
  headers: {'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'},
  body: new URLSearchParams({
    action: 'section.list',
    sessid: BX.bitrix_sessid()
  }),
  credentials: 'same-origin'
})
.then(r => r.json())
.then(console.log);
```

Назначить сайт в секцию:

```js
fetch('/local/sitebuilder/api.php', {
  method: 'POST',
  headers: {'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'},
  body: new URLSearchParams({
    action: 'site.setSection',
    siteId: '11',
    sectionId: '1',
    sessid: BX.bitrix_sessid()
  }),
  credentials: 'same-origin'
})
.then(r => r.json())
.then(console.log);
```

После этого `site.list` должен вернуть у сайта:

```json
"sectionId": 1
```

Когда это заработает, следующим сообщением добавим группировку сайтов по секциям на главной странице и админские кнопки управления секциями.
